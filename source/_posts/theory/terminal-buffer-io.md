---
title: 终端闲思录（2）- 终端与缓冲的关系
date: 2023-12-17 10:34:44
tags: computer theory
categories:
    - terminal
    - computer theory
    - buffer io
---

我们已经知道标准三剑客（标准输入、标准输出、标准错误）的本质是文件描述符，其连接的目的地可以是任意类型的文件，终端只是常用的目的地之一。那么，当目的地类型不同，IO 的行为是否也有所不同呢？那位架构师所说**“控制台是同步的”**是否过于危言耸听了呢？

让我们看回作为“Linux 一切皆文件”的通用 I/O 接口：

```c
#include <unistd.h>
ssize_t read(int fd, void *buffer, size_t count);
	Returns number of bytes read, 0 on EOF, or –1 on error
        
ssize_t write(int fd, void *buffer, size_t count);
	Returns number of bytes written, or –1 on error
```

这是 glibc 对系统调用`read`、`write`的封装，大部分应用的 IO 都是对这两个封装函数的调用，即便是不使用 C 库的语言，其标准库也提供对系统调用`read`、`write`的封装，比如 Go 语言，其标准库底层直接对接的系统调用，与 glibc 处于同一层级。

从接口定义可知，读与写都需要传入一个存储读写内容的缓冲区指针**buffer**以及存储内容大小的**count**，显而易见的是：在所需传输内容大小一定的情况下，**buffer**越小，对`read`、`write`的调用次数越多，反之则调用次数越少，我们来看一个不同 buffer 大小对传输时间影响的例子：

![图 1-1 复制 100MB 大小的文件所需时间](https://qiniu.liupzmin.com/copy-100m.png)

读和写大体涉及三块时间：**系统调用的时间**、**内核与用户空间数据复制的时间**、**内核和磁盘交互的时间**。可见系统调用的成本还是比较可观的，当 `BUF_SIZE` 增长到 4096 的时候，总耗时趋于稳定，`BUF_SIZE` 的增加对性能的提升不再显著，这是因为与其它两个时间耗时相比，系统调用的时间成本已经微不足道了。

上面的例子混合了读与写，初次操作不可不免的需要从磁盘传输数据到页高速缓存中，让我们再看一个只有写的例子：

![图 1-2 写一个 100MB 大小的文件所需的时间](https://qiniu.liupzmin.com/write-100m.png)

在 Linux 中，写是异步的，内容会先进入页高速缓存，之后`write`系统调用返回，真正落盘的操作由内核异步完成，因此只要内存充足，`write`的性能总是可以得到保证的。

> 注：图中两例引用自[Linux/UNIX系统编程手册](https://book.douban.com/subject/25809330/)，用于说明系统调用的昂贵，我偷懒没有自己写例子运行。

总之，如果有大量内容需要使用 I/O 接口传输，或者需要长时间不定时调用 I/O 接口，通过采用合理的大块空间来缓冲数据，以减少系统调用的次数，可以极大地提高 I/O 的性能，此 glibc 中 stdio 之所为也！

C 库（Linux 中为 glibc）将文件读写抽象成了一个名为`FILE*`的流（stream，标准三剑客会被抽象为 stdin、stdout、stderr），其中就包含我们上面提到的缓冲处理，这避免了程序员自行处理数据缓冲。C 库有三种缓冲类型：

- **_IOFBF（全缓冲）**。单次读写数据的大小与缓冲区大小相同，指代磁盘文件的流默认采用此模式。
- **_IOLBF（行缓冲）**。对于写入，在遇到换行符时才执行（除非缓冲区已填满）；对于读取，每次读取一行数据。当连接终端时默认采取行缓冲。
- **_IONBF（无缓冲）**。不对 I/O 进行缓冲，每个 stdio 库函数将立即调用`read`、`write`，连接到终端的标准错误即是这种类型。

用一个简单的程序来测试终端对 C 库缓冲的影响：

```c
#include <stdio.h>
#include <unistd.h>
void _isatty();

int main() {

    for (;;) {
        _isatty();
        sleep(1);
    }
    return 0;
}

void _isatty(){
    if (isatty(fileno(stdout))) {
        printf("stdout is connected to a terminal (line buffered)\n");
        fprintf(stderr,"an error painted to stderr\n");
    } else {
        printf("stdout is not connected to a terminal (fully buffered)\n");
        fprintf(stderr,"an error painted to stderr, but redirected\n");
    }
}
```

请分别**在终端中直接运行**（./isatty）和**重定向到文件运行**（./isatty > isatty.log 2>&1 &），观察终端和 isatty.log 文件的输出，会发现如下现象：

1. 重定向（不再指向终端）会让标准输出变为全缓冲。
2. 重定向对 stderr 无影响，默认情况下依然是无缓冲。

由此可知，**当向标准输出写日志时，其缓冲行为与是否连接到终端有关，当作为后台服务运行时，即便将日志打到标准输出，写日志的行为也是有缓冲的。至于标准错误，如果对错误信息没有即时要求，也是可以调整其缓冲模式的，不必依赖默认设置。**

当然，这是 C 库的做法，换一种不依赖 C 库的语言，行为可能就会不同，我们来看一下 Go 语言的表现（我总是举 Go 语言的例子，熟悉是一方面，更重要的是，Go 与 C 库脱离的很彻底）：

```go
package main

import (
	"bufio"
	"fmt"
	"log/slog"
	"os"
	"time"

	"golang.org/x/term"
)

func main() {
	w := bufio.NewWriter(os.Stdout)
	h := slog.NewJSONHandler(w, nil)

	logger := slog.New(h)

	// 检查标准输出是否连接到终端
	a := func() {
		if term.IsTerminal(int(os.Stdout.Fd())) {
			fmt.Printf("标准输出连接到终端--from fmt\n")
			fmt.Fprintf(os.Stderr, "标准错误连接到终端--from fmt\n")
			slog.Info("标准输出连接到终端--from slog")
			logger.Info("标准输出连接到终端--from slog json logger")

		} else {
			fmt.Printf("标准输出未连接到终端--from fmt\n")
			fmt.Fprintf(os.Stderr, "标准错误未连接到终端--from fmt\n")
			slog.Info("标准输出未连接到终端--from slog")
			logger.Info("标准输出未连接到终端--from slog json logger")
		}
	}

	for {
		a()
		time.Sleep(1 * time.Second)
	}
}

```

请再次分别**在终端中直接运行**（./isatty）和**重定向到文件运行**（./isatty > isatty.log 2>&1 &），观察终端和 isatty.log 文件的输出，会发现如下现象：

1. 无论是否连接到终端，对标准输出和标准错误的输出行为不会发生任何变化

2. 想要异步写入，需要自行构建带缓冲的 logger，如此例中的：

   ```go
   w := bufio.NewWriter(os.Stdout)
   h := slog.NewJSONHandler(w, nil)
   
   logger := slog.New(h)
   ```

   而 slog 默认的 logger 是写入 stderr 的，不信你可以运行`./isatty > isatty.log &`仅仅重定向标准输出试试。

产生这种现象的原因是 Go 标准库并没有像 C 库那样对标准库的 I/O 大包大揽，文件流`*os.File`是无缓冲的，Go 标准库函数的 I/O 基本都是基于接口，且提供有实现了缓冲 I/O 以及 I/O 接口的`bufio`包。

上面例子中的 json logger 就是使用`bufio`基于标准输出创建了一个带缓冲的`writer`，而 slog 包中创建 Handler 仅需传入实现`writer`接口的对象即可，因此我们得到一个带有缓冲的 logger。

![图 1-3 I/O 缓冲](https://qiniu.liupzmin.com/c-io-summary.png)

不论标准库使用如何方式提供缓冲，其目的始终是减少系统调用，图 1-3 以 C 库为例展示了这种 I/O 模型，**使用标准库函数将日志写入标准输出，是可以设置合理的缓冲区的，并不存在“同步、性能低下”的担忧。**因此，我们可以确定如下几点：

1. 标准三剑客是文件描述符，任何基于文件的读写库函数都可以向标准输出和标准错误写入。
2. 向标准输出和标准错误写入不影响日志框架的使用。
3. 即便日志框架不提供缓冲区，也是可以提供一个实现了缓冲的 writer 以实现异步写入。

对于我们提供的两个例子，可以将其做成镜像，放到 k8s 上运行，使用`kubectl logs`观察其输出行为，你觉得会是怎样的呢？

k8s 中的容器，除了使用`exec`附加到容器 namespace 启动的进程有控制终端之外，大部分以后台进程运行的程序是没有控制终端的，不妨进入容器的 namespace 看看服务进程的标准三剑客指向哪里：

```shell
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:06 /usr/local/bin/isatty
    7 root      0:00 sh
   13 root      0:00 ps aux
/ # cd /proc/1/fd
/proc/1/fd # ls -l
total 0
lrwx------    1 root     root            64 Dec 12 02:25 0 -> /dev/null
l-wx------    1 root     root            64 Dec 12 02:25 1 -> pipe:[54791]
l-wx------    1 root     root            64 Dec 12 02:25 2 -> pipe:[54792]
```

1 号进程是我们的测试进程，可见其标准输入指向`/dev/null`，标准输出和标准错误分别指向不同的管道，不知你是否好奇这是如何做到的呢？容器的日志又是如何写到`/var/log/containers`中的呢？

***参考文献***

1. [UNIX环境高级编程](https://book.douban.com/subject/25900403/)
2. [Linux/UNIX系统编程手册](https://book.douban.com/subject/25809330/)