---
title: 在 Linux 上有哪些运行程序的方式？
date: 2019-11-26 10:47:48
tags: 
    - linux
    - process
    - sysv
    - init
    - shell
    - bash
categories: linux
---

## Have you ever thought ?

前段时间，我在编写一个 `Go` 程序，这个程序要做的一件事是在操作系统上执行一个命令（可执行文件或者可执行脚本），程序大概像下面这样子：

```go
cmdSlice := strings.Fields(strings.TrimSpace(cmdString))
if len(cmdSlice) == 0 {
	return errors.New("index out of range [0] with length 0")
}
// search for an executable named file in the
// directories named by the PATH environment variable.
// If file contains a slash, it is tried directly and the PATH is not consulted.
// The result may be an absolute path or a path relative to the current directory.
executableFile, err := exec.LookPath(cmdSlice[0])
if err != nil {
	return errors.WithStack(NewPathError(cmdSlice[0], err.Error()))
}

cmd := exec.Command(executableFile, cmdSlice[1:]...)
```

当我让程序去执行一个 shell 脚本的时候，收到了 `fork/exec: exec format error` 的错误，然而我在 shell 下执行这个脚本却是正常的，这让我很迷惑。

当我弄清楚原因是我没有在脚本里加 `Shebang`（`#!`） 的时候，我更加疑惑了：`为什么操作系统会容忍我的过错呢？`对此，我会在稍后的章节中进行解释。当我搞清楚问题的始末的之后，我突然对操作系统执行程序的方式产生了极大的兴趣，并试图去搞清楚它。这就是我写这篇文章的初衷。


你是否想过，除了在 `shell` 下启动一个程序，是否还有其它的方式？ 我们是不是永远无法摆脱 `shell`？ 你是否曾经对 `shell` 下各种的执行方式感到困惑？比如，`“source xxx.sh”`、`“. xxx.sh”`、`“./xxx.sh”`、`“. ./xxx.sh”`、`“sh ./xxx.sh”`。 没关系，这篇文章会带你走出迷雾。

本文会涉及到一点 `Sysvinit` 和 `Systemd` 的内容， 但我不会过多的去介绍他们，只是简单说明，这是一种能让你的程序运行起来的方式，我最后的重点会放在 `shell` 上面。

## 让程序跑起来有多少种可能的方法？

当我产生这个疑问之后，我努力的思考，并去寻找答案，最后总结了如下几种：

1. 传统的 Sysvinit 方式
2. Systemd
3. `crontab` 或者 `Systemd Timer`
4. shell（无论是终端还是 sshd ）
5. GUI

其中 1, 2 两种方式可以算作同一类，虽然他们的工作方式有所不同，但都属于系统管理层面。如果你的程序是一个随系统启动，并托付给系统管理的 `Daemon` ，那么最好的方式就是通过 `Sysvinit` 或者 `Systemd` 来管理，他们都是 Linux 的 init 系统。相似的 init 系统还有 `Upstart` ，但我并不熟悉它，所以不准备做介绍，当然这并不影响，因为他们属于一类系统。

`定时任务`是另一个可能会拉起一个程序的方式，相信很多朋友都有在 Linux 上使用 `crontab` 的经历，而它的替代品 `Systemd Timer` 可能就没那么多人熟悉了。

`shell` 是最常见的启动程序的方式。事实上 `shell` 的主要作用就是去运行其它的程序，即便是前面 3 种方式，很多时候也是使用 shell 来启动需要运行的程序的，只不过不是我们手动在 shell 里执行而已。

还有一种方式就是在桌面环境下，使用 GUI 来启动一个前台程序，你可能通过点击一个 `.desktop` 的快捷方式来启动一个桌面应用，在我的 `Manjaro` 下桌面应用全部是由 `plasmashell` 这个进程 `fork` 出来的子进程。

## 承前启后的 SysV init

Linux 操作系统的启动首先从 BIOS 开始，然后由 Boot Loader 载入内核，并初始化内核。内核初始化的最后一步就是启动 init 进程。这个进程是系统的第一个进程，PID 为 1，又叫超级进程，也叫根进程。它负责产生其他所有用户进程。所有的进程都会被挂在这个进程下，如果这个进程退出了，那么所有的进程都被 kill 。如果一个子进程的父进程退了，那么这个子进程会被挂到 PID 1 下面。

因为大多数 Linux 发行版的 init 系统是和 Unix System V 是相互兼容的，因此 Linux 上的 init 系统也被成为 `Sysvinit` 。

在 sysvinit 下有几个 `runlevel` ，并且有 0~6 七个运行级别，比如：常见的 3 级别指定启动到多用户的字符命令行界面，5 级别指定启动到图形界面，0 表示关机，6 表示重启。其配置在 /etc/inittab 文件中。

与此配套的还有 /etc/init.d/ 和 /etc/rc[X].d，前者存放各种进程的启停脚本（需要按照规范支持 start，stop子命令），后者的 X 表示不同的 runlevel 下相应的后台进程服务，如：/etc/rc3.d 是 runlevel=3 的。 里面的文件主要是 link 到  /etc/init.d/ 里的启停脚本。其中也有一定的命名规范：S 或 K 打头的，后面跟一个数字，然后再跟一个自定义的名字，如：S01rsyslog，S02ssh。S 表示启动，K表示停止，数字表示执行的顺序。

为了将操作系统带入可操作状态，init 系统通过读取 `/etc/inittab` 获得 runlevel，然后依次顺序执行对应 level 下的脚本。rc[X].d 下都是些 link， 链接到 rc.d 中的 shell 脚本， 可见系统初始化过程中依然是使用的 `shell` 来启动相应程序的。

然而这些脚本中需要使用 awk, sed, grep, find, xargs 等等这些操作系统的命令，这些命令需要生成进程（这涉及到 shell 的工作方式，我稍后在 shell 小节详细介绍），生成进程的开销很大，关键是生成完这些进程后，这个进程就干了点屁大的事就退了。这完全是杀鸡用牛刀啊，操作系统废了九牛二虎之力拉起来一个进程，结果这个进程就干了个把字符串转为小写的活，然后丢下一脸懵逼的操作系统就潇洒的退出了。

可以想见，当rc.d中有大量的脚本，且脚本中又有成百上千个类似于 awk、sed、grep 这样的命令，系统的启动过程就会慢的要死。当然对于启停不那么频繁的服务器来说，这依然可以接受，而且这样的系统设计也很符合 Unix 设计哲学：`Do one thing and Do it well`，所以 sysvinit 可以一统江湖几十年。直到 2006年 Linux 内核进入 2.6 时代，Linux 开始进入桌面系统，而桌面系统和服务器系统不一样的是，桌面系统面临频繁重启，而且，用户会非常频繁的使用硬件的热插拔技术。于是，这些新的场景，让 sysvint 受到了很多挑战。

> 更详细的 sysvint 介绍可以参考 [浅析 Linux 初始化 init 系统-sysvinit](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init1/index.html)

## 步行夺猛马的 Systemd

**历史上总是会有人站出来对现状说不**，2010 年 [Lennart Poettering](https://en.wikipedia.org/wiki/Lennart_Poettering)和他的小伙伴们开源并发布了一个新的 init 系统——`systemd`。

`Systemd` 是 Linux 系统中最新的初始化系统（init），它主要的设计目标是克服 sysvinit 固有的缺点，提高系统的启动速度。systemd 和 ubuntu 的 upstart 是竞争对手，而 ubuntu 在15.04及后续版本中已将 systemd 设置为默认 init 程序，redhat 和 centos 也从 7.0 之后开始使用 systemd，截止目前 systemd 已经运行在大部分的 Linux发行版中。

在系统启动上 systemd 拥有绝对的优势，有张三方对比图可见分晓：

![](http://qiniu.liupzmin.com/boot.png)

如今 systemd 成为 1 号进程，后续所有的进程都是由它 fork 出来的：

```shell
systemd(1)─┬─ModemManager(494)─┬─{ModemManager}(495)
           │                   └─{ModemManager}(498)
           ├─NetworkManager(463)─┬─{NetworkManager}(467)
           │                     └─{NetworkManager}(470)
           ├─agent(1229)─┬─{agent}(1232)
           │             └─{agent}(1236)
           ├─avahi-daemon(465)───avahi-daemon(468)
           ├─baloo_file(655)───{baloo_file}(671)
           ├─bluetoothd(464)
           ├─crond(460)
           ├─cupsd(476)
           ├─dbus-daemon(462)
           ├─dbus-daemon(1458)
           ├─dockerd(557)─┬─containerd(583)─┬─{containerd}(589)
           │              │                 ├─{containerd}(590)
           │              │                 └─{containerd}(13694)
           │              ├─{dockerd}(561)
           │              └─{dockerd}(861)
           └──fcitx(1431)─┬─sh(1576)
                          └─{fcitx}(1478)

```

深入了解请参考：[LINUX PID 1 和 SYSTEMD](https://coolshell.cn/articles/17998.html)

## 定时任务

简单说一下定时任务，当我们使用 crontab 配置了一个定时执行任务之后，Cron每分钟做一次检查，看看哪个命令可执行，当 Cron 检查到有命令需要执行时则 fork 子进程，再由此子进程 fork-execve 执行真正的命令：

```
init(1)-+-NetworkManager(1747)
        └-crond(2107)-+-crond(355)---sh(356)---sleep(14329)
                      |-crond(10967)---sh(10968)---sleep(14326)
                      |-crond(12098)---sh(12099)---sleep(14327)
                      `-crond(15114)---sh(15115)---sleep(14328)
```

这篇 [Linux cron运行原理](http://www.yunweipai.com/archives/4479.html) 有更详细的介绍。

对于 systemd 我测试一个 timer，其进程是挂在 systemd 进程之下的，我猜测也是 systemd 进程去 fork 执行 timer 中的任务。

```shell
{17:22}~/PycharmProjects ➭ ps -ef|grep udp
root     17002     1  0 17:19 ?        00:00:00 /usr/bin/python3 /home/liupeng/udpClient.py
```

## 伟大的造物主——shell

通过前面的分析，会发现 Linux 上的程序绝大多数情况下是通过 shell 来执行的，所以我们接下来将重点放在 shell 上。

你可能会问： shell 有什么好讲的，它不就是个与内核交互的`外壳`程序么？

没错，它的功能就是如此纯粹——`a shell is a user interface for access to an operating system's services`，但它却无处不在。

传统的 Sysvinit 系统下绝大部分的系统服务都是通过 shell 拉起来的，虽然到了 Systemd 时代，很多工作由 C 语言重新实现了（具体见[LINUX PID 1 和 SYSTEMD](https://coolshell.cn/articles/17998.html)）,但是你依然可以使用 systemd 来管理你的启停脚本，这些脚本用来启停你的程序。而对于非 Daemon 方式的程序。你仍然需要用 shell 来启动它们到前台，或者使用 nohup、setsid等方式启动到后台。

可见，我们无法逃离 `shell`，它就像是一个造物主，系统中几乎所有的进程都是或曾是它的子民。

要讲清楚shell是一个十分艰巨的任务，对于只查过几天资料的我来说自然无法胜任，但是择其一两点来讲，以**多少理清一些 Linux 下程序启动与运行的原理**为目的，或可一试。

> 文中涉及到关于 shell 的实验或者结论皆以 Bash 作为参考依据。


### What is a shell?

[Bash](https://www.gnu.org/software/bash/manual/bash.html#What-is-a-shell_003f) 主页上有关于 shell 的定义：

> At its base, a shell is simply a macro processor that executes commands. The term macro processor means functionality where text and symbols are expanded to create larger expressions.

这段话真不太好翻译，勉强翻译一下为：`从根本上说，shell 只是执行命令的宏处理器。术语宏处理器是指将文本和符号扩展以创建更大的表达式的功能。`

对于 Unix shell 来说，它既是一个命令行解释器也是一个编程语言。shell 作为命令行解释器为丰富的 GNU 工具集提供了用户接口，而作为编程语言它成功的将这些工具集结合在一起，之后就可以将命令编写进文件，去完成各种各样的任务。

很多人可能傻傻分不清 `terminal`、`tty`、`console` 和 `shell`，这里第一个高票回答对这些概念做了详细的解释：[What is the exact difference between a 'terminal', a 'shell', a 'tty' and a 'console'?](https://unix.stackexchange.com/a/4132)。如果英文阅读不畅，知乎上有人将其翻译了一下：[终端、Shell、tty 和控制台（console）有什么区别？](https://www.zhihu.com/question/21711307/answer/56056972)，我不再做额外的阐述了，接下来只需要记住 shell 是一个命令行解释器就好，它可以运行在交互模式和非交互模式。

### shell 是如何查找命令的

当我们在交互式 shell 下敲下一个命令时，shell 查找命令文件的规则大概如下：

1. 执行命令前 shell 会先检查是否有 alias，如果有就会使用 alias 中的内容。

2. 如果 command 名字不包含 `"/"` ，shell 将尝试寻找它。如果存在同名的函数，则会调用函数。

3. 如果没有匹配到函数，则从 shell 内置命令（`builtins`）中寻找，如果找到则调用该命令。

4. 如果都没有找到则从 `$PATH` 中寻找，为了避免每次遍历 `$PATH` ,shell 维护了一张 `HASH` 表，记录了每个命令对应的绝对路径，如果 HASH 表中没有再去 `$PATH` 中的目录遍历，如果 PATH 中未找到就执行一个预定义的函数 `command_not_found_handle` 。如果函数存在，则在`子 shell` 中调用，如果不存在则打印错误信息并返回 127 状态码。

5. 如果寻找成功或者 command 中含有 `“/”`， shell 将在新环境中执行它（ fork 一个新进程 ）。

6. 如果 command 不是异步启动的，shell 将等待其完成并收集退出状态码。

如上所述就是 shell 在执行命令式的查找规则。也是时候破解一下我们在文章开头留下的谜题了，先从 `./` 开始吧。

`./` 在类 Unix 系统中表示相对路径指向某个文件或者目录，因为在 Unix 系统中 PATH 不包含当前路径，也无法包含当前路径。如 `./test`、`touch ./a`

`.` 是 BASH 的一个内建命令，它继承自 `Bourne shell (sh)`，并且是 `source` 同义词，跟 `source` 功能相同。

`. filename [arguments]` 的功能是在当前 shell 上下文中（ 不会 fork ）读取并执行 `filename` 中的命令，如果 `filename` 不包含 `“/”` ， shell 将从 `$PATH` 中寻找该文件，如果当前 shell 不是 `POSIX` 模式，则在 `PATH` 中寻找失败后，继续从当前目录中寻找。

对于 `sh ./test.sh` 这种模式，在 bash 的文档中可以找到对应的描述 `Invoked with name sh`（大多数 Linux 发行版会把 sh 设置成 bash 的软连接，所以这里只针对此种情况）：

> When invoked as sh, Bash enters POSIX mode after the startup files are read.

[POSIX mode](https://www.gnu.org/software/bash/manual/bash.html#Bash-POSIX-Mode) 我会在后面展开介绍，这里暂且略去，开始进入 shell 如何执行一个 command 吧。


### shell 是如何执行命令的

我在介绍 Systemd 和 cron 的时候用了 `fork` 这个词，而在描述 shell 的时候仅仅说`“shell 启动相应程序”`。其实，shell 执行一个程序的方式也一样使用了 `fork`，我只是为了能在本章节重点作介绍才故意没有使用 `fork` 这个词。

我们知道， Linux 下的可执行文件可以分为 `ELF 文件`和`脚本文件`，当我们在 bash 下输入一个命令执行某个 ELF 程序时，Linux 系统是如何装载并执行它的呢？

首先，在用户层面，bash 进程会调用 `fork()` 系统调用创建一个新的进程。其次，新的进程通过调用 `execve()` 系统调用来执行指定的 ELF 文件。原先的 bash 进程继续返回并等待刚才启动的新进程结束，之后继续等待用户输入命令。

当进入 `execve()` 系统调用之后，Linux 内核就开始进行真正的装载工作。在内核中，`execve()` 系统调用相应的入口是 `sys_execve()`。`sys_execve()` 进行一些参数的检查复制之后，调用 `do_execve()`。`do_execve()` 会首先查找被执行的文件，如果找到文件，则读取文件的前 128 个字节。

为什么要先读取文件的前 128 个字节？这是因为 Linux 支持的可执行文件不止 ELF 一种，还包括 `a.out`、`Java 程序`、以 `#!` 开头的脚本程序。`do_execve()` 通过读取前 128 个字节来判断文件的格式。每种可执行文件格式的开头几个字节都是很特殊的，尤其是前4个字节，被称为 `魔数`（Magic Number）。

我们用一段 C 程序来读取一些各种文件的前 4 个字节：

```c
#include <stdio.h>
int main (int argc, const char * argv[]) {
    
		FILE *fp;
		int r;
		int i;
		fp = fopen(argv[1], "rb");
		fread(&r, 4, 1, fp);
	
		printf("%X \n", r);
	
		fclose(fp);
		return 0;
}
```

1. ELF

    我们编译这段程序，并读取程序自身

    ```shell
    {15:32}~/Learing/c/src:master ✗ ➭ gcc -o read4bytes read4bytes.c 
    {15:33}~/Learing/c/src:master ✗ ➭ ./read4bytes read4bytes
    464C457F
    ```

    可以看到输出为 `464C457F`，我们查看ASCII 表，得出如下的对应关系：
    ![ELF Header](http://qiniu.liupzmin.com/elf-header-small.png)

    我的操作系统字节序是小端法排序，因此，`ELF的可执行文件格式的头 4 个字节为 0x7F、E、L、F`。

2. shell 脚本

    ```shell
    {15:33}~/Learing/c/src:master ✗ ➭ ./read4bytes ~/meta    
    622F2123 
    ```

    前 4 个字节为 `622F2123`，我们再查一下 ASCII 表的对应关系：
    ![shell script header](http://qiniu.liupzmin.com/bash-header-small.png)

    翻转一下就是 `#!/b`,可以猜测如果我们多读 7 个字节，结果肯定是`#!/bin/bash`.

    对于 `python`、`perl`、`php`脚本处理方式相同。

3. java class

    ```bash
    {15:51}~/Learing/c/src:master ✗ ➭ ./read4bytes ~/java/HelloWorld.class 
    BEBAFECA
    ```
    `《程序员的自我修养》`一书 6.5 章节介绍 Linux 装载可执行文件时，依次介绍了 `ELF`、`java 可执行文件`和 `#！`三种情况。ELF 的前4个字节将 16 进制转换为 ASCII 字符是 `E`、`L`、`F`；但是 java 的 class 文件则不同，由上可知读出的前4个字节的 16 进制表示为 `BEBAFECA`。因为是小端，所以 16 进制的表示法刚好是 `CAFEBABE`，并不需要转化成具体的字符，而书中介绍说 `“Java的可执行文件格式的头 4 个字节为 c、a、f、e”`，我猜可能书中存在前后逻辑不一致的问题，除非真的存在所谓的 `java 可执行文件`，如果有朋友了解，欢迎联系我，给我批评指正。

    关于 [CAFEBABE](https://en.wikipedia.org/wiki/Java_class_file)的来源可以参见 wiki 上 James Gosling 的自白。

当 `do_execve()` 读取了 128 个字节的文件头部之后，调用 `search_binary_handle()` 去搜索和匹配合适的可执行文件装载处理过程。Linux 中所有被支持的可执行文件格式都有相应的装载处理过程，`search_binary_handler()` 会通过判断头部的魔数确定文件的格式，并且调用相应的装载处理过程。常见的可执行程序及其装载处理过程的对应关系如下所示.

- ELF 可执行文件：load_elf_binary()
- a.out 可执行文件：load_aout_binary()
- 可执行脚本程序：load_script()

有必要提一下 [a.out](https://zh.wikipedia.org/wiki/A.out)， a.out 本身要追溯到更早的 Unix 时代，并且伴随 Linux 的诞生至今在 Linux 中有将近 ~28 年的历史。从 内核 5.1 开始， Linux 移除 a.out 格式的消息，因为ELF 自 1994 年进入 Linux 1.0 以来，已经 ~25 年了，a.out 早已年久失修，而且现在基本上找不到能产生 a.out 格式的编译器了。

大家可能会说，gcc 默认编译生成的不就是 a.out 么？非也，此 a.out 非彼 a.out。gcc 默认生成的 a.out 的实际格式也是 ELF，如果你按照刚才的方式读取 a.out 的前四个字节，你会发现同样是 `464C457F`，a.out 这个名字很大意义上属于计算机历史文化的沿袭，想了解更多可以参考[为 a.out 举行一个特殊的告别仪式](https://tinylab.org/goodbye-a.out/)。

我在这里省去 `load_elf_binary()` 的过程，只需提一下其中一步会修改系统调用的返回地址为 ELF 文件的入口地址，细节可以去参考`《程序员的自我修养》` 6.5 节。我们说当 `load_elf_binary()` 执行完毕之后，返回至 `do_execve()` 再返回至 `sys_execve()` 时，因为 `load_elf_binary()` 已经修改了返回地址，所以当 `sys_execve()` 系统调用从内核态返回到用户态时，EIP 寄存器直接跳转到了 ELF 程序的入口地址，于是开始执行新程序的代码指令， ELF 可执行文件装载完成。

是时候去破解我在文章开头留下的问题了，我用 Go 程序通过 `fork` 和 `exec` 去执行脚本的时候收获到 `fork/exec: exec format error` 的错误。现在来看是 `search_binary_handle()` 的过程出了问题，内核并没有识别到脚本文件格式，经查确认是我脚本中没有加入 [Shebang](https://zh.wikipedia.org/wiki/Shebang)，当我在首行增加了 `#!/bin/bash` 之后，程序便可以正确运行了。

我还没有解释为什么在交互式的 shell 下执行不带 `Shebang` 的脚本不会触发错误，因为说起我寻找答案的过程总让我喟叹不已，我是从一篇几近 30 年前的文章中找到答案的。这让我想到了 5 年前我学习 Oracle 调优时从一本 10 年前出版的书中获益的经历。我难以想象，我今天写就的一篇博文，有可能会在 30 年后帮助到另一个人，这会让我永葆写作的热情......

就是这篇 （[Why do some scripts start with #! ... ?](http://www.faqs.org/faqs/unix-faq/faq/part3/section-16.html)）写于 1992 年的文章帮助我找到事实的真相。

简单概括一下就是早在 Unix 时代，为了不让内核什么东西都拿来执行，程序员们发明了 `“magic number”`，通过 magic number 内核可以辨别出哪些是可执行程序，在文件不可执行时抛出 `ENOEXEC` 错误，但是 shell 代码扩充了这项功能，在收到 `ENOEXEC` 失败后会去使用 `“/bin/sh”` 尝试将其作为 shell 脚本去执行，所以脚本执行是由 shell 来完成的，而不是内核，代码逻辑大概像这样：

```c
/* try to run the program */
execl(program, basename(program), (char *)0);

/* the exec failed -- maybe it is a shell script? */
if (errno == ENOEXEC)
    execl ("/bin/sh", "sh", "-c", program, (char *)0);

/* oh no mr bill!! */
perror(program);
return -1;
```
后来，伯克利的一些 guys 扩充了内核的功能，使其可以识别魔数 `“#！”`，如果内核读到 `#！` 则将继续读取该行的剩余部分，并将其作为命令去运行文件中的内容。

试想，当你执行一个没有正确填写 `Shebang` 的脚本文件的时候，shell 很可能会给你报一个没有执行权限的错误，当你依照错误提示给予 `+x` 权限的时候，你很可能收到更多的错误，原因很可能是你正在编写一个 python 脚本。

乖乖的写 `Shebang` 吧 ！ 

事实上，后来我在 Bash 的文档中也找到了相关描述：

> this execution fails because the file is not in executable format, and the file is not a directory, it is assumed to be a shell script and the shell executes it as described in Shell Scripts.

也许这一节描述没有燃起你的兴奋点，因为我假设你对 `fork` 和 `exec` 函数族以及`虚拟内存`有所了解，如果你不了解的话可以参考下面我给出的链接：

- [Unix/Linux fork前传](https://mp.weixin.qq.com/s?__biz=MzAwMDUwNDgxOA==&mid=2652666221&idx=1&sn=ebdb45535eadde947a4f1206c195447f&scene=21#wechat_redirect)
- [Linux fork那些隐藏的开销](https://mp.weixin.qq.com/s?__biz=MzAwMDUwNDgxOA==&mid=2652666200&idx=1&sn=cce7ab86f7a57dd4cb9e372a49bfdcfa&scene=21#wechat_redirect)
- [Fork三部曲之clone的诞生](https://mp.weixin.qq.com/s/kebfrSyeyvMV--XqzBNRSA)
- 深入理解计算机系统 第9章 Virtual Memory

### login、non-login、interactive 、non-interactive 与 Startup Files

因为 shell 可以运行在交互模式和非交互模式下，并且有 login 和 non-login 的情况，所以每一种组合他们读取并执行的 Startup Files 都有所不同，下面我给出一幅图来展示各种不同的情况：

![bash and startup files](http://qiniu.liupzmin.com/bash.png)

所谓的 `login & interactive` 模式我举两个例子，一个是我们登录 Linux 字符界面的时候，输入用户名密码进入的那个 shell 就是登录交互式的，另一个就是我们使用 `sshd` 服务远程登录，在输入用户名密码后获得的 shell 也是登录交互式的。

对于非交互式的 shell 典型的情况就是执行脚本啦，而在执行脚本的时候可以通过添加 `--login` 或者 `-l` 的选项来使这个 shell 去读取 `Startup Files`，因为它没有输入口令的登录动作，只有读取和执行 Startup Files 。

另外，你在 X Windows 下运行 `terminal` 软件打开的 shell 是 `non-login & interactive` 模式的。如果你曾有**在视窗下打开 shell 却无法获取 `～/bash_profile` 中定义的变量**的疑惑的话，现在你可以释然了。

来做个实验吧，我事先在 `/etc/profile`、`/etc/bashrc`、`~/.bash_profile`、`~/.bashrc` 中增加了 `echo “Hello from xxxx”` 的语句，让我们来看看各种情况下我们得到的 shell 到底执行了哪些文件：

1. sshd

    ```shell
    {11:07}~ ➭ ssh root@192.168.1.41
    Last login: Wed Dec  4 10:44:06 2019 from 192.168.1.183
    Hello from /etc/profile
    Hello from /etc/bashrc
    Hello from ~/.bashrc
    Hello from ~/.bash_profile
    ```

    > ~/.bash_profile 调用了 ~/.bashrc， ~/.bashrc 调用了 /etc/bashrc，所以 shell 调用的是/etc/profile 和 ~/.bash_profile

2. GUI Terminal

    ![terminal bash](http://qiniu.liupzmin.com/terminal-bash.bmp)

    GUI 下打开 shell 只运行了 `~/.bashrc`。

3. 运行 bash

    ```shell
    [root@afis-db ~]# bash
    Hello from /etc/bashrc
    Hello from ~/.bashrc
    [root@afis-db ~]# exit
    exit
    [root@afis-db ~]# bash -l
    Hello from /etc/profile
    Hello from /etc/bashrc
    Hello from ~/.bashrc
    Hello from ~/.bash_profile
    ```

    默认情况下，bash 命令进入的是一个非登录的交互式子 shell，当使用 `-l` 或 `--login` 选项后进入的是登录的交互式子 shell 。

4. su

    `su` 的功能是切换用户，其中 `-` 选项表示登录，一个登录的 shell 在 ps 中显示为`-bash`，非登录的显示为 `bash`。

    ```shell
    [root@afis-db ~]# su - oracle
    Hello from /etc/profile
    Hello from /etc/bashrc
    Hello from ~/.bashrc
    Hello from ~/.bash_profile
    [oracle@afis-db ~]$ echo $$
    11935
    [oracle@afis-db ~]$ ps -ef|grep 11935
    oracle   11935 11934  0 13:22 pts/1    00:00:00 -bash
    oracle   11960 11935  0 13:22 pts/1    00:00:00 ps -ef
    oracle   11961 11935  0 13:22 pts/1    00:00:00 grep 11935
    [oracle@afis-db ~]$ exit
    logout
    [root@afis-db ~]# su oracle
    Hello from /etc/bashrc
    Hello from ~/.bashrc
    [oracle@afis-db root]$ echo $$
    11965
    [oracle@afis-db root]$ ps -ef|grep 11965
    oracle   11965 11964  0 13:22 pts/1    00:00:00 bash
    oracle   11982 11965  4 13:22 pts/1    00:00:00 ps -ef
    oracle   11983 11965  0 13:22 pts/1    00:00:00 grep 11965
    ```

5. 运行脚本—非交互模式

    ```shell
    [root@afis-db ~]# ./script.sh 
    I am script!
    [root@afis-db ~]# bash script.sh 
    I am script!
    [root@afis-db ~]# bash -l script.sh 
    Hello from /etc/profile
    Hello from /etc/bashrc
    Hello from ~/.bashrc
    Hello from ~/.bash_profile
    I am script!
    ```

    可见在非交互模式下不会读取任何文件，增加了登录选项则会依次读取 Startup Files。

我们很少以 `bash script.sh` 这种方式执行脚本，更多的是以 `./script.sh`运行，当以后一种方式执行时，真正执行脚本的解释器依赖于具体的 `Shebang`。而我们经常看到使用 `sh script.sh` 这样的方式执行，那 `sh` 究竟是什么呢？

在大多数 Linux 发行版中，`sh 通常是 bash 的软连接`，但是 bash 文章中有如下描述：

> If Bash is invoked with the name sh, it tries to mimic the startup behavior of historical versions of sh as closely as possible, while conforming to the POSIX standard as well.

> When invoked as sh, Bash enters POSIX mode after the startup files are read.

意思是`当你用 sh 来启动 shell 的时候，bash 会以 posix 标准模式运行`，就如同调用了 `bash --posix`。需要注意的是：**`sh` 并不是一个具体的 shell 实现，而是一种规格标准，bash 在这种模式下运行的时候，将遵循 posix 的标准去读取执行文件**。如下图所示：

![sh and startup files](http://qiniu.liupzmin.com/sh.png)

来实地验证一下：

1. 运行 sh

    ```shell
    [root@afis-db ~]# sh
    sh-4.1# exit
    exit
    [root@afis-db ~]# sh -l
    Hello from /etc/profile
    Hello from ~/.profile.(this file is touched by me)
    ```
2. 执行脚本

    ```shell
    [root@afis-db ~]# sh script.sh 
    I am script!
    [root@afis-db ~]# sh -l script.sh 
    Hello from /etc/profile
    Hello from ~/.profile.(this file is touched by me)
    I am script!
    ```

因为生产服务器上使用 Bash 居多，而线上服务多少都依托于 shell 去调用，因为不同的调用方式下 shell 读取执行文件的规则不同，这样就可能对应用造成一定程度的困扰。我曾经维护一个线上的 java 项目，这个项目有几百个 java 服务需要每天定时重启，项目上线时反复检查验证了 cron 服务的配置以确保万无一失，而没有想到启停脚本对环境变量 `JAVA_HOME` 的依赖会在 cron 调用的时候失效。如今看来，只要使用登录非交互模式即可。

## 总结

本片文章细节太过零散，去验证以及查阅资料花了不少时间，其实写作的最初兴奋点是想从 `fork & exec` 的角度去理解 Linux 上各种执行程序的方式，但是回头一看，关于 fork 和 exec 的介绍只有寥寥几笔，剩下的都是关于细节的追求与验证，但是 `Done is better than perfect`～

因能力有限，行文或有疏漏与错误之处，望阅读本文的朋友给予斧正，也希望了解其它启动方式的朋友不吝赐教。




**参考文章：**

1. [Why do some scripts start with #! ... ?](http://www.faqs.org/faqs/unix-faq/faq/part3/section-16.html)
2. [Bash Reference Manual](https://www.gnu.org/software/bash/manual/bash.html)
3. [浅析 Linux 初始化 init 系统——sysvinit](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init1/index.html)
4. [浅析 Linux 初始化 init 系统——Systemd](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init3/index.html)
5. [Linux 的启动流程](https://www.ruanyifeng.com/blog/2013/08/linux_boot_process.html)
6. [LINUX PID 1 和 SYSTEMD](https://coolshell.cn/articles/17998.html)
7. [Difference between a 'terminal', a 'shell', a 'tty' and a 'console'?](https://unix.stackexchange.com/questions/4126/what-is-the-exact-difference-between-a-terminal-a-shell-a-tty-and-a-con?answertab=active#tab-top)
8. [为什么执行自己的程序时需要加上点斜杠](https://www.yanbinghu.com/2019/10/07/14442.html)
9. [Shebang](https://zh.wikipedia.org/wiki/Shebang)
10. [Java class file](https://en.wikipedia.org/wiki/Java_class_file)
11. [为 a.out 举行一个特殊的告别仪式](https://tinylab.org/goodbye-a.out/)
12. [Linux cron运行原理](http://www.yunweipai.com/archives/4479.html)
13. `《程序员的自我修养》`