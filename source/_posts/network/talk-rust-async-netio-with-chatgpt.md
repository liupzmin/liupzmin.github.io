---
title: 对话 ChatGPT 理解 Rust 异步网络 io
date: 2023-06-08 21:18:20
tags: 
	- async/await
    - tokio
    - rust
categories:
    - network
    - rust

---

从已知理解未知是人类的自然思维模式，但这很容易让人陷入认知偏差的思维陷阱。我在理解 Rust 异步网络 io 时就陷入了思维定式，总是试图以线程切换或者 goroutine 协程切换的模式来理解 Rust 协程的调度，而这两种调度方式有着天壤之别。

以 tokio 为首的 Rust 异步运行时，为用户呈现了一种无栈协程，既然都是协程，那么拿来与 go 的协程作比较是很自然的一种想法，加之我对协程切换、CPU 执行流改变近乎迷恋。所以，从 go 的角度去推测 Rust 也就情有可原了。

很多文章或者名家专栏在介绍异步 io 时，无一例外都是从 Richard Stevens 老爷子在 《UNIX环境高级编程》中 对于 io 模型的介绍开始的，然后介绍 Linux 下的 epoll 多路复用，之后引出 reactor 模式，最后进入对 Rust 异步 io 的介绍。这其中有很大的断层，并不是很容易理解其中的脉络，所以，Rust 的异步 io 模型一直没搞明白，直到 chatgpt 的出现......

我先问了它go 的netpoller 和 tokio 两种网络模型的异同。

1. **请比较一下go的netpoller网络模型和tokio的网络模型的异同**

   >Go的netpoller网络模型是基于epoll/kqueue/iocp等多路复用技术的，它使用同步编程模式达到异步执行的效果。它以网络描述符netFD为中心，当在一个netFD上读写遇到EAGAIN错误时，就将当前goroutine存储到这个netFD对应的PollDesc中，并调用gopark把当前goroutine给park住，直到这个netFD上再次发生读写事件，才将此goroutine给ready激活重新运行。
   >Tokio的网络模型也是基于poll模型的，它使用异步编程模式实现高效的任务调度。它提供了TCP、UDP等网络类型，在tokio::net模块中可以找到像TcpListener、TcpStream、UdpSocket这些类型。Tokio的任务由Tokio调度器管理的最小可执行单元，它需要实现Future trait，并在poll方法中返回Async::NotReady或Async::Ready状态。
   >两者的异同点有：
   >* Go使用同步编程模式，Tokio使用异步编程模式
   >* Go以netFD为中心管理网络操作，Tokio以Future为中心管理任务
   >* Go使用goroutine作为轻量级线程，Tokio使用task作为轻量级线程
   >* Go通过gopark和ready来控制goroutine的状态转换，Tokio通过Executor来调度task的状态转换
   
   这个回答不是很抽象，如果没有以前了解相关内容的话，并不太好理解。但是解释大致是正确的，“Go使用同步编程模式，Tokio使用异步编程模式”这句话并不准确，Go 是通过netpoller 和 goroutine 的配合为用户提供了编写同步编码的接口，网络的 io 本质上还是基于 epoll 的非阻塞 io，而 Tokio 就需要用户真正的编写异步代码了。

2. **请介绍一下tokio的future使用poll来推进执行的内部原理**

   因为事先了解过 Rust 的 funture 、调度器、执行器体系，所以我直接问其内部原理

   > Tokio的future使用poll来推进执行的内部原理是：
   > * Future是一个trait，它有一个poll方法，用于检查future是否已经完成或者还需要等待
   > * poll方法返回一个Poll枚举类型，它有三个可能的值：Ok(Async::Ready(t))表示future已经成功完成并返回t；Ok(Async::NotReady)表示future还没有准备好完成；Err(e)表示future已经失败并返回e
   > * 当future被创建时，它会注册自己到Tokio调度器中，并告诉调度器它依赖哪些资源（比如TCP socket）来完成
   > * 当资源变得可用时（比如TCP socket收到数据），调度器会通知future，并调用它的poll方法
   > * 如果poll方法返回Ok(Async::Ready(t))，那么future就完成了，并将t传递给下一个future；如果poll方法返回Ok(Async::NotReady)，那么future就继续等待资源变得可用；如果poll方法返回Err(e)，那么future就失败了，并将e传递给下一个future
   > 这样，Tokio可以通过不断地调用poll方法来驱动异步任务的执行。
   
   这些回答流于表面，但是能起到帮我建立上下文的作用，然后继续追问。

3. **我以前不理解执行流是如何改变的，现在有些模糊的认识，你帮我参谋一下：对于一个 future 执行流的改变总是发生在对于一个异步函数的 await 的调用处，其它非 await 调用的语句是不会中断的，所以一个 future 只会在 await 处被中断和恢复，我理解的对吗？**

   之所以有此一问，是因为我还局限在线程切换和 goroutine 切换的思维定式当中，不论是线程还是 goroutine 都会被抢占，也就是说执行流在任务中的任何一行代码上都可能发生改变，被调度器保存上下文换上新的任务运行，既然 Rust 中也是任务，任务之间的切换没理由不遵循已经被验证过的设计方式。

   > 你的理解基本上是对的。await是一个语法糖，它会将异步函数分割成多个future，并在每个await处暂停和恢复执行流。但是有一点需要注意，就是如果一个future返回Ok(Async::Ready(t))，那么它不会被中断，而是直接继续执行下一个future。所以只有当一个future返回Ok(Async::NotReady)时，它才会被中断，并等待资源变得可用再恢复。
   
   很好，这个回答验证了一个猜想，那就是 Rust 的协程无法被 tokio 运行时自由抢占，抢占的任务依赖于底层的线程调度。

4. **嗯，我明白，现在我们再深入一下，tokio 是如何在 await 返 Ok(Async::NotReady) 时保护现场的？又是如何恢复现场的呢？**

   我依旧对于执行流的改变耿耿于怀，我的认知是：**当 await 无法完成时，执行器会转而执行其他的任务，而且会在合适的时机再次回到上次的中断处继续。** 后面会证明这个想法是错误的

   > Tokio是如何在await返回Ok(Async::NotReady)时保护现场的，又是如何恢复现场的呢：
   > * 当一个future被创建时，它会被包装在一个Task结构体中，这个结构体包含了future的状态和上下文信息
   > * 当一个future被调度器调用poll方法时，它会获取Task的引用，并将其传递给future
   > * 当一个future返回Ok(Async::NotReady)时，它会创建一个Waker对象，并将其存储在Task中。Waker对象是一个通知机制，它可以让future告诉调度器什么时候再次poll自己
   > * 当资源变得可用时，Waker对象会被触发，并通知调度器再次poll对应的future
   > * 调度器会从Task中获取future的状态和上下文信息，并恢复执行流
   > 这样，Tokio可以通过Task和Waker来保护和恢复现场。
   
   表示对这个回答无感，可能是了解的不够深，无法 get 到正确的点，那就继续发问

5. **还有一个疑问：poll 函数到底怎么推进 future 的执行呢？如果实现 future 的这个异步函数中有很多普通的语句，这些语句怎么能通过 poll 得到执行呢？**

   我其实隐约感觉到了 future 的执行比较特别，因为很多资料的介绍都是说 future 的执行是在被执行器一次次的 poll 中前进的。

   > poll函数到底怎么推进future的执行呢：
   > * poll函数是Future trait的核心方法，它会尝试将future推进到完成状态
   > * 如果future已经完成，poll函数会返回Poll::Ready(t)，其中t是future的结果
   > * 如果future还没有完成，poll函数会返回Poll::Pending，并安排一个wake()函数在future准备好继续执行时被调用
   > * 当一个异步函数被调用时，它会被编译器分割成多个future，并在每个await处暂停和恢复执行流
   > * 每个future都有一个状态机，它记录了当前的执行位置和局部变量
   > * 当poll函数被调用时，它会根据状态机来决定接下来要执行哪些语句，并更新状态机
   > * 如果遇到一个await表达式，poll函数会检查被等待的future是否已经完成
   > * 如果已经完成，poll函数会获取结果并继续执行下一个语句；如果还没有完成，poll函数会返回Poll::Pending，并等待wake()函数再次唤醒自己
   > 这样，poll函数可以通过状态机来控制异步任务的执行流程
   
   不知道它说的这些哪些有问题，所以我想要一个被编译过后的实现了 future trait的异步函数内容，先看看再说。

6. **能给我一个poll函数执行future的示例吗？**

   ```rust
   use std::future::Future;
   use std::pin::Pin;
   use std::task::{Context, Poll};
   
   struct MyFuture {
       // some fields
   }
   
   impl Future for MyFuture {
       type Output = i32;
   
       fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
           // do some work
           if self.is_ready() {
               // return the result
               Poll::Ready(42)
           } else {
               // register a waker
               cx.waker().wake_by_ref();
               // return pending
               Poll::Pending
           }
       }
   }
   ```

   因为它没给出原来的异步函数内容，所以我有点生气（因为自己太平庸了，后来才明白 `// do some work` 就是异步函数中的普通语句）。

7. **但是这个异步函数的内容是什么？我没看到poll执行了异步函数的内容**

   

   ```异步函数的内容是由编译器自动分割成多个future的，每个future都有一个poll方法，它会执行一部分异步函数的内容，直到遇到一个await表达式。例如，下面这个异步函数：```

   ```rust
   async fn foo() {
       // do something
       let x = await bar();
       // do something else
   }
   ```
   ```会被编译器转换成类似这样的代码：```

   ```rust
   enum FooFuture {
       State1,
       State2(Pin<Box<dyn Future<Output = i32>>>),
   }
   
   impl Future for FooFuture {
       type Output = ();
   
       fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
           match self {
               FooFuture::State1 => {
                   // do something
                   let fut = Box::pin(bar());
                   *self = FooFuture::State2(fut);
                   cx.waker().wake_by_ref();
                   Poll::Pending
               }
               FooFuture::State2(fut) => match fut.as_mut().poll(cx) {
                   Poll::Ready(x) => {
                       // do something else
                       Poll::Ready(())
                   }
                   Poll::Pending => Poll::Pending,
               },
           }
       }
   }
   ```
   ```这样，你可以看到poll执行了异步函数的内容，并在每个await处暂停和恢复执行流。```

   看到这里我才恍然大悟，以前没有细细追究编译器将异步函数编译为一个状态机到底意味着什么。

8. **你的意思是编译器会把异步函数中的语句拆分到不同的状态机状态中，poll 的时候这些代码就会被执行是吗?**

   看，多么痛的领悟~

   > 是的，你理解得很对。编译器会把异步函数中的语句拆分到不同的状态机状态中，poll的时候这些代码就会被执行。这样，异步函数就可以实现非阻塞的并发操作。
   
   但是，这仅仅是发现了新大陆而已，不能忘记为何出发

9. **那 await 放弃执行权是根据一个非阻塞文件描述符返回 EAGAIN 标志吗？**

   根据 go 的经验，我问了一个底层的问题，想试着在Rust上缝合这一块儿内容

   > await放弃执行权不是根据一个非阻塞文件描述符返回EAGAIN标志，而是根据future的poll方法返回Poll::Pending。当future的poll方法返回Poll::Pending时，它会注册一个waker，这个waker会在future可以继续执行时唤醒它。这样，异步函数就可以在等待某些操作完成时让出CPU资源给其他任务。
   
   它显然没理解我的意思，是我太跳跃了，那就给它点提示。

10. **那 poll 方法根据什么返回一个 Poll::Pending 呢？**

    > poll方法根据future是否完成来返回一个Poll::Pending。如果future还没有完成，poll会返回Poll::Pending，并存储一个从当前Context复制的Waker的克隆。这个Waker会在future可以继续执行时唤醒它。例如，一个等待socket变为可读的future会调用Waker的.clone()方法并存储它。当socket变为可读时，Waker会被唤醒，然后再次调用poll方法。
    
    显然不能让它这么轻易几句就打发了，我决定要一个具体的例子。

11. **给我一个读取 socket 产生 pending 的例子，我要看看这个函数生成的 poll 内容，因为我对如何发出 socket 读取，并且不会陷入系统调用发生阻塞感兴趣。我是想要 tokio 异步编程中，一个异步函数读取 socket，被编译器编译出来的 poll 方法的内容，可以吗？**

    你总是需要小心描述你的要求，这对中文来说是个很大的挑战，但是我又没有耐心一点点儿地抠英文，学渣的无奈啊！

    ```tokio异步编程中，一个异步函数读取socket，被编译器编译出来的poll方法的内容可能是这样的： ```

    ```rust 
    use std::future::Future;
    use std::pin::Pin;
    use std::task::{Context, Poll};
    use tokio::io::{AsyncReadExt, AsyncWriteExt};
    use tokio::net::TcpStream;
    
    // 定义一个异步函数，从socket中读取数据，并写回去
    async fn echo(mut stream: TcpStream) -> Result<(), Box<dyn std::error::Error>> {
        let mut buf = [0; 1024];
    
        loop {
            // 读取socket，如果没有数据可读，返回Poll::Pending，并注册一个waker
            let n = stream.read(&mut buf).await?;
    
            if n == 0 {
                // socket已经关闭，退出循环
                break;
            }
    
            // 将读取到的数据写回socket
            stream.write_all(&buf[0..n]).await?;
        }
    
        Ok(())
    }
    
    // 编译器会将上面的异步函数转换为一个实现了Future特质的结构体类型
    struct Echo {
        stream: TcpStream,
        buf: [u8; 1024],
        state: State,
    }
    
    enum State {
        Reading { n: usize },
        Writing { n: usize },
    }
    
    impl Future for Echo {
        type Output = Result<(), Box<dyn std::error::Error>>;
    
        fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
            loop {
                match self.state {
                    State::Reading { mut n } => {
                        // 尝试从socket中读取数据
                        let result = Pin::new(&mut self.stream).poll_read(cx, &mut self.buf[n..]);
    
                        match result {
                            // 如果成功读取到数据，更新n和state，并继续循环
                            Poll::Ready(Ok(m)) => {
                                if m == 0 {
                                    // socket已经关闭，返回Poll::Ready(Ok(()))
                                    return Poll::Ready(Ok(()));
                                } else {
                                    n += m;
                                    self.state = State::Writing { n };
                                }
                            }
                            // 如果没有数据可读，返回Poll::Pending，并注册一个waker
                            Poll::Pending => return Poll::Pending,
                            // 如果出现错误，返回Poll::Ready(Err(e))
                            Poll::Ready(Err(e)) => return Poll::Ready(Err(e.into())),
                        }
                    }
                    State::Writing { mut n } => {
                        // 尝试将数据写回socket
                        let result = Pin::new(&mut self.stream).poll_write(cx, &self.buf[..n]);
    
                        match result {
                            // 如果成功写入数据，更新n和state，并继续循环
                            Poll::Ready(Ok(m)) => {
                                if m == n {
                                    // 数据已经全部写入，重置n和state，并继续循环
                                    n = 0;
                                    self.state = State ::Reading { n };
                                } else {
                                    // 数据还有剩余部分没有写入，减少n并保持state不变，并继续循环
                                    n -= m;
                                    self.state = State ::Writing { n };
                                }
                            }
                            // 如果没有空间可写，返回Poll ::Pending，并注册一个waker
                            Poll ::Pending => return Poll ::Pending,
                            // 如果出现错误，返回Poll ::Ready (Err (e))
                            Poll ::Ready (Err (e)) => return Poll ::Ready (Err (e.into())),
                        }
                    }
                }
            }
        }
    }
    ```

    从这个例子就可以看出对于有 2 个 await 的异步函数，编译过后的状态机模样了

12. **tokio::net::TcpStream.poll_read 是如何做到不阻塞的？**

    我选了 read 方法问它是如何做到不阻塞的

    > tokio::net::TcpStream.poll_read 是一个异步读取方法，它可以从一个 TCP 流中接收数据，但不会等待新数据到达。它是非阻塞的，因为它不会挂起当前线程，而是返回一个 Poll 结果，表示操作是否完成或需要重试 。要使用这个方法，你需要确保 TCP 流是在非阻塞模式下创建的
    
    显然，它不知道我想要更底层的内容

13. **可以给出源码吗？它是在收到 EAGAIN  时返回一个 pending 吗？**

    > 你可以在这里查看 tokio::net::TcpStream.poll_read 的源码。它是一个 trait 方法，它调用了 TcpStream 的 poll_read_priv 方法，该方法使用了 PollEvented 来封装一个 mio::net::TcpStream。如果收到 EAGAIN 错误，它会返回一个 Pending 结果，并注册当前任务的唤醒器，以便在流变得可读时再次轮询。
    
    OK，问到我想要的答案了，接下来转移话题。

14. **我有个疑问，Rust 不内置异步运行时，那么编译器在编译 await 时，如何在返回 pending 时放弃 future 的执行权将执行流引向 executor 代码的呢？**

    之所以问出这种问题，是因为我对Rust的异步运行时并不了解，也不清楚 future 是如何被催动的

    > 这是一个很好的问题。Rust 的 async/await 特性是基于 trait 的，这意味着它不依赖于任何特定的运行时或执行器。编译器在编译 await 时，会将当前函数转换为一个状态机，其中每个 await 点都是一个状态转换。当一个 future 被轮询时，它会执行到下一个 await 点，然后返回 Pending 或 Ready 结果。执行器负责轮询 future，并在它们变得可用时恢复它们的执行。因此，编译器不需要知道执行器的细节，只需要生成合适的状态机和轮询逻辑。
    
15. **奥，是不是执行器负责执行 future，当 future 中的 await 返回 pending 时，执行器放弃执行当前的 future，转而去寻找其它的 future 继续轮询，我理解的对吗？**

    > 是的，你的理解是对的。执行器就是负责管理和调度 future 的组件，它可以根据不同的策略来选择哪些 future 要轮询或暂停。执行器可以是单线程或多线程的，也可以支持不同的并发模型，如协程、线程池、事件循环等。
    
    ChatGPT 的优点就是可以节约你很多研究时间，当看到被肯定时，还是挺激动地，但是不能止步，终极问题还没解决

16. **详细讲解一下future是如何保存现场和恢复现场的？**

    还是绕回到这个话题上来了，因为我的思维定式就是任务切换，要保存上下文的，IP，SP这些寄存器，及其他通用寄存器

    > future 是一种抽象的类型，它表示一个异步计算的结果，它可以在某个时刻被轮询并返回一个值或者继续挂起。future 本身不保存任何现场，它只是一个状态机，它根据输入的上下文（Context）和缓冲区（ReadBuf）来决定下一步的动作。当 future 被轮询时，它会执行到下一个 await 点，并将自己的状态和局部变量封装在一个 Pin<Box<…>> 中返回给执行器。这个 Pin<Box<…>> 就是 future 的保存的现场，它可以在之后被再次传入 future 的 poll 方法来恢复执行。
    
    这个回答的好处是引出了`Pin<Box<…>>`,虽然细节不太清楚，但以前就混过脸熟，只要记住上下文放在这里就行。

17. **那么为什么要用 Pin<Box<…>> 来封装 future 上下文呢？**

    > 是为了保证 future 的内存地址不会改变，从而避免破坏 future 的安全性和正确性。Pin 是一个指针类型，它表示一个不能被移动的值。Box 是一个堆分配的智能指针，它可以存储任意类型的值 。将 future 包装在 Pin<Box<…>> 中，就相当于将 future 放在堆上，并且保证它不会被移动或者删除 。这样，future 就可以安全地保存自己的状态和局部变量，并且可以在任何时候被恢复执行。
    
    这个时候感觉的一点匪夷所思的是，执行器在一个线程上不断轮询若干个 future，而且 Rust 的协程是无栈协程，那么代码执行就只用到线程栈，按照前面的说法使用 Pin 保存现场，那么是否future的执行不依赖于栈中的内容，否则，在多个future之间跳来跳去岂不是乱了套？

18. **那future是不是对堆栈的依赖不大？**

    > 这个问题没有一个确定的答案，因为 future 的实现和使用方式可能不同。一般来说，future 会尽量减少对堆栈的依赖，因为堆栈是有限的资源，而 future 可能会有很多。future 通常会将自己的状态和局部变量存储在堆上，或者使用一些优化技术来避免不必要的内存分配。但是，future 也可能会使用一些堆栈空间来执行一些临时的计算或调用其他函数。因此，future 对堆栈的依赖程度取决于具体的情况和设计。
    
19. **那 future 在保存现场的时候不考虑堆栈中的内容吗？**

    > future 在保存现场的时候，只需要考虑自己的状态和局部变量，而不需要考虑堆栈中的内容。这是因为 future 的 poll 方法是一个纯函数，它不会修改堆栈中的任何数据，也不会依赖于堆栈中的任何数据。堆栈中的内容只是一些临时的计算或调用结果，它们在 future 挂起或恢复时并不重要。因此，future 只需要将自己封装在一个 Pin<Box<…>> 中，并将其返回给执行器，就可以实现现场的保存和恢复。
    
    至此，我大概理解 Rust 的异步网络 io 模型了。

20. **我有点理解了，每次 poll 或许会使用堆栈，但 poll 完之后，poll 函数返回堆栈内容就弹出了，但是上一次执行的状态会被保存在 pin 结构中，下一次 poll 的时候，会基于 pin 中的内容决定状态机的走向，进而一步步推动整个 future 逻辑的前行，是不是？**

    > 是的，你的理解是正确的。future 的 poll 方法就是一个状态机，它会根据自己的状态和输入的上下文来决定下一步的动作。每次 poll 完成后，future 会将自己的状态和局部变量保存在 pin 结构中，并返回一个值表示 future 是否已经完成或者还需要继续轮询。下一次 poll 的时候，future 会从 pin 结构中恢复自己的状态和局部变量，并继续执行到下一个 await 点或者结束。这样，future 就可以实现异步计算的逻辑。
    
    

因为我用的是NewBing，所以20句就截止了，所幸问题已经聊清楚了，其实类似于 tokio 之类的异步运行时在调度协程时并没有我惯性思维中以为的那种保存寄存器恢复寄存器的过程，它的核心在于每个异步函数都会被编译为一个状态机，这个状态机是玄机所在，再加上一个 pin 的结构来保存和恢复一些中间变量或者说状态。这样一来，依靠 epoll 事件，调度器和执行器的配合，不停地去 poll 这个状态机就行了，事件的发生会导致状态的改变，从而执行不同状态下的代码。这种架构理解起来不算直观，不如go的协程好理解。

go 的 netpoller 网络模型是与 goroutine 的密切配合实现的，作为 reactor，netpoller 只是根据网络文件描述符的状态修改关注它的 goroutine 的状态，剩余的工作交给调度器；以同步模式使用的网络 socket 读取代码，会在一个非阻塞的 socket 上获得 EAGAIN 错误时执行 gopark 进入调度循环，放弃执行，等待再次被调度。

至于 tokio 的网络模型，虽然 reactor 的底层逻辑都是使用 epoll 事件循环，但是当被激活时的行为是完全不同的，tokio 的 reactor被激活时会调用 executor 事先注册好的 waker 函数来通知 executor 去轮询 future，猜测这种模型的运行时要比 go 的运行时要小，加上 future 是无栈协程，理论上在处理大量网络连接时的效率要高。但是，tokio 也仅仅适用于 io 相关的操作，无法像 go 的goroutine 一样万事皆可异步！
