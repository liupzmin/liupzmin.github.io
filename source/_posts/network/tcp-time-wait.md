---
title: 好朋友 TIME_WAIT
date: 2020-02-26 15:18:20
tags: 
	- TIME_WAIT
	- tcp_tw_reuse
	- tcp_tw_recycle
	- PAWS
	- ISN
categories:
    - TCP/IP
    - computer theory
    - network
mathjax: true
---

**今天这篇文章我准备说一说 `TIME_WAIT` ，我相信很多人在工作中都或多或少遇到 TIME_WAIT 的情况，而且在面试中也经常被问到 `遇到大量的 TIME_WAIT 时应该怎么办？`这样的问题。 因为对 TIME_WAIT 这个状态印象并不是很深，所以被问到类似问题的时候也经常说不出所以然，因此，我花了很长的时间深入研究了一下 TCP 的相关问题，然后在此记录一下。**

先放一个 TCP Header 在这里。

![TCP 头格式](http://qiniu.liupzmin.com/tcp-header.jpg)

## 什么是 TIME_WAIT

`TIME_WAIT` 是 TCP 状态机中的一个，它出现在连接正常断开的时候。

![Figure 1. The TCP state transition diagram](http://qiniu.liupzmin.com/tcp-state.png)

从状态转换图中可以看出，`TIME_WAIT` 是断开连接时的最后一个状态，其上有个计时器表示连接在 `TIME_WAIT` 这个状态停留的时长为 `2MSL(Maximum Segment Lifetime)`，意为 2 个最大报文生存时间（ [RFC793](https://tools.ietf.org/html/rfc793#page-28) 定义了MSL为2分钟，然而在常见的实现中有 30s，1 分钟，2 分钟的情况；其中 Linux 设置成了 30s，并且是硬编码无法更改，除非重新编译内核；Windows 下默认是 1 分钟，且可以修改）。

大部分人对 `TIME_WAIT` 的第一印象并不好，但是 [Richard Stevens](https://zh.wikipedia.org/wiki/%E7%90%86%E6%9F%A5%E5%BE%B7%C2%B7%E5%8F%B2%E8%92%82%E6%96%87%E6%96%AF) 在他的 [《UNIX网络编程》](https://book.douban.com/subject/1500149/)一书中却说道：`TIME_WAIT 是我们的朋友，它是有助于我们的，不要试图避免这个状态，而是应该弄清楚它。`

这也正是我所希望的，让我们从 TCP 的四次挥手说起。

![Figure 2. TCP states corresponding to normal connection establishment and termination](http://qiniu.liupzmin.com/tcp-3estab-4term.png)

从上图我们可以看出 TCP 四次挥手的过程：

1. 客户端调用 close()，协议层发送 FIN 报文表示主动断开连接，而后进入 `FIN_WAIT_1` 状态。
2. 服务端收到客户端发送的 FIN ，返回一个 ACK 通知对端：我已知晓，并进入 `CLOSE_WAIT` 状态。
3. 客户端收到 ACK 后进入 `FIN_WAIT_2` 状态，等待服务端应用程序调用 close()操作。
4. 服务端处理完数据之后调用 close()，协议层发送 FIN 报文给客户端，等待 ACK，然后进入 `LAST_ACK` 状态。
5. 客户端收到 FIN 报文之后发送 ACK，并进入 `TIME_WAIT` 状态。
6. 服务端收到 ACK 之后进入 `CLOSED` 状态。
7. 客户端等待 `2*MSL` 时间后进入 `CLOSED` 状态。

通过上面的步骤我们可以得出以下几点：

1. 只有主动关闭的一方才会进入 `TIME_WAIT`，这既可以发生在客户端，也可以发生在服务端。
2. `TIME_WAIT` 会持续 `2*MSL` 的时间，之后进入`CLOSED` 状态。
3. 在连接没有进入 `CLOSED` 之前是无法被重用的。

## 几个概念

在继续深入讨论 `TIME_WAIT` 之前，我们先来明确几个术语概念：

- **TTL （Time-to-Live）**，意为生存期，它是 IP 头部的一个字段，用于设置一个数据报可经过的路由器的数量上限。发送方将它初始化为某个值（RFC 建议设为 64，但是 128 或 255 也不少见），每个路由器在转发数据报时将该值减 1。当这个字段达到 0 时，该数据报被丢弃，用于防止路由环路导致数据报在网络中永远循环。有意思的是最初 TTL 这个字段意思是指定 IP 数据报在网络上的最大生存秒数，但路由器总需要将这个值至少减 1。实际上，路由器在正常操作下通常不会持有数据报超过 1 秒钟，因此较早的规则早已被遗忘，这个字段在 IPv6 中根据实际用途已被重命名为跳数限制。
- **MSL（Maximum Segment Lifetime）**，意为最大报文生存时间，是 TCP 协议中的概念，指的是一个 TCP 的 Segment 在网络上生存的最大时间。很多书籍中都提到 MSL 是由 IP 层的 TTL 来保证的，但 TTL 和 MSL 不绝对相等，一般 MSL 会大于 TTL。我观 [RFC793](https://tools.ietf.org/html/rfc793) 中使用了 **assume** 一词（`Since we assume that segments will stay in the network no more than the Maximum Segment Lifetime`），所以我理解 TCP 并不会保证一个报文的最大生存时间，只是将 MSL 作为前提假设进行协议上的设计。
- **SN（Sequence Number）**，`序列号`是 TCP 一个头部字段，标识了 TCP 发送端到 TCP 接收端的数据流的一个字节，因为 TCP 是面向字节流的可靠协议，为了保证消息的顺序性和可靠性，TCP 为每个传输方向上的每个字节都赋予了一个编号，以便于传输成功后确认、丢失后重传以及在接收端保证不会乱序。SN 是一个 32 位的无符号数，因此在到达 $2^{32} -1$ 之后再循环回到 0。
- **ISN（Initial Sequence Number）**，`初始序列号` 是一个连接建立之前双方商议的初始的序列号，也就是传输的第一个字节的编号（通常是 SYN 位，因为 SYN 消耗一个序列号，相对的 ACK 不会消耗序列号，毕竟 SYN 和 FIN 是要保证可靠传输的）。很显然这个初始序列号不能是 0 或者 1，它必须是一个随机的数，并会随时间改变，这样每一个连接都拥有不同的`初始序列号`。[RFC793](https://tools.ietf.org/html/rfc793#page-27) 指出初始化序列号可被视为一个32位的计数器，该计数器的数值每 4 微秒加 1，这样循环一次需要 4.55 小时。此举的目的在于为一个连接的报文段安排序列号，以防止出现与其它连接的序列号重叠的情况，尤其是对于同一个连接的两个不同实例（也叫不同的化身）而言，新的序列号也不能出现重叠的情况。

**解释一下所谓的`同一个连接的不同实例`，我们知道标识一个连接需要 2 个 IP 和 2 个端口号，称为四元组。如果再加上协议类型就称为五元组，因为我们只讨论 TCP，所以也可以直接叫四元组。所谓的同一个连接的不同实例是指一个连接关闭后，又以相同的四元组打开了一个新的连接，通常称老的连接为前一个化身。**

试想一下，一个连接因为某种原因被关闭，紧接着又以相同的四元组被重新建立。如果这时旧连接的一个延时的报文又到达了，碰巧这个延时的报文段的序列号对新连接又是有效的，那么这个报文就会被视为有效的数据进入新连接的数据流中。TCP 的很多设计目标都是为了避免这种恼人的情况，更为复杂的是随着网路性能的提升，带宽的增加，这种问题甚至出现在同一个连接实例当中，因为 SN 这个 32 位的字段也是要回绕重用的，如果一个回绕的时间太短，小于一个 MSL 的时候，我们也要面对延时报文的问题。

 `TIME_WAIT` 的设计初衷就是为了解决 TCP 新旧连接延迟报文的。

## TIME_WAIT 的目的

[RFC1185](https://tools.ietf.org/html/rfc1185#page-12)、[RFC1323](https://tools.ietf.org/html/rfc1323#page-28)、[RFC7323](https://tools.ietf.org/html/rfc7323#page-35) 中阐明了 `TIME_WAIT` 存在的两个目的：

1. 可靠的实现 TCP 全双工连接的终止。
2. 允许老的重复的报文段在网络中消失。

第一个理由可以通过查看 `Figure 2` 四次挥手过程，并假设最终的 ACK 丢失来解释。这时服务端将重新发送它的最后的那个 FIN，因此客户端必须维护状态信息，以允许它重新发送最终的 ACK。要是客户端不维护状态信息，其所在的服务器将响应一个 RST，服务端将收到这个 RST 并将其解释为一个错误，这对于一个可靠的协议来说是不完美的终止方式。为了防止这种情况出现，客户端必须等待足够长的时间确保对端收到 ACK，如果对端没有收到 ACK，那么就会触发 TCP 重传机制，服务端会重新发送一个 FIN，这样一去一来刚好两个 MSL 的时间。但是你可能会说重新发送的 ACK 还是有可能丢失啊，没错，但 TCP 已经等待了那么长的时间了，已经算仁至义尽了。

为了理解第二个理由必须要很好的理解 `Squence Number` 和 `ISN`，也就是`序列号`和`初始序列号`，我们再次强调：**TCP 是面向字节流的协议。** 为了解决可靠传输和乱序问题，**TCP 为每一个传输的字节都编上了号**，TCP 头部的 squence number 表示的是该报文中携带数据的第一个字节的编号，下图可以形象的说明这一点：

![Figure 3. 文件数据划分成 TCP 报文段](http://qiniu.liupzmin.com/dividing-data-into-tcp-segments.png)

图中例子假设主机 A 上的一个进程想通过一条 TCP 连接向主机 B 上的一个进程发送一个数据流，这个数据流可能是一个文件。主机 A 中的 TCP 将隐式的对数据流中的每一个字节编号。假定该字节流是一个由 500000 字节的文件组成，其 MSS 为 1000 字节，数据流的首字节编号为 0。如图所示，TCP 将构建 500 个报文段，给第一个报文段分配序号 0，第二个报文段分配序号 1000，第三个报文段分配序号 2000，以此类推。每一个序号被填入到相应的 TCP 报文段头部的序列号字段中。

这个例子中我们假定数据流的首字节编号从 0 开始是不准确的，因为 SYN 消耗一个序列号，所以第一个字节总是 ISN 加上 1。前面讲到 ISN 是一个基于时间的计数器，它是一个随机值，每 4.55 小时回绕一次。相似地，Sequence Number 也是一个 32 位的循环使用字段。也就是说，大约每传送 4GB 的数据之后，序列号就会回到最初的值（也就是 ISN，因为当达到最大值时会再从 0 开始一直加到 ISN），如此循环往复。

有了上面的基本概念，我们现在就可以假设如下情况了。

一个由四元组标识的唯一连接，因某种原因被关闭。但随即以相同的四元组重新打开。那么这是同一个连接的新化身（incarnation），这个时候 TCP 必须要防止前一个连接的老的报文段因为网络延迟而重新到达，以致被误解为新连接的数据。如 `Figure 4` 所示：

![Figure 4. Due to a shortened TIME-WAIT state, a delayed TCP segment has been accepted in an unrelated connection.](http://qiniu.liupzmin.com/duplicate-segment.png)

图中该连接的前一个化身序列号为 3 的报文段超时重传了，之后连接关闭。假设没有经过 `TIME_WAIT`，或者 `TIME_WAIT` 的时间很短，在新的连接化身刚刚发送完序列号为 2 的报文段之后，前一个化身迷途的 `seq=3` 的报文段又抵达了。TCP 必须防止这种情况。

为了做到这一点，TCP 设计了 `TIME_WAIT` 。它会锁定这个四元组 `2*MSL` 的时间 ，这期间不允许以相同的四元组打开新连接，我们知道 MSL 是 TCP 报文段最大的网络生存时间，那么 `2*MSL` 足以让这个连接上的所有延迟的报文被丢弃，通过 `TIME_WAIT` 这个规则，我们就能保证每成功建立一个 TCP 连接时，来自该连接先前化身的老的重复的报文都已在网络中消逝了。

前一个连接的老的报文给当前的连接造成错乱的前提是其携带的序列号刚好在当前的连接有效，虽然这不太可能发生，但是因为 ISN 和 SN 两个字段值都是会 `wrap around（回绕）` 的，所以理论上是会出现这种情况的，[RFC1185](https://tools.ietf.org/html/rfc1185#page-15) 中有针对该问题详细的讨论描述，我们借用里面的图来说明这种情况。

为了应对老的重复报文，Tomlinson 提出了基于时间驱动的 ISN 方案，但这个方案只对短连接和传输速率慢的连接有效，如下图所示：

```text
        |- 2**32       ISN             ISN
        |              *               *
        |             *               *
        |            *               *
        |           *x              *
        |          o               *
    ^   |         *               *
    |   |        *  x            *
        |       * o             *
    S   |      *o              *
    e   |     o               *
    q   |    *               *
        |   *               *
    #   |  * x             *
        | *o              *
        |o_______________*____________
                         ^         Time -->
                       4.55hrs


     Figure 5.  Clock-Driven ISN  avoiding duplication on
                short-Lived, slow connections.
```

横轴为时间，纵轴为序列号或初始序列号（它们本质上是同一个东西，ISN 只是以某种方式选择的序列号初始值），`*` 代表 ISN，`o` 代表同一个短时连接的不同化身的轨迹，其序列号在 `x` 处终止。因为`慢`，所以每个化身序列号的增长曲线是低于 ISN 的增长曲线的。也就是说，此时没有 `TIME_WAIT` 也一样能排除老的重复报文段，因为每个化身的 ISN 均会保证大于前一个化身最后使用的序列号。只要一个报文段的序列号小于当前连接的初始序列号，TCP 自动就可以识别出来从而将其丢弃。但是，对于长连接或者一个快速的网络环境中，仅仅是 ISN 就不能保证这一点了：

```text
        |- 2**32       ISN               ISN
        |              *                 *
        |             *                 *
        |            *                 *
        |           *                 *
        |          *                 *
    ^   |         *                 *
    |   |        *                 *
        |       *                 *
    S   |      *                 *
    e   |     *                x*
    q   |    *           o     *
        |   *      o          *
    #   |  *o                *
        | *                 *
        |*_________________*____________
                           ^         Time -->
                          4.55hrs

        Figure 6.  Slow, Long-Lived Connection
```

如图 `Figure 6` ,一个长连接且慢速传输的情况，如果 ISN 回绕到与 seq 相同的值时，连接关闭又随即重新打开，那么正在传输中的老的连接的报文段就很可能会进入新连接的窗口中。

在一个高性能的快速的网络环境中，问题更显复杂。因为这时有两种情况使得老的报文段可能造成问题，先看第一种：

```text
        |- 2**32       ISN               ISN
        |              *                 *
        |       x   o *                 *
        |            *                 *
        |      o-->o*                 *
        |          *                 *
    ^   |     o   o                 *
    |   |        *                 *
        |    o  *                 *
    S   |      *                 *
    e   |   o *                 *
    q   |    *                 *
        |  o*                 *
    #   |  *                 *
        | o                 *
        |*_________________*____________
                           ^         Time -->
                          4.55hrs

     Figure 7.  Duplication on Fast Connection: Nc < 2**32 bytes
```

如图 `Figure 7` 所示是连接传输的字节小于 `4GB` 的情况，并在序列号 `x` 处连接关闭或者崩溃，紧接着一个新的化身被建立，而此时的 ISN 尚远远小于 `x` 的值。这样前一个化身的老的重复的报文段很轻易的就侵入到当前的连接。

```text
        |- 2**32       ISN               ISN
        |      o       *                 *
        |           x *                 *
        |            *                 *
        |     o     *                 *
        |          o                 *
    ^   |         *                 *
    |   |    o   *                 *
        |       * o               *
    S   |      *                *
    e   |   o *                 *
    q   |    *   o             *
        |   *                 *
    #   |  o                 *
        | *     o           *
        |*_________________*____________
                           ^         Time -->
                          4.55hrs
     Figure 8.  Duplication on Fast Connection: Nc > 2**32 bytes
```

图 `Figure 8` 所示是连接传输的字节大于 `4GB` 的情况，且序列号已经回绕（wrap around），原本已经分离的曲线，在序列号回绕到 `x` 处时又相交了。一旦窗口有重合，老的延迟报文段就会造成问题。

通过上面的图例，我们能得出一个结论：**只要新连接的 ISN 大于前一个化身的最终序列号，那么问题便会迎刃而解**。果不其然，Garlick, Rom, and Postel 在 [RFC1185](https://tools.ietf.org/html/rfc1185#page-19) 中提出了这样的想法：**由 TCP 维护每一个连接的最终序列号，当新的连接被打开的时候，保证 ISN 的值永远大于上一个化身的最终使用序列号**。

但是，貌似这个方案没有被最终采用，TCP 选择了 `ISN` 以及 `TIME_WAIT`。

## TIME_WAIT 造成的影响

`TIME_WAIT` 主要造成的影响有两个方面：

1. `TIME_WAIT` 会长时间占用（`2*MSL`）一个四元组连接，这可能导致后续相同元组的连接创建失败。
2. `2*MSL` 期间内核需要维持该 SOCKET 的数据结构，因此数量过多的话会消耗内存、并且增加内核遍历有关 hash table 的时间（更消耗CPU），从而导致性能问题。

因为处于 `TIME_WAIT` 等待状态的连接可能要花费 1 ~ 4 分钟才能进入 `CLOSED` 的状态并释放相应的四元组，所以在 `TIME_WAIT` 等待期间具有相同四元组的连接便不能建立。这是很多人对 `TIME_WAIT` 深恶而痛绝之的根源。

资源不能重用算是一个痛恨的理由，但第二个理由却值得细细推敲一番。多少 `TIME_WAIT` 算多呢？一千个？一万个？其实这个量级根本不用在乎，`TIME_WAIT` 所占用的内存很少，主要涉及到两个内核结构：

1. 内核里有保存所有连接的一个 hash table，这个 hash table 里面既包含 TIME_WAIT 状态的连接，也包含其他状态的连接。主要用于有新的数据到来的时候，从这个 hash table 里快速找到这条连接。不同的内核对这个 hash table 的大小设置不同，你可以通过 dmesg 命令去找到你的内核设置的大小：

   ```shell
   [root@VM_0_6_centos ~]# dmesg |grep "TCP established hash table"
   [    0.204805] TCP established hash table entries: 16384 (order: 5, 131072 bytes)
   ```

2. 还有一个 hash table 用来保存所有的 bound ports，主要用于可以快速的找到一个可用的端口或者随机端口：

   ```shell
   [root@VM_0_6_centos ~]# dmesg |grep "TCP bind hash table"
   [    0.207227] TCP bind hash table entries: 16384 (order: 6, 262144 bytes)
   ```

由于内核要保存这些数据，所以会占用一部分内存。**那么会消耗 CPU 么？**

**当然！**每次找到一个随机端口，还是需要遍历一遍  bound ports 的吧，这必然需要一些 CPU 时间。

TIME_WAIT 很多，既占内存又消耗 CPU，这也是为什么很多人，看到 TIME_WAIT 很多，就蠢蠢欲动的想去干掉他们。其实，如果你再进一步去研究，1 万条 TIME_WAIT 的连接，也就多消耗1M左右的内存，对现代的很多服务器，已经不算什么了。至于 CPU，能减少它当然更好，但是不至于因为 1 万多个 hash item 就担忧。

各种操作系统实现也提供了绕过 TIME_WAIT 的方法，如果你去 google 解决方法，网上十有八九会让你设置 **net.ipv4.tcp_tw_reuse** 和 **net.ipv4.tcp_tw_recycle** 这两个参数。如果你不深入了解 TIME_WAIT 的机制以及这两个参数的内在原理而贸然使用，那么结果可能会更加糟糕。

## 与 TIME_WAIT 有关的几个参数

- **net.ipv4.tcp_timestamps**
- **net.ipv4.tcp_tw_reuse**
- **net.ipv4.tcp_tw_recycle**

需要郑重说明的一点：**net.ipv4.tcp_tw_reuse** 和 **net.ipv4.tcp_tw_recycle** 这两个参数全部依赖于 **net.ipv4.tcp_timestamps**。

[RFC1323](https://tools.ietf.org/html/rfc1323#page-12) 引入了 `timestamp` 时间戳选项，主要为了精确计算 `RTT（ROUND-TRIP TIME）` 。时间戳的另一个功用是为了防止序列号回绕，我们先来介绍这个功能。

### PAWS

`PAWS（PROTECT AGAINST WRAPPED SEQUENCE NUMBERS）`是防序列号回绕的意思。

经过前面的介绍，我们对序列号回绕（wrap around）已经有了清晰的认识。但之前我们都是在讨论新旧连接之间怎么防止延迟的报文段，随着网络性能的提升，同一个连接内也面临着延迟报文的问题。

我们知道序列号是一个 32 位的无符号整型，用它来编码，我们最多编码 4GB 的数据，超过 4GB 后就需要将序列号回绕进行重用。这在以前网速慢的年代不会造成什么问题，但在一个速度足够快的网络中传输大量数据时，序列号的回绕时间就会变短。当 `wrap around time` 小于 MSL 的时候会发生什么呢？TCP 一切的设计都基于假设报文段最大的生存时间不会超过 MSL 的，如果序列号回绕的时间极短，我们就会再次面临之前延迟的报文段抵达后序列号依然有效的问题。这就让 TCP 协议难以自圆其说，试看下面的示例：

![Figure 9. TCP 通过时间戳选项消除相同序列号报文段的二义性](http://qiniu.liupzmin.com/seq-warp-around.png)

我们假设 TCP 的发送窗口是 1 GB，并且使用了时间戳选项，发送者会针对每个发送窗口分配时间戳数值，我们假设每个窗口时间加 1，然后使用这个连接传输一个 6GB 大小的数据流。

32 位的序列号字段在时刻 D 和 E 之间回绕。假设在时刻B有一个报文段丢失并被重传，又假设这个报文段在网络上绕了远路并在时刻 F 重新出现。而且这段时间小于 MSL ，也就是说回绕时间很短，比 MSL 还要小，之前的报文段还没有在网络中消逝，并且还及时绕了回来。

如果 TCP 无法识别这个“乔装”的报文段，那么数据完整性就会遭到破坏。使用时间戳选项能够有效的防止上述问题。接收者可以将时间戳看作一个 32 位的扩展序列号。丢失的报文段会在时刻 F 重新出现，由于它的时间戳为 2，小于最近的有效时间戳（5 或 6），因此防回绕序列号算法会将其丢弃。

需要注意的是，防回绕序列号算法并不要求在发送者与接受者之间有任何形式的时钟同步，接受者所需要的是保证时间戳数值单调增长。

### net.ipv4.tcp_tw_reuse

我们先看看这个参数的定义：

```text
tcp_tw_reuse - INTEGER
	Enable reuse of TIME-WAIT sockets for new connections when it is
	safe from protocol viewpoint.
	0 - disable
	1 - global enable
	2 - enable for loopback traffic only
	It should not be changed without advice/request of technical
	experts.
	Default: 2
```

其含义是对处于 `TIME_WAIT` 的 sockets 启动重用功能，使得新建连接时可以重用 `TIME_WAIT` 状态的四元组。定义中声称这是协议安全的，默认只对回环网卡启用。

这里面有一句很鲜明的话：**It should not be changed without advice/request of technical**。

**意思是如果没有专业指导就请不要修改它！** 我们回顾一下 TCP 的四次挥手，只有主动断开连接的一方才会进入 `TIME_WAIT` 状态，并且只有重新发起连接的时候才会重用具有相同四元组的处于 `TIME_WAIT` 状态的 socket。这也就是说： **tcp_tw_reuse 这个参数只对主动发起连接的客户端才奏效，只适用于 outbound 连接！**

那么使用 `tcp_tw_reuse` 跳过了 `TIME_WAIT` 阶段后，如何防止旧连接的延迟报文段呢？这就要依靠`timestamp` 时间戳选项提供的功能了。当新连接建立后，时间戳更新为最新的时间，当延迟的报文段到达后，其时间戳是小于当前连接的最近有效时间戳的。这个时候 TCP 就可以将这个报文段直接丢弃了。

再次强调，这个参数只适用于发起连接的一方，只影响出站连接，也就是作为客户端的情况。

### net.ipv4.tcp_tw_recycle

> 这个参数在 Linux 内核 4.12 中已经被废弃，但这里还是值得拿出来说一说

[RFC1323](https://tools.ietf.org/html/rfc1323#page-29) 中有下面一段话：

```text
An additional mechanism could be added to the TCP, a per-host
cache of the last timestamp received from any connection.
This value could then be used in the PAWS mechanism to reject
old duplicate segments from earlier incarnations of the
connection, if the timestamp clock can be guaranteed to have
ticked at least once since the old connection was open.  This
would require that the TIME-WAIT delay plus the RTT together
must be at least one tick of the sender's timestamp clock.
Such an extension is not part of the proposal of this RFC.
```

意思是有了时间戳选项，我们可以在 TCP 中用 cache 针对每个 host 记录任何连接最后的接收时间戳。这个值可以像 PAWS 机制那样工作，以防止前一个连接的延迟报文段，也就是用来解决 `TIME_WAIT` 要解决的问题。我们可以看出来这就是 `Garlick, Rom, and Postel` 在 [RFC1185](https://tools.ietf.org/html/rfc1185#page-19) 中提出**记录最终序列号**方案的一种变体。Linux 将其实现了，这就是 **net.ipv4.tcp_tw_recycle** 参数（网上都这么说，但我没有查到官方的说法）。

官方定义是：**Enable fast recycling TIME-WAIT sockets. Default value is 0. It should not be changed without advice/request of technical experts.**

这个配置同样依赖于双方对 timestamps 的支持，当开启了这个配置后，内核会快速的回收处于 TIME_WAIT 状态的socket 连接。多快？不再是 `2*MSL`，而是一个`RTO（retransmission timeout，数据包重传的timeout时间）`的时间，这个时间根据 `RTT` 动态计算出来，但是远小于`2*MSL`。

它对出站连接和入站连接均有影响，但主要是影响入站连接的重用。所以，之前网上的文章大部分都是使用它来解决服务端 `TIME_WAIT` 的情况，也就是服务端主动断开了连接。

`net.ipv4.tcp_tw_recycle` 这个选项特别激进。简单来说就是，Linux 会丢弃所有来自远端的 timestramp 时间戳小于上次记录的时间戳(由同一个远端发送的)的任何数据包。也就是说要使用该选项，则必须保证数据包的时间戳是单调递增的。

但是，如果对端是一个 NAT 网络的话（如：一个公司只用一个 IP 出公网）或是对端的 IP 被另一台重用了，这个事就复杂了。建连接的 SYN 可能就被直接丢掉了（你可能会看到 connection time out 的错误）。所以 NAT 就是 `net.ipv4.tcp_tw_recycle` 的死穴。

***其实我认为理论上在正常的 NAT 网络中也会出现此问题。假设 NAT 网络中一个客户端主动关闭了连接，在收到服务端的 FIN 报文后向对端发送了最后的 ACK，之后进入了 TIME_WAIT 状态。这个报文在 RTT/2 时间到达了服务端，服务端进入了 CLOSED 状态，此时 NAT 网络中的另一个客户机发起了连接，注意原客户机此时仍在 TIME_WAIT 状态，那么服务端的四元组已经可以重用了，但是如果此时发起新连接的客户机携带的时间戳没有保证递增，那么还是会出现 SYN 被丢弃的情况。（之所以有此疑问，是因为我不清楚中间路由是否会记录连接信息，如果中间路由能阻止连接的发起那么就不会出现这种情况，希望懂的朋友给以指正。）***



***更新：[RFC5382](https://tools.ietf.org/html/rfc5382#page-11">https://tools.ietf.org/html/rfc5382#page-11) 中针对 `RST` 和 `TIME_WAIT` 情况下的 NAT 行为并未给予明确规定，仅仅阐述了一下利弊。所以由实现者自行处理：***

```text
NAT behavior for handling RST packets, or connections in TIME_WAIT
state is left unspecified.  A NAT MAY hold state for a connection in
TIME_WAIT state to accommodate retransmissions of the last ACK.
However, since the TIME_WAIT state is commonly encountered by
internal endpoints properly closing the TCP connection, holding state
for a closed connection may limit the throughput of connections
through a NAT with limited resources.
```

***[RFC7857](https://tools.ietf.org/html/rfc7857#section-2.2) 中又对 `RST` 的情况做了修正，建议在删除连接信息之前保存　`4` 分钟的时间：***

```text
Concretely, when the NAT receives a TCP RST matching
an existing mapping, it MUST translate the packet according to the
NAT mapping entry.  Moreover, the NAT SHOULD wait for 4 minutes
before deleting the session and removing any state associated with
it if no packets are received during that 4-minute timeout.
```

***我并不清楚主流的路由器是如何处理这两种情况的，而且要研究其究竟超越了我目前的能力范围。因此，鉴于 RFC 中的相关描述，我暂时先从协议角度理解为上述担心的情况不会发生。***



但有一点我不是很明白，为什么 Linux 可以实现 tcp_tw_recycle，却不去实现**记录前一个化身的最终使用序列号**？这样完全可以避免 NAT 网络不同客户端时间戳无法保证单调递增的问题。不管这个连接是从 NAT 网络中哪台客户机发起的，始终为新连接选择一个大于前一个化身最终序列号的 ISN 就不会有任何问题。

### 其它参数

Linux 有个内核参数 `tcp_max_tw_buckets`控制并发的 TIME_WAIT 的数量，在我的 Linux 上的默认值是 `262144`，如果超限，那么，系统会把多的给 destory 掉，然后在日志里打一个警告（如：time wait bucket table overflow），这相当于跳过了 TIME_WAIT 直接将连接销毁。如果对端（被动关闭方）没有收到最后的 ACK，那么会重发 FIN 报文，因为主动端服务器已没有相关的连接信息，服务器收到 FIN 后会回一个 RST 报文将连接重置，到这里还没有什么大问题，只是 TCP 连接没有优雅的关闭而已。

问题在于这个连接跳过了 `TIME_WAIT` 阶段，如果一个相同四元组的连接随即被创建，而此时恰好前一个连接的迷途的报文又到达了，这会让当前的连接发生数据错乱。

这个值该设置为多少还是有待考量的。

## 总结

围绕着 TIME_WAIT 的概念已经说的很多了，但其实仍然难以回答大家最关心的问题：**如何去避免 TIME_WAIT ？**

其实我翻译的 [TIME_WAIT及其对协议和可伸缩客户端服务器系统的设计实现](http://liupzmin.com/2020/01/09/theory/time-wait-system-design/) 这篇文章中已经给出了完美的答案，我再简单总结一下：

1. 永远不要主动断开连接，让客户端去断开，由分布在各处的客户端去承受 TIME_WAIT。
2. 如果你非要在服务端断开连接，且你的服务不会有出站连接，那么服务器除了承受维护 TIME_WAIT 状态所带来的性能影响和资源占用之外，不需要对其有过多的担心。
3. 如果你的服务端既要建立出站连接又要建立入站连接，黄金法则依然是：如果 TIME_WAIT 注定要发生，那么就让它发生在对端。如果对端超时了，直接使用 RST 终止连接。但你服务端也要避免发起频繁的短时连接，尽量使用长连接或者连接池。
4. 如果你是客户端，尽量使用长连接和连接池，并发扬风格主动断开连接。
5. 如果你非要写那种快速打开关闭，分分钟干出一堆 TIME_WAIT 的客户端，或许你可以设计一个应用层的关闭机制，你在客户端发送“我干完了”，服务端接收之后返回一个“再见”，之后客户端便可以发送 RST 终止连接。注意，这虽然解决了 TIME_WAIT 和数据完整性问题，但是延时报文仍然可能成为问题，而 timestamps 选项无法完全解决不同“化身”之间的报文延时，比如 NAT 网络。如果你使用这种方式与服务端断开连接，那么在你的 NAT 网络中恰好另一台机器使用相同的客户端程序连接同一个服务端的时候就有可能遭遇时间戳无法保证单调递增的情况。
6. 作为客户端请合理的设置 `net.ipv4.ip_local_port_range`。

***参考文章：***

1. [RFC-0793](https://tools.ietf.org/html/rfc793l)
2. [RFC-1185](https://tools.ietf.org/html/rfc1185)
3. [RFC-1323](https://tools.ietf.org/html/rfc1323)
4. [RFC-7323](https://tools.ietf.org/html/rfc7323)
5. [TCP/IP详解 卷1：协议](https://book.douban.com/subject/1088054/)
6. [UNIX网络编程](https://book.douban.com/subject/1500149/)
7. [计算机网络自顶向下方法](https://book.douban.com/subject/30280001/)
8. [Coping with the TCP TIME-WAIT state on busy Linux servers](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux)
9. [TIME_WAIT and its design implications for protocols and scalable client server systems](http://www.serverframework.com/asynchronousevents/2011/01/time-wait-and-its-design-implications-for-protocols-and-scalable-servers.html)
10. [TIME_WAIT and “port reuse”](http://blog.davidvassallo.me/2010/07/13/time_wait-and-port-reuse/)
11. [TCP 的那些事儿（上）](https://coolshell.cn/articles/11564.html)
12. [TCP 的那些事儿（下）](https://coolshell.cn/articles/11609.html)
13. [你所不知道的TIME_WAIT和CLOSE_WAIT](https://blog.oldboyedu.com/tcp-wait/)
14. [那些与TIME_WAIT有关的参数](https://xiaochai.github.io/2019/03/16/tcp-sysctl-opts/)
15. [深入理解TCP的TIME-WAIT](http://blog.qiusuo.im/blog/2014/06/11/tcp-time-wait/)
16. [TCP timestamp](http://perthcharles.github.io/2015/08/27/timestamp-intro/)
17. [一个NAT问题引起的思考](http://perthcharles.github.io/2015/08/27/timestamp-NAT/)
18. [被抛弃的tcp_recycle](https://juejin.im/post/5c0642e65188251a82662912)