---
title: 关于并发的一点思考
date: 2023-08-08 18:08:30
tags:
	- golang
    - network
    - tokio
    - asynchronous
categories: golang

---

计算机之所以需要并发，是为了提高 CPU 的利用率，因为大多数任务场景是混合了计算和 I/O 的，那么为了减少响应时间，使任务能够“同时”进行，计算机程序便演化出了并发的概念。

本文就针对 CPU-bound 和 IO-bound 两种极端场景，来聊一聊 Go 和 Tokio 的并发模型在应对不同并发场景下的异同。

[异步 IO 探秘](https://liupzmin.com/2023/06/28/golang/netpoller/) 和 [对话 ChatGPT 理解 Rust 异步网络 io](https://liupzmin.com/2023/06/08/network/talk-rust-async-netio-with-chatgpt/) 已基于 Linux 平台就 Go 和 Tokio 的网络模型做了简要剖析，大致有如下几个要点：

1. 底层 Reactor 都是 非阻塞 I/O + epoll 模型。
2. 事件处理方式不同。Go 紧密结合 goroutine，让网络事件转化为对网络文件描述符感兴趣的 goroutine，并将其注入运行队列，伺机调度；Tokio 基于唤醒机制催动 Executor 去轮询每个 Future，每个 Future 都被编译为一个状态机。
3. 异步编程是对并发模型的考验。程序必须有能力挂起不能继续的任务，转而执行其它的任务，因为网络文件描述符非阻塞的特性，异步网络 I/O 才会成为可能。
4. 普通文件 I/O 的异步解决方案需要等待 io_uring 的普及。

关于“**异步编程是对并发模型的考验**"这一点，可以从 Tokio 官方对于异步编程的论述中得到印证：

```
What is asynchronous programming?

Most computer programs are executed in the same order in which they are written. The first line executes, then the next, and so on. With synchronous programming, when a program encounters an operation that cannot be completed immediately, it will block until the operation completes. For example, establishing a TCP connection requires an exchange with a peer over the network, which can take a sizeable amount of time. During this time, the thread is blocked.

With asynchronous programming, operations that cannot complete immediately are suspended to the background. The thread is not blocked, and can continue running other things. Once the operation completes, the task is unsuspended and continues processing from where it left off. Our example from before only has one task, so nothing happens while it is suspended, but asynchronous programs typically have many such tasks.

Although asynchronous programming can result in faster applications, it often results in much more complicated programs. The programmer is required to track all the state necessary to resume work once the asynchronous operation completes. Historically, this is a tedious and error-prone task.
```

**With asynchronous programming, operations that cannot complete immediately are suspended to the background**，不能继续的任务，要被扔到后台。

**The thread is not blocked, and can continue running other things**，底层线程不因此而阻塞，继续运行其它任务。

**Once the operation completes, the task is unsuspended and continues processing from where it left off**，当被异步的操作完成后，被终止的任务恢复执行。

所以，“**异步编程**”和“**异步**”这两个概念是有所区别的，“**异步**”是一种特性，“**异步编程**”是基于此特性演化出的编程范式。

“异步”并不会使单个任务加速，Netpoller 和 Tokio 都是为了解决高并发网络 I/O 而生的，并不会加速某个单一的任务，而是让多个任务在有限的 CPU资源下，跑出接近单个任务的响应时间，本质上是对 CPU 的充分利用。

在我看来，Tokio 口中异步编程的复杂性，完全来自于对性能考量下的权衡，它解决的是高并发网络 I/O 的问题，而不是并发的问题。不同的设计哲学，让它们在并发 CPU-bound 任务上走向了不同的目标。

我们看一个 1 万个并发递归计算斐波那契数列的例子，运行环境为 8 核，16G 内存，Manjaro Linux：

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func fib(n int) int {
	if n == 0 || n == 1 {
		return n
	}
	return fib(n-1) + fib(n-2)
}

func main() {
	//runtime.GOMAXPROCS(24)
	ch := make(chan float64, 8)
	done := make(chan struct{})

	before := time.Now()
	var wg sync.WaitGroup

	for i := 0; i < 10000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			_ = fib(40)
			el := time.Since(before).Seconds()
			ch <- el
		}()
	}

	go func() {
		for {
			select {
			case v := <-ch:
				fmt.Printf("耗时：%fs", v)
			case <-done:
				return
			}
		}
	}()
	wg.Wait()
	elapsed := time.Since(before)
	close(done)
	fmt.Println(elapsed, "total,", elapsed/10000, "avg per iteration")
}
```

运行结果：

> 耗时区间：676 s ~ 814 s    总耗时：814s   平均耗时：81.4 ms 

再来看看 Tokio，依据 Tokio 官网建议，计算型任务使用[`spawn_blocking`](https://docs.rs/tokio/1.29.1/tokio/task/fn.spawn_blocking.html) ，它会将任务派发到一个专门的线程池，依据并发任务数量，这个线程池会增长到 500：

```rust
use std::time::Instant;
use tokio::task;
use futures::future::{self, join_all};
use std::sync::mpsc;
use std::fs::File;
use std::io::{Read, Write};


#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel();
    let start = Instant::now();
    for i in 1..=10000 {
        let tt = tx.clone();
        task::spawn_blocking(move|| {
            let r = fib(40);
            let duration = start.elapsed();
            let mut dev_null = File::create("/dev/null").unwrap();
            dev_null.write(&r.to_le_bytes()).unwrap();
            tt.send(duration.as_secs_f64() ).unwrap();
            drop(tt);
            0
        });
    }

    drop(tx);
    for received in rx {
        print!("耗时: {:?}", received);
    }

    let duration = start.elapsed();
    println!("总耗时: {:?}", duration);
    println!("平均耗时: {:?}", duration / 10000);
}


fn fib(n: u64) -> u64 {
    if n == 0 || n == 1 {
        return n;
    }
    fib(n - 1) + fib(n - 2)
}
```

运行结果：

> 耗时区间：0.6 s ~ 564 s    总耗时：564s   平均耗时：56.4 ms 

Tokio 建议使用 Rayon 来运行 CPU-bound 的任务，我们再来看一下，Rayon 的版本：

```rust
use rayon::prelude::*;
use std::time::Instant;
use std::sync::mpsc;

fn fib(n: u32) -> u32 {
    if n < 2 {
        return n;
    }
    fib(n - 2) + fib(n - 1)
}

// 使用rayon的并行迭代器来重复计算一万次
fn main() {
    let (tx, rx) = mpsc::channel();
    let start = Instant::now();
    let mut results = vec![0; 10000];
    results.par_iter_mut().for_each_with(tx,|tx,r| {
        *r = fib(40);
        let duration = start.elapsed();
        let tt = tx.clone();
        tt.send(duration.as_secs_f64() ).unwrap();
        drop(tt);
    });

    for received in rx {
        print!("耗时: {:?}", received);
    }

    let duration = start.elapsed();
    println!("总耗时: {:?}", duration);
    println!("平均耗时: {:?}", duration / 10000);
}
```

运行结果：

> 耗时区间：0.00420079 s ~ 503 s    总耗时：507s   平均耗时：50 ms 

Rayon 默认只使用与 CPU 数量相同的线程来执行任务，执行效率反而比 Tokio 略好，Tokio 因为启动了大量的线程，导致我的电脑已无法正常相应键鼠了。

| Object | 区间                 | 总耗时 | 平均    |
| ------ | -------------------- | ------ | ------- |
| Go     | 676 s ~ 814 s        | 814 s  | 81.4 ms |
| Tokio  | 0.6 s ~ 564 s        | 564 s  | 56.4 ms |
| Rayon  | 0.00420079 s ~ 503 s | 507 s  | 50 ms   |

我们这里主要观察每个任务的耗时区间，因为 Go 和 Rust 的定位不同，性能也有差距，所以比较总耗时并没有意义。

这里有意思的是 Go 的所有任务耗时趋向于“平均”，而 Rust 的两个框架是在每个线程上串行执行任务，任务耗时如同信号图标📶，由低到高渐进式增长。

所以，如果计算任务之间没有依赖，更看重总的响应时间的话，使用与 CPU 核数相当的线程池进行并行计算能得到最佳效果；如果任务是并发的，更加注重单个任务的响应时间，类似于 Go 的并发模型可能是更好的选择。

Go 是为并发而生的语言，所以你会发现，在编写 Go 代码的时候，你根本不用去考虑并发任务是计算型还是I/O型的，在其并发模型下所有的任务都能得到很好的处理；而缺乏完善调度运行时的线程池就要注意很多了，你要小心不能在异步任务中写太多计算的代码，甚至有博主指出：[在进入`.await`之前，最好不要超过10 ~ 100 微秒](https://ryhl.io/blog/async-what-is-blocking/)。

道理不难理解，以 Tokio 为例，虽然可以运行 CPU 密集型任务，但是官方很明确的说你要新开实例去运行，不要饿死 I/O 任务，显然这是因为运行时缺乏调度能力的折中方案。

CPU 密集型任务属于会阻塞 executor 线程的任务，容易霸占 CPU 而饿死其它任务，此时只能靠手动 yield 来让出 CPU，给其它任务以运行的机会；而网络 I/O 之所以适合，完全是因为有非阻塞特性和 Reactor 的存在，每个 I/O 读写点都是一次 yield 的机会！

不难想见的是，Tokio 虽然适合网络 I/O 型并发，但是也要在 I/O 任务里小心地控制计算型代码的时间，否则会导致运行时任务调度不均，长时间阻塞其它任务的运行。