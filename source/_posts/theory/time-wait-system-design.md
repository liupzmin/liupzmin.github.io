---
title: 译:TIME_WAIT及其对协议和可伸缩客户端服务器系统的设计实现
date: 2020-01-09 16:23:47
tags: computer theory
categories:
    - network
    - computer theory
    - TCP/IP
---

**看了左耳朵耗子推荐的这篇 [TIME_WAIT and its design implications for protocols and scalable client server systems](http://www.serverframework.com/asynchronousevents/2011/01/time-wait-and-its-design-implications-for-protocols-and-scalable-servers.html)　讲 TIME_WAIT 的文章，感觉比较清晰，就翻译出来，在这里记录了一下，其实其中内容在《TCP/IP详解》和《UNIX网络编程》中有更详细的讲解。但是文章仍然值得一读，它从程序设计角度描述了应该如何避免 TIME_WAIT 的困扰，但对 TCP 本身着墨不是很多，比如要理解 TIME_WAIT 绕不开 ISN 和 MSL，以及在消除了 TIME_WAIT 之后如何解决延迟报文的问题也未予以详述，我会在我的下一篇文章中就 TIME_WAIT 问题再展开论述，增加自己对 TCP 掌握的熟练度，并结合 ISN、MSL、TIMESTAMP 等概念进行说明。**

***注：才疏学浅，翻译如有错漏之处，还望斧正。***

> When building TCP client server systems it's easy to make simple mistakes which can severely limit scalability. One of these mistakes is failing to take into account the TIME_WAIT state. In this blog post I'll explain why TIME_WAIT exists, the problems that it can cause, how you can work around it, and when you shouldn't.

当构建基于 TCP 的 C/S系统的时候，非常容易犯一些简单的错误，这些错误会严重限制系统的可伸缩性。其中之一就是对 TIME_WAIT 状态疏于考虑。在此博客文章中，我将说明 TIME_WAIT 存在的原因、它可能引起的问题、该如何解决它以及何时不应该考虑解决它。

> TIME_WAIT is an often misunderstood state in the TCP state transition diagram. It's a state that some sockets can enter and remain in for a relatively long length of time, if you have enough socket's in TIME_WAIT then your ability to create new socket connections may be affected and this can affect the scalability of your client server system. There is often some misunderstanding about how and why a socket ends up in TIME_WAIT in the first place, there shouldn't be, it's not magical. As can be seen from the TCP state transition diagram below, TIME_WAIT is the final state that TCP clients usually end up in.

TIME_WAIT 是 TCP 协议状态转换图中一个经常被误解的状态。一些 socket 会进入这个状态并且保持这个状态相当长的一段时间，如果你有相当多的 socket 处于 TIME_WAIT 状态，那么你创建新连接的能力可能会受到影响，这进而会影响到你 C/S 系统的伸缩性。通常，对于套接字如何以及为什么最终以 TIME_WAIT 结尾的情况经常会产生一些误解，不应该是这样，它不是魔术。如你将在下面的转换图中看到的那样，TIME_WAIT 通常是 TCP 客户端的最终状态（指通常情况下都是客户端主动断开连接）。

![TCP StateTransitionDiagram NormalTransitions](http://qiniu.liupzmin.com/TCP-StateTransitionDiagram-NormalTransitions.png)

> Although the state transition diagram shows TIME_WAIT as the final state for clients it doesn't have to be the client that ends up in TIME_WAIT. In fact, it's the final state that the peer that initiates the "active close" ends up in and this can be either the client or the server. So, what does it mean to issue the "active close"?

虽然状态转换图中展示的 TIME_WAIT 是客户端的最终状态，但是以 TIME_WAIT状态结束并不要求一定是客户端。事实上，TIME_WAIT 是发出 “主动关闭” 一端的最终状态，它既可以是客户端，也可以是服务端。那么问题来了，什么是“主动关闭”？

> A TCP peer initiates an "active close" if it is the first peer to call Close() on the connection. In many protocols and client/server designs this is the client. In HTTP and FTP servers this is often the server. The actual sequence of events that leads to a peer ending up in TIME_WAIT is as follows.

在 TCP 连接中，率先调用 close() 的一方就是 “主动关闭” 的发起方。在很多协议实现和C/S系统中通常是 client 主动断开连接。HTTP（只在 HTTP 1.0 ） 和 FTP 里是 server 主动关闭。主动端走向 TIME_WAIT 的时序图如下：

![TCP StateTransitionDiagram Closure Transitions](http://qiniu.liupzmin.com/TCP-StateTransitionDiagram-ClosureTransitions.png)

> Now that we know how a socket ends up in TIME_WAIT it's useful to understand why this state exists and why it can be a potential problem.

现在我们知道了一个socket 是如何以 TIME_WAIT 状态结束的，这对我们理解 TIME_WAIT 状态存在的意义以及为何它可能成为潜在的问题非常有益。

> TIME_WAIT is often also known as the 2MSL wait state. This is because the socket that transitions to TIME_WAIT stays there for a period that is 2 x Maximum Segment Lifetime in duration. The MSL is the maximum amount of time that any segment, for all intents and purposes a datagram that forms part of the TCP protocol, can remain valid on the network before being discarded. This time limit is ultimately bounded by the TTL field in the IP datagram that is used to transmit the TCP segment. Different implementations select different values for MSL and common values are 30 seconds, 1 minute or 2 minutes. RFC 793 specifies MSL as 2 minutes and Windows systems default to this value but can be tuned using the TcpTimedWaitDelay registry setting.

TIME_WAIT state 也称 2MSL wait state。这是因为转换为 TIME_WAIT 的套接字在此停留的时间为 2 个最大报文生存时间（MSL）。MSL 是 TCP 协议中任何报文在网络上最大的生存时间，任何超过这个时间的数据都将被丢弃。MSL 是由网络层的 IP 包中的 TTL来保证的（MSL 不与TTL 绝对相等，事实上 MSL 是 TCP 协议的假设基础）。不同的协议实现规定的 MSL 都不相同，通常为 30 seconds, 1 minute or 2 minutes。RFC 793 规定 MSL 为 2 minutes，Linux 默认为 30 seconds，windows 默认为 2 minutes，但是可以通过修改注册表 TcpTimedWaitDelay 的值来自定义。

> The reason that TIME_WAIT can affect system scalability is that one socket in a TCP connection that is shutdown cleanly will stay in the TIME_WAIT state for around 4 minutes. If many connections are being opened and closed quickly then socket's in TIME_WAIT may begin to accumulate on a system; you can view sockets in TIME_WAIT using netstat. There are a finite number of socket connections that can be established at one time and one of the things that limits this number is the number of available local ports. If too many sockets are in TIME_WAIT you will find it difficult to establish new outbound connections due to there being a lack of local ports that can be used for the new connections. But why does TIME_WAIT exist at all?

TIME_WAIT 之所以会影响系统的伸缩性是因为连接正常关闭下进入这个状态的 socket 会持续 4 minutes 的时间（根据不同的实现，TIME_WAIT 时间在1 ~ 4 minutes之间）。如果有大量的连接被打开然后随即关闭的话，那么停留在 TIME_WAIT 状态的 socket 就会在操作系统上积累；你可以通过 netstat 来观察 TIME_WAIT 状态的 socket。一次只能建立有限数量的套接字连接，而限制此数量的因素之一是就可用的本地端口数。如果太多的 socket 处于 TIME_WAIT 状态，你会发现很难再建立新的出站连接，原因是缺少可用于新连接的本地端口。但是，为什么 TIME_WAIT 会存在呢？

> There are two reasons for the TIME_WAIT state. The first is to prevent delayed segments from one connection being misinterpreted as being part of a subsequent connection. Any segments that arrive whilst a connection is in the 2MSL wait state are discarded.

TIME_WAIT 的存在主要有两个原因，第一个就是阻止前一个连接的延时报文被后续的连接错误接收（相同的五元组，不同的实例称为“化身”，这里指要阻止之前“化身”的报文被当前的连接错误接收）。一个连接处于 2 MSL 等待状态时，任何抵达的报文都将被丢弃（治理应该主要指数据报文，对与 FIN 报文还是要接收的，以便重新发送 ACK ，重新开始 2MSL 计时）。

![TIME_WAIT why](http://qiniu.liupzmin.com/TIME_WAIT-why.png)

> In the diagram above we have two connections from end point 1 to end point 2. The address and port of each end point is the same in each connection. The first connection terminates with the active close initiated by end point 2. If end point 2 wasn't kept in TIME_WAIT for long enough to ensure that all segments from the previous connection had been invalidated then a delayed segment (with appropriate sequence numbers) could be mistaken for part of the second connection...

上图有两个从 end point 1 到 end point 2 的连接，这两个连接的五元组相同（双方地址和端口相同）。第一个连接因 end point 2 主动关闭而终止，如果 end point 2 不在 TIME_WAIT 保持足够的时间以确保所有的报文在网络上失效的话，一个延迟的报文（拥有合适的 sequence number，意即对第二个连接来说是有效报文）会被第二个连接错误接收。

> Note that it is very unlikely that delayed segments will cause problems like this. Firstly the address and port of each end point needs to be the same; which is normally unlikely as the client's port is usually selected for you by the operating system from the ephemeral port range and thus changes between connections. Secondly, the sequence numbers for the delayed segments need to be valid in the new connection which is also unlikely. However, should both of these things occur then TIME_WAIT will prevent the new connection's data from being corrupted.

需要注意的是，像这样延迟的报文造成问题的情况是非常不太可能的。首先，地址和端口号就不太可能相同，因为客户端的端口号是由操作系统临时分配的。其次，报文的 sequence 号必须是有效的，这个也不太可能发生（涉及到 ISN 的循环 和 sequence number 的 warp around）。然而，如果这两种情况都发生，则 TIME_WAIT 将防止新连接的数据被破坏。

> The second reason for the TIME_WAIT state is to implement TCP's full-duplex connection termination reliably. If the final ACK from end point 2 is dropped then the end point 1 will resend the final FIN. If the connection had transitioned to CLOSED on end point 2 then the only response possible would be to send an RST as the retransmitted FIN would be unexpected. This would cause end point 1 to receive an error even though all data was transmitted correctly.

TIME_WAIT 存在的第二个原因就是：实现 TCP 的全双工连接可靠终止。如果 end point 2 最后发出的 ACK 丢失，依照 TCP 协议规则，end point 1 会重发 FIN。如果 end point 2 此时转为 CLOSED 状态，end point 2 的协议栈会响应一个 RST 报文，因为 这个重传的 FIN 并未被认可。 这将导致 end point 1 收获一个 error，即使所有的数据都已正确传输。

> Unfortunately the way some operating systems implement TIME_WAIT appears to be slightly naive. Only a connection which exactly matches the socket that's in TIME_WAIT need by blocked to give the protection that TIME_WAIT affords. This means a connection that is identified by client address, client port, server address and server port. However, some operating systems impose a more stringent restriction and prevent the local port number being reused whilst that port number is included in a connection that is in TIME_WAIT. If enough sockets end up in TIME_WAIT then new outbound connections cannot be established as there are no local ports left to allocate to the new connection.

不幸的是，某些操作系统实现 TIME_WAIT 的方式似乎有些幼稚。只有阻塞在 TIME_WAIT 中的套接字连接，才需要给予 TIME_WAIT 提供的保护。这意味着由客户端地址，客户端端口，服务器地址和服务器端口标识的连接。但是，某些操作系统施加了更严格的限制，并阻止了本地端口号被重用，而该端口号包含在 TIME_WAIT 的连接中（这句不是很明白，感觉意思是处于 TIME_WAIT 的 socket 涉及到的所有端口号都不能重用）。如果以 TIME_WAIT 结束的套接字足够多，那么将无法建立新的出站连接，因为没有剩余的本地端口可分配给新连接。

> Windows does not do this and only prevents outbound connections from being established which exactly match the connections in TIME_WAIT.

Windows 不会这样做，它仅阻止建立与 TIME_WAIT 中的连接完全匹配的出站连接。

> Inbound connections are less affected by TIME_WAIT. Whilst the a connection that is actively closed by a server goes into TIME_WAIT exactly as a client connection does the local port that the server is listening on is not prevented from being part of a new inbound connection. On Windows the well known port that the server is listening on can form part of subsequently accepted connections and if a new connection is established from a remote address and port that currently form part of a connection that is in TIME_WAIT for this local address and port then the connection is allowed as long as the new sequence number is larger than the final sequence number from the connection that is currently in TIME_WAIT. However, TIME_WAIT accumulation on a server may affect performance and resource usage as the connections that are in TIME_WAIT need to be timed out eventually, doing so requires some work and until the TIME_WAIT state ends the connection is still taking up (a small amount) of resources on the server.

入站的连接很少会受到 TIME_WAIT 的影响。服务端主动关闭连接进入 TIME_WAIT 的过程与客户端类似，但是不会阻止服务器正在侦听的本地端口成为新的入站连接的一部分。在 Windows 操作系统上，那些服务端侦听的知名端口可以用于创建新的入站连接，只要新的连接的 ISN （初始序列号）大于处于 TIME_WAIT 的那个连接的最终序列号，那么即使远端地址和远端端口也是当前 TIME_WAIT 连接的一部分， 这个连接也会创建成功（[RFC 1122](https://tools.ietf.org/html/rfc1122#page-87)中有相关描述，但是新“化身”的 ISN 号不敢保证一定会比老“化身”最后的序列号大，特别是中间在 NAT 网络的情况下）。但是，服务器上 TIME_WAIT 的累积可能会影响性能和资源使用，因为 TIME_WAIT 中的连接最终需要超时，这需要做一些工作，直到TIME_WAIT状态结束，连接仍会占用（少量）服务器上的资源。

> Given that TIME_WAIT affects outbound connection establishment due to the depletion of local port numbers and that these connections usually use local ports that are assigned automatically by the operating system from the ephemeral port range the first thing that you can do to improve the situation is make sure that you're using a decent sized ephemeral port range. On Windows you do this by adjusting the MaxUserPort registry setting; see here for details. Note that by default many Windows systems have an ephemeral port range of around 4000 which is likely too low for many client server systems.

鉴于 TIME_WAIT 会因本地端口号的耗尽而影响到出站连接的建立，并且这些连接通常使用由操作系统从临时端口范围自动分配的本地端口，因此，你可以做的第一件事就是确保正在使用适当大小的临时端口范围。在 Windows 上你可以通过修改注册表选项 MaxUserPort 来自定义。请注意，默认情况下，许多 Windows 系统的临时端口范围约为 4000，这对于许多 C/S 系统而言可能太低了。

> Whilst it's possible to reduce the length of time that socket's spend in TIME_WAIT this often doesn't actually help. Given that TIME_WAIT is only a problem when many connections are being established and actively closed, adjusting the 2MSL wait period often simply leads to a situation where more connections can be established and closed in a given time and so you have to continually adjust the 2MSL down until it's so low that you could begin to get problems due to delayed segments appearing to be part of later connections; this would only become likely if you were connecting to the same remote address and port and were using all of the local port range very quickly or if you connecting to the same remote address and port and were binding your local port to a fixed value.

尽管可以减少 socket 在 TIME_WAIT 中花费的时间，但这实际上并没有帮助。鉴于 TIME_WAIT 成为问题仅仅出现在有大量连接主动关闭的情况下，因此调整 2MSL 时间值只会让更多的连接不断的被建立和关闭，从而你会陷入不断的调小 2MSL 的恶性循环中，直到这个值小到让你开始遭遇延迟报文干扰后续连接的情况。这经常出现在你作为客户端对同一个服务端发起大量短连接的情况下，本地端口被快速申请，却慢速的释放，或者是作为客户端你使用了本地固定端口。

> Changing the 2MSL delay is usually a machine wide configuration change. You can instead attempt to work around TIME_WAIT at the socket level with the SO_REUSEADDR socket option. This allows a socket to be created whilst an existing socket with the same address and port already exists. The new socket essentially hijacks the old socket. You can use SO_REUSEADDR to allow sockets to be created whilst a socket with the same port is already in TIME_WAIT but this can also cause problems such as denial of service attacks or data theft. On Windows platforms another socket option, SO_EXCLUSIVEADDRUSE can help prevent some of the downsides of SO_REUSEADDR, see here, but in my opinion it's better to avoid these attempts at working around TIME_WAIT and instead design your system so that TIME_WAIT isn't a problem.

更改 2MSL 延时通常是操作系统级别的配置改动。你可以改为在 socket 层面使用 SO_REUSEADDR 选项针对 TIME_WAIT 问题做一些工作。这个选项允许你在相同的正在使用的地址和端口上创建 socket，新的 socket 本质上是劫持了老的socket。你可以使用此选项来创建新的 socket，即便它正处在 TIME_WAIT 状态，但是这样做有可能造成问题，例如拒绝服务攻击或者数据丢失。在 Windows 平台上，另一个 socket 选项 SO_EXCLUSIVEADDRUSE 可以帮助弥补 SO_REUSEADDR 的某些缺点。但我认为最好避免这些尝试来解决 TIME_WAIT 的问题，而应精心设计系统从而使 TIME_WAIT 不会成为问题。

> The TCP state transition diagrams above both show orderly connection termination. There's another way to terminate a TCP connection and that's by aborting the connection and sending an RST rather than a FIN. This is usually achieved by setting the SO_LINGER socket option to 0. This causes pending data to be discarded and the connection to be aborted with an RST rather than for the pending data to be transmitted and the connection closed cleanly with a FIN. It's important to realise that when a connection is aborted any data that might be in flow between the peers is discarded and the RST is delivered straight away; usually as an error which represents the fact that the "connection has been reset by the peer". The remote peer knows that the connection was aborted and neither peer enters TIME_WAIT.

上面的 TCP 转换图均展示的是有序连接终止。然而还有一种终止连接的方式，就是发送一个 RST 报文而不是 FIN 报文。一般是通过将 SO_LINGER 套接字选项设置为0来实现的。发送 RST 将导致待传输的数据（尚在发送缓冲区中）被丢弃，它不像发送 FIN 进行有序终止那样会等待数据被传送完成（一个 RST 报文会立即被送出，而 FIN 需要在缓冲区排队）。连接被终止时连接两端任何尚流中的数据都将被丢弃，认识到这一点非常重要，因为 RST 报文会立即被传输；应用通常会收到“connection has been reset by the peer”的错误。这样远端就会意识到连接已经终止了，而且不会有任何一方进入 TIME_WAIT 状态。

> Of course a new incarnation of a connection that has been aborted using RST could become a victim of the delayed segment problem that TIME_WAIT prevents, but the conditions required for this to become a problem are highly unlikely anyway, see above for more details. To prevent a connection that has been aborted from causing the delayed segment problem both peers would have to transition to TIME_WAIT as the connection closure could potentially be caused by an intermediary, such as a router. However, this doesn't happen and both ends of the connection are simply closed.

如此一来，没有了 TIME_WAIT 的保护，一个被 RST 终止的连接的新“化身”可能会成为延迟报文的受害者，但是无论如何，出现这种情况的可能性微乎其微，请参阅上文以了解更多详细信息。

> There are several things that you can do to avoid TIME_WAIT being a problem for you. Some of these assume that you have the ability to change the protocol that is spoken between your client and server but often, for custom server designs, you do.

当然，要避免 TIME_WAIT 造成的困扰，我们尚有可为。前提是你可以控制客户端和服务端之间的设计实现，对于自定义的服务端来说，通常都需要这样做。

> For a server that never establishes outbound connections of its own, apart from the resources and performance implication of maintaining connections in TIME_WAIT, you need not worry unduly.

对于一个不会建立出站连接的服务端来说，除了承受维护 TIME_WAIT 状态所带来的性能影响和资源占用之外，不需要对其有过多的担心。

> For a server that does establish outbound connections as well as accepting inbound connections then the golden rule is to always ensure that if a TIME_WAIT needs to occur that it ends up on the other peer and not the server. The best way to do this is to never initiate an active close from the server, no matter what the reason. If your peer times out, abort the connection with an RST rather than closing it. If your peer sends invalid data, abort the connection, etc. The idea being that if your server never initiates an active close it can never accumulate TIME_WAIT sockets and therefore will never suffer from the scalability problems that they cause. Although it's easy to see how you can abort connections when error situations occur what about normal connection termination? Ideally you should design into your protocol a way for the server to tell the client that it should disconnect, rather than simply having the server instigate an active close. So if the server needs to terminate a connection the server sends an application level "we're done" message which the client takes as a reason to close the connection. If the client fails to close the connection in a reasonable time then the server aborts the connection.

如果你的服务端既要建立出站连接又要建立入站连接，那么设计的黄金法则就是：如果 TIME_WAIT 注定要发生，那么就让它发生在对端。简单来说就是不管什么原因，永远不要在服务端主动关闭连接。如果对端超时，请使用 RST 中止连接，而不要关闭它。如果对端发送了无效的数据，使用 RST 终止连接等等。其中诀窍就是如果服务端永远不主动关闭连接，那么服务器上就不会积累 TIME_WAIT 的 socket，因此便不会遭受可伸缩性问题。我们看到在发生异常的情况下如果发送 RST 终止连接，那么正常情况下的终止又该如何避免呢？理想情况下，你应该在应用层设计一种协议方式，让服务端告知客户端可以断开连接，而不是由服务端主动发起关闭。所以，服务端如果想断开连接了，那么发送一个应用层的消息“我们分手吧”，客户端收到后再主动发起连接关闭，如果客户端在预定时间内没有成功关闭，那么服务端可以主动终止连接。

> On the client things are slightly more complicated, after all, someone has to initiate an active close to terminate a TCP connection cleanly, and if it's the client then that's where the TIME_WAIT will end up. However, having the TIME_WAIT end up on the client has several advantages. Firstly if, for some reason, the client ends up with connectivity issues due to the accumulation of sockets in TIME_WAIT it's just one client. Other clients will not be affected. Secondly, it's inefficient to rapidly open and close TCP connections to the same server so it makes sense beyond the issue of TIME_WAIT to try and maintain connections for longer periods of time rather than shorter periods of time. Don't design a protocol whereby a client connects to the server every minute and does so by opening a new connection. Instead use a persistent connection design and only reconnect when the connection fails, if intermediary routers refuse to keep the connection open without data flow then you could either implement an application level ping, use TCP keep alive or just accept that the router is resetting your connection; the good thing being that you're not accumulating TIME_WAIT sockets. If the work that you do on a connection is naturally short lived then consider some form of "connection pooling" design whereby the connection is kept open and reused. Finally, if you absolutely must open and close connections rapidly from a client to the same server then perhaps you could design an application level shutdown sequence that you can use and then follow this with an abortive close. Your client could send an "I'm done" message, your server could then send a "goodbye" message and the client could then abort the connection.

客户端这头就略有复杂，正所谓“我不入地狱，谁入地狱”，毕竟总要有人主动发起关闭使 TCP 正常结束，也总要有人去承受 TIME_WAIT。然而，将 TIME_WAIT 终止在客户端还具有几个好处呢！首先，如果出于某种原因，客户端由于 TIME_WAIT 中 socket 的累积而导致连接问题话，只是它一个客户端的问题，其它的客户端不会受其影响。其次，快速打开和关闭与同一服务器的 TCP 连接的效率很低，因此 TIME_WAIT 的维持维持一个更长的等待时间反而有比较有意义。不要设计那种每分钟都要通过新建连接来连接到服务端的客户端。相反，使用长连接来替代，只有在连接失败的时候才去重连，如果中间路由不愿意维护长时间的空闲连接，可以在应用层添加心跳机制，或者打开 TCP 的 keep alive 保活机制，再或者就主动接受连接被重置。这样做的好处就是你不会积累 TIME_WAIT 了，如果你做的工作就是短期任务，那么考虑使用连接池吧。如果你非要写那种快速打开关闭，分分钟干出一堆 TIME_WAIT 的客户端，或许你可以设计一个应用层的关闭机制，你在客户端发送“我干完了”，服务端接收之后返回一个“再见”，之后客户端便可以发送 RST 终止连接了（注意，虽然解决了 TIME_WAIT 和数据完整性问题，但是延时报文仍然可能成为问题，而 per-host PAWS 无法完全解决不同“化身”之间的报文延时）。

> TIME_WAIT exists for a reason and working around it by shortening the 2MSL period or allowing address reuse using SO_REUSEADDR are not always a good idea. If you're able to design your protocol with TIME_WAIT avoidance in mind then you can often avoid the problem entirely.

存在即合理，一味的通过 TCP 选项来重用地址或是盲目的减少 MSL 的值的做法都是不可取的。

念念不忘，必有回响，若你在设计协议时一直考虑 TIME_WAIT 的问题，那么你终将会完全避开。
