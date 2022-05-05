---
title: 一个学渣对于stack的顿悟（2）：穿越虚拟内存的迷雾
date: 2021-07-20 20:30:40
tags: 
    - os
    - stack
categories: 
    - CPU
    - computer theory
---

> 文中的示例基于 X86_64 体系架构，基于 Linux 内核 5.9.16 版本，汇编语言采用 AT&T 汇编

在上一篇文章中，我简要介绍了 CPU 执行指令的过程，以及 CPU 如何将一段内存用于 stack。这篇文章会将讨论的重点集中于**进程的内存布局**上面，因为要搞明白 stack 的运作原理，必然要知晓 stack 在内存中分配的位置。而谈到位置，必然绕不开虚拟内存。

图 2-1 描述了一个 Linux 进程的虚拟内存布局，类似的布局图已经充斥于互联网的每个角落，虽形态各异，但描述的内容大抵相同。这张图我引用自 [深入理解计算机系统.第三版](https://book.douban.com/subject/26912767/)，是一个比较详尽且权威的图（以后的文章中会经常出现此书，这是一本每个程序员必读的书）：

![图 2-1 Linux 进程的虚拟内存布局](https://qiniu.liupzmin.com/virtual-memory-of-a-linux-process.png)

这张图是进程的内存视角，从图中可知，进程虚拟内存的地址空间被分成两大部分：**用户空间**（**Process virtual memory**，也被称为**user space**）和 **内核空间**（**Kernel virtual memory**，也被称为**kernel space**）。每个进程拥有相同的`用户空间`视图（但彼此隔离），共享部分**内核空间**（如内核代码，图中的**Identical for each process**部分）。内核空间的另一部分是每个进程私有的， 如进程的内核堆栈、页表等。每个进程在内核空间都会有一块这样的私有区域（图中的**Different for each process**部分）。

这样的内存空间布局会在你的脑海里形成怎样的影像呢？

想象一下香蕉皮的样子🍌~，我心中虚拟内存地址空间的立体模型就是下面这个样子：

![图 2-2 Linux 进程地址空间形象图](https://qiniu.liupzmin.com/BananaPeel.png)

如果你尚未被**进程地址空间布局**折磨过，那图 2-1 或许会让你眼前一亮。你或许以为自己终于发现了事情的本质，但事实远没有那么简单。在陷入泥潭不能自拔之前，我们不妨先扪心自问一下：**程序或者说进程为什么需要这样的虚拟内存抽象呢？**

不妨让时光倒流，回到上个世纪70年代去，回到大型机落幕、小型机兴起的时代去，回到个人计算机开始萌芽的时代去，回到计算机先辈们为之奋斗的精彩纷呈的那个时代去......

## 1. 为什么需要虚拟内存

计算机的洪荒时代是没有内存抽象的，比如早期的大型机、小型机以及最早的个人计算机，它们都没有如今我们习以为常的虚拟内存抽象。没有抽象的直观表现就是：**程序直接访问物理内存**，比如我在上一篇文章中以 8086 演示汇编指令的示例，示例中对内存的访问就是直接操纵物理内存。

在**简单批处理时代**，直接访问物理内存并没有任何不妥。人们把二进制程序制成穿孔卡，有专门的人员将穿孔卡输入计算机运行。彼时计算机一次只能处理一个任务，每个任务运行时内存被其独占，各个程序也就相安无事。

但由于当时的计算机非常昂贵，人们很自然地想要减少这种浪费。在计算机的体系结构里，IO 设备和 CPU 是两种独立运行的部件，一个程序在进行 IO 的时候，CPU 往往无事可干。这当然是一种不能容忍的浪费，更何况任务的接续是由人来进行干预的。为了解决作业切换需要人员干预而造成的 CPU 浪费问题，人们改良了作业的运行机制，使得计算机在运行完一件作业之后可以自动的载入下一个作业，这就是真正的**批处理时代**。

然而，批处理并没有解决作业在进入 IO 等待时 CPU 的闲置问题，如果一个程序在等待 IO 完成，那么何不将下一个程序调入内存来执行呢？于是计算机便进入了**多道程序（multiprogramming）时代**。图 2-3 展示了三个作业同驻内存的情况：

![图 2-3 一个内存中有3个作业的多道程序系统](https://qiniu.liupzmin.com/mulitiprogram.png)

这种解决方案将内存分为几个部分，每个部分存放不同的作业，当一个作业在等待 I/O 完成时，另一个作业就可以使用 CPU。假如内存足够大，就可以容纳所需的全部任务，而 CPU 也就可以达到 100% 的利用率。

但在那个内存以 KB 计算，动辄几百美元的时代，这种假设也只能是天方夜谭。于是聪明的先驱们发明了**交换技术**，交换技术是解决内存超载的方法，即把一个进程完整的调入内存，使该进程运行一段时间，然后把它存回磁盘。空闲的进程主要存储在磁盘上，因此一个进程不运行就不会占用内存，图 2-4 展示了随时间流逝内存分配的情况：

![图 2-4 内存分配情况随着进程进出而变化](https://qiniu.liupzmin.com/mulitiprogramming.png)

开始时内存中只有进程 A，之后创建进程 B 和 C。当 D 需要运行的时候，由于内存已经不足以容纳 D，A 即被交换到磁盘，D 装入内存。之后 B 运行完毕被调出，A 又被调入，但 A 的位置较之前发生了改变，所以在其被调入内存时需要特殊的手段对其进行重定位，比如我们之前介绍的段寄存器就适用于这种场景，代码段和堆栈段的寄存器都需要修改。

多道程序增加了计算机的利用率，提高了作业处理效率。很快，人们对计算机的要求变得更多，比如程序员们厌倦了较长周期的开发-调试循环，**交互性**变得更加迫切，因为许多用户可能在同时使用机器，每个人都在等待他们执行的任务及时响应。计算机便进入了**分时共享**时代。

**分时共享系统**其实是多道程序系统的变体，在分时系统中假设有 20 个用户登录，其中有17个在思考或者在喝咖啡，那么 CPU 就可以分配给其它 3 个需要的作业来轮流执行。由于调试程序的用户常常只发出简短的命令，很少有长的费时的指令，所以计算机能够为许多用户提供快速的交互式的服务。对每个用户来讲，计算机就像是他们独占一样。而此时，计算机还有余力在后台运行批量作业。

第一个通用的分时系统是 **CCTS (Compatible Time Sharing System)**，它于 1962 年由 MIT（麻省理工学院）在一台改装过的 7094 大型机上开发出来。但直到第三代计算机广泛采用了必须的保护硬件之后，分时系统才逐渐流行开来。

系统中的所有进程共享 CPU 和主存，这本身就是对操作系统内核的巨大挑战。如果其中的某些进程需要太多的内存，那么有可能就无法运行；如果某个进程不小心写了另一个进程的内存，程序便会出现某些迷惑的 BUG；甚至操作系统内核也暴露在这种风险之下，也许在某个时候整个机器就会莫名其妙的停止运行。所以现代的操作系统都提供了一种对主存的抽象，叫做**虚拟内存**。

## 2. 虚拟内存

***虚拟内存是硬件异常、硬件地址翻译、主存、磁盘文件和内核软件的完美交互，它为每个进程提供了一个大的、一致的和私有的地址空间。***

这是  [深入理解计算机系统.第三版](https://book.douban.com/subject/26912767/) 中对于虚拟内存的定义，简明扼要！

虚拟内存提供了三个重要的能力：

1. 它将主存看成是一个存储在磁盘上的地址空间的高速缓存，在主存中仅保存活动的区域，并根据需要在磁盘和主存之间来回传送数据。
2. 它为每个进程提供了一致的地址空间，从而简化了内存管理。
3. 它保护了每个进程的地址空间不被其他程序破坏。

顾名思义，虚拟内存其地址是虚拟的，不是真实的物理地址。你在程序中打印出的地址以及在 Linux `/proc/{pid}/maps`中看的地址都是虚拟地址。图 2-5 给出了虚拟寻址的过程：

![图 2-5 虚拟寻址](https://qiniu.liupzmin.com/virtual-addressing.png)

在虚拟寻址系统中，传送给 CPU 的是虚拟地址，CPU 在访问内存之前，需要将这个虚拟地址转换为物理地址。事实上，这一转换动作是依靠另一种单独的硬件来完成的，将虚拟地址转换为物理地址的过程被称为**地址翻译（address translation）**。而负责地址翻译的专用硬件叫做内存管理单元，即图中的 **MMU(Memory Management Unit)** 。

当 MMU 完成地址翻译之后，真实的物理地址便被送入地址总线到达主存，随后相关的数据就会经数据总线读入 CPU 寄存器或由寄存器写入内存当中。当然，MMU 并不能凭空将虚拟地址翻译为物理地址，它需要一个叫做**页表**的内存结构的辅助。

虚拟内存系统将虚拟地址空间中固定大小的块分隔为虚拟页，将物理内存分割为物理页。虚拟内存所作的主要工作就是处理虚拟页和内存页之间的映射，虚拟页有3种状态：

1. 未分配：程序尚未用到（比如尚未执行到的代码片段所在页），VM 系统尚未创建此页。那么这种未分配的页没有任何数据相关联，更不会在物理内存中
2. 缓存的：程序已经使用的页，并且 VM 已分配且已经驻留在物理内存中
3. 未缓存的：程序用过，VM 已经分配，但此刻不在物理内存中（已被交换到文件或者交换到磁盘上的交换空间中）

![图 2-6 一个 VM 系统是如何使用主存作为缓存的](https://qiniu.liupzmin.com/vp-pp.png)

图 2-6 展示了一个有 8 个虚拟页的小虚拟内存。虚拟页 0 和 3 还没有被分配，1、4、6 已经被缓存在物理内存中，2、5、7 已经被分配了，但是当前并不在主存中，已经被交换到文件或者 swap 空间中去了。

虚拟系统必须用某种方式记录下这层映射关系以及缓存与否，否则地址翻译便不能工作。用于记录映射关系与缓存有效情况的内存结构叫做**页表（page table）**，它位于进程内核空间的进程私有区域，可参见图 2-1 进程空间布局视图。

本文并不想介绍**页表**的工作原理，这超出了文章的讨论范围。此处读者只需要明白页表的作用即可：**为地址翻译提供帮助。**页表相关内容可以参考任意一本介绍操作系统的书籍。

上述组织内存的方式被称为**分页**，事实上还有一种方式叫做分段，先将内存分段，再将这段内存分页，可以想见其复杂性。有必要指出的是第一个实现了分段加分页的操作系统是大名鼎鼎的 **MULTICS**。

 **MULTICS** 是有史以来最具影响力的操作系统之一，每一个 Unix 爱好者都曾闻其大名。它始于麻省理工学院的一个研究项目，其设计者着眼于建造一台满足整个波士顿地区所有用户计算需求的机器。除了 MIT 还有贝尔实验室以及通用电气公司参与。然而项目的难度远远超出了人们的预料，贝尔实验室退出了，通用电气公司也退出了，最后只有 MIT 坚持了下来。系统最后于 1969 年上线，在运行了 31 年后于 2000 年关闭。

几乎没有系统能像  **MULTICS**  一样没有修改地持续运行 31 年之久，尽管  **MULTICS**  在商业上失败了，但其许多原创的概念却散布于各种计算机文献。它的设计思想也经由 Ken Thompson 传承给了 Unix，一度催动了 C 语言的蓬勃发展，而 C 和 Unix 又反过来深深影响了后来的 Linux。

在 **MULTICS** 的这一波影响下，Intel 也未能幸免。 **MULTICS** 分段的思想在 X86 处理器上得到了继承，我们上篇提到的段寄存器便是受此影响的产物。换言之，8086 段设计的一部分思想来源于 **MULTICS**，并不完全是因为寄存器大小限制而催生的奇思妙想。虽然 x86-64 架构下仍有分段机制的某些痕迹，但正如上篇所言，这大多数情况下只是为了兼容。

那么，Intel 为什么要剔除它支持了近 30 年，且源自表现良好的 **MULTICS** 存储模型的变形体呢？

或许是因为 Unix 和 Windows 都不曾真正的使用过该模型！

## 3. 线程堆栈

### 3.1 stack 在哪儿 ？

经过上面的铺垫，终于可以聊一聊进程地址空间中的 stack 了。那么，stack 在进程地址空间中的什么位置呢？从图 2-1 来看，答案似乎显而易见：**stack 开始于用户空间的顶端，从高地址向低地址增长**。而作为动态增长的内存区域，**heap 排在代码段、数据段和 bss 段之上，由低地址向高地址增长，与 stack 遥相呼应，相对增长。**为了方便对照，此处再贴一次进程的地址空间布局，如图 2-7 ：

![图 2-7 Linux 进程的虚拟内存布局](https://qiniu.liupzmin.com/virtual-memory-of-a-linux-process.png)

x86_64 架构下 Linux 平台二进制可执行文件的代码段总是从地址的 `0x40000` 处开始，这是在链接阶段就决定的。但是如果你在稍微新一点的 Linux 上去做测试的话，代码段的起始地址很大程度上并不是从 `0x40000` 处开始，它似乎是一个随机的值。比如，我用下面一个简单的 C 程序来观察其地址空间的分布：

```c
#include <stdio.h>

int main(int argc, char **argv) {
	printf("Hello World\n");
	getchar();
}
```

使用`gcc demo.c`来编译为`a.out`，并运行。之后获取其进程号，查看`/proc/{pid}/maps`中的内容来观察地址空间的映射情况，如 表 2-1 ：

<pre>~/play/c ➭ cat /proc/2231/maps 
<span style="color:red">56487f376000-56487f377000 r--p 00000000 103:02 7093179                   /home/richard/play/c/a.out</span>
56487f377000-56487f378000 r-xp 00001000 103:02 7093179                   /home/richard/play/c/a.out
56487f378000-56487f379000 r--p 00002000 103:02 7093179                   /home/richard/play/c/a.out
56487f379000-56487f37a000 r--p 00002000 103:02 7093179                   /home/richard/play/c/a.out
56487f37a000-56487f37b000 rw-p 00003000 103:02 7093179                   /home/richard/play/c/a.out
56487fbc6000-56487fbe7000 rw-p 00000000 00:00 0                          [heap]
7f593eb72000-7f593eb74000 rw-p 00000000 00:00 0 
7f593eb74000-7f593eb9a000 r--p 00000000 103:02 4859211                   /usr/lib/libc-2.33.so
7f593eb9a000-7f593ece5000 r-xp 00026000 103:02 4859211                   /usr/lib/libc-2.33.so
7f593ece5000-7f593ed31000 r--p 00171000 103:02 4859211                   /usr/lib/libc-2.33.so
7f593ed31000-7f593ed34000 r--p 001bc000 103:02 4859211                   /usr/lib/libc-2.33.so
7f593ed34000-7f593ed37000 rw-p 001bf000 103:02 4859211                   /usr/lib/libc-2.33.so
7f593ed37000-7f593ed42000 rw-p 00000000 00:00 0 
7f593ed6f000-7f593ed70000 r--p 00000000 103:02 4859198                   /usr/lib/ld-2.33.so
7f593ed70000-7f593ed94000 r-xp 00001000 103:02 4859198                   /usr/lib/ld-2.33.so
7f593ed94000-7f593ed9d000 r--p 00025000 103:02 4859198                   /usr/lib/ld-2.33.so
7f593ed9d000-7f593ed9f000 r--p 0002d000 103:02 4859198                   /usr/lib/ld-2.33.so
7f593ed9f000-7f593eda1000 rw-p 0002f000 103:02 4859198                   /usr/lib/ld-2.33.so
<span style="color:red">7ffcf3aea000-7ffcf3b0b000 rw-p 00000000 00:00 0                          [stack]</span>
7ffcf3ba2000-7ffcf3ba6000 r--p 00000000 00:00 0                          [vvar]
7ffcf3ba6000-7ffcf3ba8000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]</pre>

<p align="center">表2-1 进程的内存布局映射</p>

可见，其代码段的起始地址为`56487f376000`，stack 段的其实地址为`7ffcf3b0b000`，如果多观察几次，每次的地址都不相同。

这其实归根于 Linux 在 2.6.12 之后加入的**地址空间布局随机化（Address space layout randomization，缩写ASLR）**，ASLR 的目的是为了安全，要禁用它，只需执行如下命令：

```shell
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

并且在使用`gcc`编译时加入`-no-pie`选项，然后再观察其内存布局，这一次我们直接观察proc中的maps：

<pre>~/play/c ➭ cat /proc/5206/maps
<span style="color:red">00400000-00401000 r--p 00000000 103:02 7095689                           /home/richard/play/c/a.out</span>
00401000-00402000 r-xp 00001000 103:02 7095689                           /home/richard/play/c/a.out
00402000-00403000 r--p 00002000 103:02 7095689                           /home/richard/play/c/a.out
00403000-00404000 r--p 00002000 103:02 7095689                           /home/richard/play/c/a.out
00404000-00405000 rw-p 00003000 103:02 7095689                           /home/richard/play/c/a.out
00405000-00426000 rw-p 00000000 00:00 0                                  [heap]
7ffff7dca000-7ffff7dcc000 rw-p 00000000 00:00 0 
7ffff7dcc000-7ffff7df2000 r--p 00000000 103:02 4859211                   /usr/lib/libc-2.33.so
7ffff7df2000-7ffff7f3d000 r-xp 00026000 103:02 4859211                   /usr/lib/libc-2.33.so
7ffff7f3d000-7ffff7f89000 r--p 00171000 103:02 4859211                   /usr/lib/libc-2.33.so
7ffff7f89000-7ffff7f8c000 r--p 001bc000 103:02 4859211                   /usr/lib/libc-2.33.so
7ffff7f8c000-7ffff7f8f000 rw-p 001bf000 103:02 4859211                   /usr/lib/libc-2.33.so
7ffff7f8f000-7ffff7f9a000 rw-p 00000000 00:00 0 
7ffff7fc7000-7ffff7fcb000 r--p 00000000 00:00 0                          [vvar]
7ffff7fcb000-7ffff7fcd000 r-xp 00000000 00:00 0                          [vdso]
7ffff7fcd000-7ffff7fce000 r--p 00000000 103:02 4859198                   /usr/lib/ld-2.33.so
7ffff7fce000-7ffff7ff2000 r-xp 00001000 103:02 4859198                   /usr/lib/ld-2.33.so
7ffff7ff2000-7ffff7ffb000 r--p 00025000 103:02 4859198                   /usr/lib/ld-2.33.so
7ffff7ffb000-7ffff7ffd000 r--p 0002d000 103:02 4859198                   /usr/lib/ld-2.33.so
7ffff7ffd000-7ffff7fff000 rw-p 0002f000 103:02 4859198                   /usr/lib/ld-2.33.so
<span style="color:red">7ffffffde000-7ffffffff000 rw-p 00000000 00:00 0                          [stack]</span>
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]</pre>

<p align="center">表2-2 无地址随机的进程内存布局映射</p>

可见，代码段的起始位置已经是固定的`00400000`，stack 段的起始地址为固定的`7ffffffff000`了。

值得注意的是：在不开启地址随机的情况下，stack 的起始地址位于**用户空间最高地址 - 4k** 处，即0x7ffffffff000`。用户空间的最高地址是`00007fffffffffff，即用户空间有 128TB 大小，Linux 文档 [mm](https://www.kernel.org/doc/Documentation/x86/x86_64/mm.txt) 描述了x86_64 架构下的内存空间布局，如表 2-3：

```
====================================================
Complete virtual memory map with 4-level page tables
====================================================

Notes:

 - Negative addresses such as "-23 TB" are absolute addresses in bytes, counted down
   from the top of the 64-bit address space. It's easier to understand the layout
   when seen both in absolute addresses and in distance-from-top notation.

   For example 0xffffe90000000000 == -23 TB, it's 23 TB lower than the top of the
   64-bit address space (ffffffffffffffff).

   Note that as we get closer to the top of the address space, the notation changes
   from TB to GB and then MB/KB.

 - "16M TB" might look weird at first sight, but it's an easier to visualize size
   notation than "16 EB", which few will recognize at first sight as 16 exabytes.
   It also shows it nicely how incredibly large 64-bit address space is.

========================================================================================================================
    Start addr    |   Offset   |     End addr     |  Size   | VM area description
========================================================================================================================
                  |            |                  |         |
 0000000000000000 |    0       | 00007fffffffffff |  128 TB | user-space virtual memory, different per mm
__________________|____________|__________________|_________|___________________________________________________________
                  |            |                  |         |
 0000800000000000 | +128    TB | ffff7fffffffffff | ~16M TB | ... huge, almost 64 bits wide hole of non-canonical
                  |            |                  |         |     virtual memory addresses up to the -128 TB
                  |            |                  |         |     starting offset of kernel mappings.
__________________|____________|__________________|_________|___________________________________________________________
                                                            |
                                                            | Kernel-space virtual memory, shared between all processes:
____________________________________________________________|___________________________________________________________
                  |            |                  |         |
 ffff800000000000 | -128    TB | ffff87ffffffffff |    8 TB | ... guard hole, also reserved for hypervisor
 ffff880000000000 | -120    TB | ffff887fffffffff |  0.5 TB | LDT remap for PTI
 ffff888000000000 | -119.5  TB | ffffc87fffffffff |   64 TB | direct mapping of all physical memory (page_offset_base)
 ffffc88000000000 |  -55.5  TB | ffffc8ffffffffff |  0.5 TB | ... unused hole
 ffffc90000000000 |  -55    TB | ffffe8ffffffffff |   32 TB | vmalloc/ioremap space (vmalloc_base)
 ffffe90000000000 |  -23    TB | ffffe9ffffffffff |    1 TB | ... unused hole
 ffffea0000000000 |  -22    TB | ffffeaffffffffff |    1 TB | virtual memory map (vmemmap_base)
 ffffeb0000000000 |  -21    TB | ffffebffffffffff |    1 TB | ... unused hole
 ffffec0000000000 |  -20    TB | fffffbffffffffff |   16 TB | KASAN shadow memory
__________________|____________|__________________|_________|____________________________________________________________
                                                            |
                                                            | Identical layout to the 56-bit one from here on:
____________________________________________________________|____________________________________________________________
                  |            |                  |         |
 fffffc0000000000 |   -4    TB | fffffdffffffffff |    2 TB | ... unused hole
                  |            |                  |         | vaddr_end for KASLR
 fffffe0000000000 |   -2    TB | fffffe7fffffffff |  0.5 TB | cpu_entry_area mapping
 fffffe8000000000 |   -1.5  TB | fffffeffffffffff |  0.5 TB | ... unused hole
 ffffff0000000000 |   -1    TB | ffffff7fffffffff |  0.5 TB | %esp fixup stacks
 ffffff8000000000 | -512    GB | ffffffeeffffffff |  444 GB | ... unused hole
 ffffffef00000000 |  -68    GB | fffffffeffffffff |   64 GB | EFI region mapping space
 ffffffff00000000 |   -4    GB | ffffffff7fffffff |    2 GB | ... unused hole
 ffffffff80000000 |   -2    GB | ffffffff9fffffff |  512 MB | kernel text mapping, mapped to physical address 0
 ffffffff80000000 |-2048    MB |                  |         |
 ffffffffa0000000 |-1536    MB | fffffffffeffffff | 1520 MB | module mapping space
 ffffffffff000000 |  -16    MB |                  |         |
    FIXADDR_START | ~-11    MB | ffffffffff5fffff | ~0.5 MB | kernel-internal fixmap range, variable size and offset
 ffffffffff600000 |  -10    MB | ffffffffff600fff |    4 kB | legacy vsyscall ABI
 ffffffffffe00000 |   -2    MB | ffffffffffffffff |    2 MB | ... unused hole
__________________|____________|__________________|_________|___________________________________________________________
```

<p align="center">表2-3 x86_64 下 Linux 48 bit 地址空间</p>

可以简单的用图 2-8 来表示空间的跨度，其中最高位的 128 TB 是留给内核空间的，低 128 TB 则属于用户空间，stack 起始地址紧随用户空间顶部。由于 `ASLR`的原因，其真实的地址会随机进行偏移。

![mm](https://qiniu.liupzmin.com/mm.jpg)

那么，**stack 会增长到多大呢？**这个问题似乎跟体系结构和操作系统有很大的关系，各类书籍不仅没有定论，甚至含糊其词，更没有一个可以让你一览无遗的列表清单。但我们似乎可以根据 Linux 来窥视一二。

从图 2-7 中可以看到，`stack` 和 `heap`之间还夹着一块**内存映射区域(mmap)**，且 Linux 下可以使用 **ulimit -s size** 来设置 stack 的大小。问题开始变得有些复杂，因为 mmap 区域的起点要根据 stack 的大小来进行计算，但 stack 的大小似乎可以任意指定，甚至可以设置为无限。

[深入 Linux 内核架构](https://book.douban.com/subject/4843567/) 在其**第 4 章：进程虚拟内存** 的 246 页指出：**可以根据栈的最大长度，来计算栈最低的可能位置，以用作 mmap 区域的起始点。但内核会确保 stack 至少跨越 128 MiB 的空间。另外，如果指定的栈界限非常巨大，那么内核会保证至少有一小部分地址空间不会被占据。**

书中的论断于当今的 Linux 内核是否仍适用，我目前也无从证明，但这已无关紧要。**进程的虚拟地址空间布局**也经过多次变迁，我们如今讨论的默认空间布局想来也不会一成不变。但 Linux 内核志在为用户呈现虚拟内存抽象的思想已可见一斑。可以肯定的是，它一定在繁芜的细节上做了大量细致的工作，才有了用户空间代码使用内存时的举重若轻。

### 3.2 stack 能长到多大 ？

要验证 stack 能长多大，说来也很容易，下面这段 C 代码就可以探测到 stack 的边界：

```c
#include <alloca.h>
#include <stdio.h>
int main (void)
{
  long a = 0;
  for (;;) {
	 void *y;
	 y = alloca(128);
	 a += 128;
   printf ("\nStack Size = %ld", a);
  }
}
```

代码在死循环中不断地从 stack 上申请`128`字节的空间并打印出当前已申请空间的大小，编译运行之，程序会在栈溢出时终止。令人意外的是，收到的错误并不是意料中的`stack overflow`，而是`segmentation fault (core dumped)`。

其实，**Stack overflow is [a] cause, segmentation fault is the result.**，详见[What is the difference between a segmentation fault and a stack overflow?](https://stackoverflow.com/questions/2685413/what-is-the-difference-between-a-segmentation-fault-and-a-stack-overflow)

这段代码探测到的 stack 大小在我的 Linux 内核 5.9.16 版本上是受 **ulimit** 控制的。那么，如果不对其设限，是否会碰撞到 mmap 区域，甚至 heap 区域呢？

执行`ulimit -s unlimited`之后再来运行程序，会发现程序使用的内存一路飙升，用完了 16 GB的内存后，又填满了 16GB 的 swap 空间，最后 OOM 被内核杀死。

是的，回忆一下图 2-8 ，用户空间有 128 TB 的大小呢，我的弱机并不配去探测极限🤣！

### 3.3 其他线程的 stack 呢 ？

似乎有些东西被我们遗忘了，一直以来我们都是在测试主线程的 stack，那其它的线程呢？

图 2-8 表示的内存布局看起来非常完美，`stack` 和 `heap` 相对增长，各自都有充裕的空间可以使用，何况大多数情况下 `stack` 并不会增长太大。因此，**48 bit**的寻址空间即便在中间又加入了 `mmap` 区域的情况下依旧可以处之泰然。

我们还没有介绍 `mmap` 区域，此处只需要知道 mmap 区域用于`私有文件映射`、`私有匿名映射`、`共享文件映射`、`共享匿名映射`即可。需要指出的是，在默认的虚拟内存地址空间布局下`mmap` 区域是和`heap`相对增长的，如图 2-9 所示：

![图2-9 mmap 区域自顶向下扩展](https://qiniu.liupzmin.com/mmap-region.png)

同时，从表 2-1 和 2-2 可以推出：stack 和 mmap 之间分别有 654 GiB 和 128 MiB，mmap 和 heap 之间占据了用户空间的绝大部分。如果我们不限制 stack 的大小，再来看一看布局情况。先执行`ulimit -s unlimited`，再来执行下面的代码：

```c
#include <stdio.h>

int main(int argc, char **argv) {
	printf("Hello World\n");
	getchar();
}
```

内存布局见表 2-4：

<pre>~/play/c ➭ cat /proc/6612/maps
00400000-00401000 r--p 00000000 103:02 7096036                           /home/richard/play/c/getchar
00401000-00402000 r-xp 00001000 103:02 7096036                           /home/richard/play/c/getchar
00402000-00403000 r--p 00002000 103:02 7096036                           /home/richard/play/c/getchar
00403000-00404000 r--p 00002000 103:02 7096036                           /home/richard/play/c/getchar
00404000-00405000 rw-p 00003000 103:02 7096036                           /home/richard/play/c/getchar
<span style="color:green">00405000-00426000 rw-p 00000000 00:00 0                                  [heap]</span>
155555321000-155555323000 rw-p 00000000 00:00 0 
155555323000-155555349000 r--p 00000000 103:02 4859211                   /usr/lib/libc-2.33.so
155555349000-155555494000 r-xp 00026000 103:02 4859211                   /usr/lib/libc-2.33.so
155555494000-1555554e0000 r--p 00171000 103:02 4859211                   /usr/lib/libc-2.33.so
1555554e0000-1555554e3000 r--p 001bc000 103:02 4859211                   /usr/lib/libc-2.33.so
1555554e3000-1555554e6000 rw-p 001bf000 103:02 4859211                   /usr/lib/libc-2.33.so
1555554e6000-1555554f1000 rw-p 00000000 00:00 0 
15555551e000-155555522000 r--p 00000000 00:00 0                          [vvar]
155555522000-155555524000 r-xp 00000000 00:00 0                          [vdso]
155555524000-155555525000 r--p 00000000 103:02 4859198                   /usr/lib/ld-2.33.so
155555525000-155555549000 r-xp 00001000 103:02 4859198                   /usr/lib/ld-2.33.so
155555549000-155555552000 r--p 00025000 103:02 4859198                   /usr/lib/ld-2.33.so
155555552000-155555554000 r--p 0002d000 103:02 4859198                   /usr/lib/ld-2.33.so
<span style="color:orange">155555554000-155555556000 rw-p 0002f000 103:02 4859198                   /usr/lib/ld-2.33.so</span>
<span style="color:red">7ffffffde000-7ffffffff000 rw-p 00000000 00:00 0                          [stack]</span>
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]</pre>

<p align="center">表2-4 进程的地址映射表</p>

可见，`stack` 和 `mmap` 之间有 106TiB 之大，`heap` 和 `mmap`之间仅剩 21TiB 的大小。

作为一个未曾充分使用过 C 语言的人，我看遍了所有的虚拟内存地址空间布局图之后，脑海中对于 `stack` 和 `heap` 的位置早已烂熟于胸。这也自然而然让我产生了思维定式：我以为线程所有的 stack 都是在这些布局图所示的 stack 位置上分配的。

然而事实并非如此，这些图的产生也有其历史原因。依我推测，它们大多是成图于线程概念出现之前，而线程出现之后，其内存布局并未在图中充分的体现，让我们来看[操作系统导论](https://book.douban.com/subject/33463930/)中的一句话：

***不是地址空间中只有一个栈，而是每个线程都有一个栈 ...... 你可能注意到，多个栈也破坏了地址空间的美感。以前堆和栈可以互不影响地增长，直到空间耗尽。多个栈就没有这么简单了。幸运的是，通常栈不会很大。***

细细体悟，此处**破坏了空间布局美感**背后的深意：即**后续的线程 stack 可以在很多地方分配，包括 `heap` 和 `mmap` 区域。**

除了[操作系统导论](https://book.douban.com/subject/33463930/)中这蜻蜓点水的一句之外，就没有其它的权威资料来说明这一点了么？

还真有，而且还被我找到了😉。

[TLPI](https://book.douban.com/subject/25809330/) 第 29.1 节展示了同时执行 4 个线程的进程，美中不足的是这幅图是基于 32 位地址空间的，如图 2-10 所示：

![图 2-10：同时执行 4 个线程的进程](https://qiniu.liupzmin.com/multiple-threads-stack.png)

主线程的 stack 位于我们熟知的内存布局的 stack 区域，而其余 3 个线程的栈位于 mmap 区域。事实上，**使用 `Pthread`线程库来创建线程的时候，对于线程 stack 的分配就是使用的 `mmap` 系统调用，其位置也必然位于 mmap 区域内。**

然而，Pthread 创建线程使用的是 `clone` 系统调用，而 clone 系统调用需要使用者自行传入一个内存区域以用作该线程的 stack。我们可以看一下文档中使用 clone 创建子进程的例子：

```c
 /* Allocate memory to be used for the stack of the child. */

 stack = mmap(NULL, STACK_SIZE, PROT_READ | PROT_WRITE,
              MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);
 if (stack == MAP_FAILED)
    errExit("mmap");

 stackTop = stack + STACK_SIZE;  /* Assume stack grows downward */

 /* Create child that has its own UTS namespace;
    child commences execution in childFunc(). */

 pid = clone(childFunc, stackTop, CLONE_NEWUTS | SIGCHLD, argv[1]);
 if (pid == -1)
    errExit("clone");
 printf("clone() returned %jd\n", (intmax_t) pid);
```

可见子进程的 stack 并没有使用传统意义上主线程使用的 stack 区域，而是使用`mmap` 在内存映射区域开辟了一块内存用于 stack。既然是自行传入的，那是不是就说明可以把 heap 中的一块内存当作线程的 stack 来用呢？恰好 Pthread 也提供自行设置 stack 的接口，那么，不妨来试试吧：

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

void* start_thread(){
 long a = 0;
 for (;;){
  void *y;
	y = alloca(1024);
	a += 1024;
  printf ("\nStack Size = %ld", a);
  sleep(1);
 }
}

int main(int argc, char **argv) {
  printf("Hello World\n");
  pthread_t thing1, pattr;
  pthread_attr_t tattr;
  void *stackbase;
  void *ptr;

  // 在 heap 总申请 100KiB 的内存
  long size = 100 * 1024;
  
  for (int i=0;i<5;i++) {
    ptr = (void *)malloc(size);
  }

  stackbase = (void *) malloc(size);
  printf("malloc address: %p\n",stackbase);

  // const char *message = "ThingHive";
	
  pthread_attr_init(&tattr);
  pthread_attr_setstack(&tattr, stackbase, size);
  // 自定义 stack 线程
  pthread_create(&pattr, &tattr, start_thread, NULL);
  // 默认线程
  pthread_create(&thing1, NULL, start_thread, NULL);
  
  pthread_join(pattr,NULL);
  pthread_join(thing1, NULL);
	// getchar();
  return 0;
}
```

上述代码除主线程外，额外创建了 2 个线程，一个 `pattr` 使用自定义 stack，另一个`thing1`使用默认选项。需要注意的是，代码使用`malloc`在 heap 上开辟一块内存用于线程的 stack，但这块内存不能大于 128KiB，否则 `malloc`会使用`mmap`来申请内存，详见[TLPI](https://book.douban.com/subject/25809330/)  49.7 节。

线程的执行体会每隔 1 秒钟在 stack 上分配 1KiB 的内存，便于我们观察 stack 的位置。编译并运行程序，这次使用`pmap -X {pid}`来观察区域的大小，如表 2-5 所示：

<pre>~/play/c ➭ pmap -X 9692
9692:   ./a.out
         Address Perm   Offset Device   Inode  Size  Rss Pss Referenced Anonymous  Mapping
        00400000 r--p 00000000 103:02 7093179     4    4   4          4         0  a.out
        00401000 r-xp 00001000 103:02 7093179     4    4   4          4         0  a.out
        00402000 r--p 00002000 103:02 7093179     4    4   4          4         0  a.out
        00403000 r--p 00002000 103:02 7093179     4    4   4          4         4  a.out
        00404000 rw-p 00003000 103:02 7093179     4    4   4          4         4  a.out
        <span style="color:orange">00405000 rw-p 00000000  00:00       0   732   <span style="color:red">92  92</span>         92        92  [heap]</span>
    7ffff75a7000 ---p 00000000  00:00       0     4    0   0          0         0  
    <span style="color:orange">7ffff75a8000 rw-p 00000000  00:00       0  8204   <span style="color:red">76  76</span>         76        76  </span>
    7ffff7dab000 r--p 00000000 103:02 4859211   152  148   1        148         0  libc-2.33.so
    7ffff7dd1000 r-xp 00026000 103:02 4859211  1324  876   6        876         0  libc-2.33.so
    7ffff7f1c000 r--p 00171000 103:02 4859211   304   64   0         64         0  libc-2.33.so
    7ffff7f68000 r--p 001bc000 103:02 4859211    12   12  12         12        12  libc-2.33.so
    7ffff7f6b000 rw-p 001bf000 103:02 4859211    12   12  12         12        12  libc-2.33.so
    7ffff7f6e000 rw-p 00000000  00:00       0    36   12  12         12        12  
    7ffff7f77000 r--p 00000000 103:02 4865362    28   28   0         28         0  libpthread-2.33.so
    7ffff7f7e000 r-xp 00007000 103:02 4865362    60   60   0         60         0  libpthread-2.33.so
    7ffff7f8d000 r--p 00016000 103:02 4865362    16   16   0         16         0  libpthread-2.33.so
    7ffff7f91000 ---p 0001a000 103:02 4865362     4    0   0          0         0  libpthread-2.33.so
    7ffff7f92000 r--p 0001a000 103:02 4865362     4    4   4          4         4  libpthread-2.33.so
    7ffff7f93000 rw-p 0001b000 103:02 4865362     4    4   4          4         4  libpthread-2.33.so
    7ffff7f94000 rw-p 00000000  00:00       0    24   12  12         12        12  
    7ffff7fc7000 r--p 00000000  00:00       0    16    0   0          0         0  [vvar]
    7ffff7fcb000 r-xp 00000000  00:00       0     8    4   0          4         0  [vdso]
    7ffff7fcd000 r--p 00000000 103:02 4859198     4    4   0          4         0  ld-2.33.so
    7ffff7fce000 r-xp 00001000 103:02 4859198   144  144   0        144         0  ld-2.33.so
    7ffff7ff2000 r--p 00025000 103:02 4859198    36   36   0         36         0  ld-2.33.so
    7ffff7ffb000 r--p 0002d000 103:02 4859198     8    8   8          8         8  ld-2.33.so
    7ffff7ffd000 rw-p 0002f000 103:02 4859198     8    8   8          8         8  ld-2.33.so
    7ffffffde000 rw-p 00000000  00:00       0   132   12  12         12        12  [stack]
ffffffffff600000 --xp 00000000  00:00       0     4    0   0          0         0  [vsyscall]
                                              ===== ==== === ========== =========  
                                              11300 1652 279       1652       260  KB </pre>

<p align="center">表2-5 进程的地址映射表</p>

程序运行之初会打印出`malloc address: 0x482700`，可以证明内存确实分配于 heap 上。

表 2-5 中黄色的两行代表，两个线程的 stack 区域，一个位于 heap 区域，另一个位于 mmap 区域。而红色部 rss 和 pss 可以理解为实际占用的物理内存，这部分会不断的增长，意味着我们的线程 stack 一直在扩充，直到发生`segmentation fault`。

表 2-5 中 heap 段的 size 为 732 KiB，这是因为我在申请 stack 所用的内存之前，连续调用了 5 次 malloc 共申请了 500KiB 的内存。否则，其大小会是 132KiB，这是由 malloc 管理内存的机制决定的，可参照 [Understanding glibc malloc](https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/comment-page-1/) 这篇文章。可见 pmap 读取的内容并不能识别出我们申请的每一块内存，对于无法识别的内容，在 smaps 中只会放在一个匿名的条目中，比如这里的 heap 区域。

Finally，我们终于可以画一个全面的 stack 角度的内存布局图了：

![图 2-11 位于不同区域的线程 stack](https://s3.ap-northeast-1.amazonaws.com/qiniu.liupzmin.com/stack-region.png)

## 4. 函数调用与栈帧

堆栈是一个后进先出的结构，它天然的适用于函数调用这种情况。这一节我们简单来看一下 stack 在函数调用方面的应用，同时也解答一下引发我写此系列文章的 stack 中变量引用的问题。

x86_64 架构的堆栈由高地址向低地址增长，寄存器`%rsp`始终指向栈顶元素。将堆栈的栈顶指针减小就可以在 stack 上分配空间，将指针增大就可以在 stack 上释放空间。

### 4.1 利用 stack 进行控制转移

每一个函数在 stack 分配的空间统称为该函数的栈帧（stack fram），图 2-12 给出了运行时 stack 的通用结构，包括把它划分为栈帧，当前正在执行的函数的帧总是在栈顶。假设 函数 P 在执行过程中调用了函数 Q，当函数 P 调用函数 Q 时，会把返回地址压入栈中，指明当 Q 返回时，要从 P 程序的哪个位置继续执行。我们把这个返回地址当做 P 的栈帧的一部分，因为它存放的是与 P 相关的状态。

![图 2-12 通用的栈帧结构](https://qiniu.liupzmin.com/stack-frame.png)

Q 的代码会扩展当前堆栈的边界，为它的栈帧分配所需的空间。在此空间中， Q 可以保存寄存器的值，分配局部变量，如果它还调用其它函数，则为被调用的函数设置参数。函数 P 可以通过寄存器传递最多 6 个参数，如果 Q 的参数个数超过了 6 个，则 P 在调用之前会在自己的栈帧里存储好这些参数。

函数调用需要打破当前 CPU 顺序执行指令的状态，使其跳转至另外一部分代码块。这种控制转移自然是通过修改程序计数器（PC）来达成的，将控制从 P 转移到 Q，仅需将 PC 修改为 Q 的代码的起始位置。不过当 Q 返回的时候，CPU 必须要知道它要在 P 中继续的位置。x86_64 机器中，这个过程是通过 `call`和`ret`指令配合完成的。

首先，函数调用通过`call Q`来进行，该指令会把地址 A 压入堆栈中，并将 PC 设置为 Q 的起始地址。压入的地址 A 通常叫做**返回地址**，是紧随 call 指令之后的那条指令的地址。对应的指令 `ret`会从堆栈中弹出地址 A，并把 PC 设置为 A。

可以看到，这种把返回地址压入堆栈的简单机制能够让函数在稍后返回到程序中正确的位置。函数的调用和返回机制刚好与堆栈提供的后进先出的内存管理方式相吻合。

### 4.2 需要保存的寄存器

CPU 中的寄存器是被所有的函数共享的，虽然在一个执行流中，同一时刻只有一个函数是活动的（在 CPU 上执行），但我们必须保证在函数返回时，它不会覆盖或者破坏调用者稍后会使用的寄存器的值。为此，x86_64 采用了一组统一的寄存器使用惯例，所有的函数调用都必须遵循。

依照此管理，寄存器被分为两类：

1. **需要被调用者保存的寄存器（callee-saved）**，寄存器 %rbx、%rbp 和 %r12~%r15 属于被调用者保存寄存器，图 2-12 中被保存的寄存器区域就是被调用者栈帧中存放寄存器值的位置。当函数 P 调用函数 Q 时，Q 必须保存这些寄存器的值，保证它们的值在 Q 返回到 P 的时候与 Q 被调用的时候是一样的。要做到这一点，Q 要么不去改变就这些寄存器的值，要么就要把原始值压入堆栈，然后在返回的时候从堆栈中弹出旧的值。我们稍后会看到`%rbp`寄存器就是这样的一个例子。
2. **需要调用者保存的寄存器（caller-saved）**，事实上除了栈顶指针寄存器 %rsp，所有其它的寄存器都属于调用者保存寄存器。

### 4.3 %rbp 和 %rsp

`%rsp` 是栈顶指针寄存器，其值永远指向 stack 的顶部。`%rbp` 大家可能会陌生一点，x86_64 代码使用该寄存器作为`帧指针（frame pointer）`，有时也称为`基指针（base pointer）`，这也是 %rbp 中 `bp` 两个字母的由来。看如下代码：

```c
long swap_add(long *xp, long *yp) {
  long x = *xp;
  long y = *yp;
  *xp = y;
  *yp = x;
  return x + y;
}

long caller(){
  long arg1 = 534;
  long arg2 = 1057;
  long sum = swap_add(&arg1, &arg2);
  long diff = arg1 - arg2;
  return sum * diff;
}
```

使用 `gcc -S` 编译为汇编指令：

```assembly
caller:
.LFB1:
	.cfi_startproc
	pushq	%rbp              ;将 %rbp 入栈
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp        ;将此时的栈顶指针设为 rbp
	.cfi_def_cfa_register 6
	subq	$48, %rsp         ;为此函数栈帧申请 48 字节
	movq	%fs:40, %rax      ;获取金丝雀值（堆栈保护机制）
	movq	%rax, -8(%rbp)    ;将金丝雀值存入堆栈
	xorl	%eax, %eax
	movq	$534, -40(%rbp)   ;将 534 存入 arg1
	movq	$1057, -32(%rbp)  ;将 1057 存入 arg2
	leaq	-32(%rbp), %rdx   ;将 arg2 的地址存入 %rdx 寄存器
	leaq	-40(%rbp), %rax   ;将 arg1 的地址存入 %rax 寄存器
	movq	%rdx, %rsi        ;将 %rdx 的值存入 %rsi 寄存器
	movq	%rax, %rdi        ;将 %rax 的值存入 %rdi 寄存器
	call	swap_add          ;调用 swap_add 函数
	movq	%rax, -24(%rbp)   ;将函数调用结果 sum 存入 stack 的 -24(%rbp) 处
	movq	-40(%rbp), %rax   ;将 arg1 放入 %rax
	movq	-32(%rbp), %rdx   ;将 arg2 放入 %rdx
	subq	%rdx, %rax        ;计算  diff = arg1 - arg2 将结果 diff 放入 %rax
	movq	%rax, -16(%rbp)   ;将 diff 入栈，放在 -16(%rbp) 处
	movq	-24(%rbp), %rax   ; 将 sum 读入 %rax
	imulq	-16(%rbp), %rax   ; 计算 sum * diff，结果放在 %rax 中准备返回
	movq	-8(%rbp), %rdx    ; 下面检查堆栈结构是否有损坏
	subq	%fs:40, %rdx
	je	.L5
	call	__stack_chk_fail@PLT
.L5:
	leave                     ;释放栈帧
	.cfi_def_cfa 7, 8
	ret                       ; 返回
	.cfi_endproc
```

此时函数的栈帧如图 2-13 一样：

![图 2-13 函数 caller 的栈帧结构](https://qiniu.liupzmin.com/rbp.png)

我们可以看到 caller 汇编指令的开始就是保存 %rbp 寄存器的值，因为 %rbp 属于 **callee-saved** 寄存器，它的值要由 `caller` 函数负责保存（此时的caller看作是被调用者）。在函数结尾，`ret`返回之前有一句`leave` 指令，它的作用是释放函数的栈帧，将 %rbp 寄存器恢复到 caller 被调用前的值，它等价于下面两条指令：

```assembly
movq %rbp,%rsp    ;直接将栈顶指针拉回到 %rbp 处
popq %rbp         ;将保存的 %rbp 出栈 放入 %rbp
```

从图 2-13 可以看出，`movq %rbp,%rsp`会将 %rbp 的值设置为栈顶，而此时 %rbp 指向栈帧的开始处，即刚保存完旧 %rbp 的位置。紧接着的操作`popq %rbp`会将栈顶的元素（保存的 %rbp）弹出，并设置到 %rbp 寄存器，这样 caller 函数的调用者的 %rbp 就恢复了。

我们从汇编指令上很容易看出堆栈上的寻址方式：在写入和读取堆栈上的数据时，使用的是类似于**-32(%rbp)**的**基址+偏移量**的寻址方式。毕竟，在一个函数的栈帧生存期中，`%rbp`是永远固定不变的，使用`%rbp`来相对寻址也就顺理成章了。

但是，gcc 中的优化选项会对能在编译期确定栈帧大小的函数使用`%rsp`定位，让我们再对比一下`gcc -Og -S`生成的汇编代码：

```assembly
caller:
.LFB1:
	.cfi_startproc
	subq	$40, %rsp           ;为此函数栈帧申请 40 字节
	.cfi_def_cfa_offset 48
	movq	%fs:40, %rax        ;获取金丝雀值（堆栈保护机制）
	movq	%rax, 24(%rsp)      ;将金丝雀值存入堆栈
	xorl	%eax, %eax
	movq	$534, 8(%rsp)       ;将 534 存入 arg1
	movq	$1057, 16(%rsp)     ;将 1057 存入 arg2
	leaq	16(%rsp), %rsi      ;将 arg2 的地址存入 %rsi 寄存器
	leaq	8(%rsp), %rdi       ;将 arg1 的地址存入 %rdi 寄存器
	call	swap_add            ;调用 swap_add 函数
	movq	8(%rsp), %rdx       ;将 arg1 存入 %rdx 寄存器
	subq	16(%rsp), %rdx      ; 计算  diff = arg1 - arg2
	imulq	%rdx, %rax          ; 计算 sum * diff
	movq	24(%rsp), %rdx      ; 下面检查堆栈结构是否有损坏
	subq	%fs:40, %rdx
	jne	.L5
	addq	$40, %rsp           ; 释放栈帧
	.cfi_remember_state
	.cfi_def_cfa_offset 8
	ret                         ;返回
```

可见此时的汇编代码中，用于定位堆栈内存的寻址方式变成了类似 **8(%rsp)** 的模式，而且使用**-Og**生成的汇编代码更符合 C 代码整体结构，我们的第一版汇编代码的和原始的 C 代码相比就有很大程度的变形，导致汇编代码和源代码之间的关系难以理解。

但是，这种使用栈顶指针定位的方式只适用于在编译期能够确定栈帧大小的情况，如果函数的栈帧是在运行中动态改变的。比如使用了变长数组、alloca等情况下，在堆栈上分配的字节数是任意的，也就无法确定 %rsp 和要寻址的内容之间的地址偏移量，此时便只有 %rbp 一种定位方式了，让我们看下面这个变长数组的例子：

```c
long vframe(long n, long idx, long *q){
  long i;
  long *p[n];
  p[0] = &i;
  for(i =1; i<n;i++){
    p[i] = q;
  }
  return *p[idx];
}
```

默认情况生成的汇编代码为：

```assembly
vframe:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	pushq	%rbx
	subq	$72, %rsp
	.cfi_offset 3, -24
	movq	%rdi, -56(%rbp)
	movq	%rsi, -64(%rbp)
	movq	%rdx, -72(%rbp)
	movq	%fs:40, %rax
	movq	%rax, -24(%rbp)
	xorl	%eax, %eax
	movq	%rsp, %rax
	movq	%rax, %rsi
	movq	-56(%rbp), %rax
	leaq	-1(%rax), %rdx
	movq	%rdx, -40(%rbp)
	movq	%rax, %rdx
	movq	%rdx, %r8
	movl	$0, %r9d
	movq	%rax, %rdx
	movq	%rdx, %rcx
	movl	$0, %ebx
	leaq	0(,%rax,8), %rdx
	movl	$16, %eax
	subq	$1, %rax
	addq	%rdx, %rax
	movl	$16, %ebx
	movl	$0, %edx
	divq	%rbx
	imulq	$16, %rax, %rax
	subq	%rax, %rsp
	movq	%rsp, %rax
	addq	$7, %rax
	shrq	$3, %rax
	salq	$3, %rax
	movq	%rax, -32(%rbp)
	movq	-32(%rbp), %rax
	leaq	-48(%rbp), %rdx
	movq	%rdx, (%rax)
	movq	$1, -48(%rbp)
	jmp	.L2
.L3:
	movq	-48(%rbp), %rdx
	movq	-32(%rbp), %rax
	movq	-72(%rbp), %rcx
	movq	%rcx, (%rax,%rdx,8)
	movq	-48(%rbp), %rax
	addq	$1, %rax
	movq	%rax, -48(%rbp)
.L2:
	movq	-48(%rbp), %rax
	cmpq	%rax, -56(%rbp)
	jg	.L3
	movq	-32(%rbp), %rax
	movq	-64(%rbp), %rdx
	movq	(%rax,%rdx,8), %rax
	movq	(%rax), %rax
	movq	%rsi, %rsp
	movq	-24(%rbp), %rdx
	subq	%fs:40, %rdx
	je	.L5
	call	__stack_chk_fail@PLT
.L5:
	movq	-8(%rbp), %rbx
	leave
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
```

vframe 的函数栈帧如图 2 - 14 所示：

![图 2-14 函数 vframe 的栈帧](https://qiniu.liupzmin.com/vframe.png)

可以看到，在整个函数的执行过程中，使用不变的 %rbp 帧指针寄存器和不同的偏移量来引用堆栈上的内容。

## 5. 总结

本篇名为《穿越虚拟内存的迷雾》，但并未深入的介绍虚拟内存，只是通过对虚拟内存的溯源来更好的理解 stack。

虚拟内存并非与生俱来，乃是先驱们在计算机的发展过程中总结出的有效的内存管理方式。它通过对存储的抽象为运行在计算机中的每个进程提供了统一的地址空间，并使用交换技术和分页使得计算机可以在有限的物理内存上运行比较大的程序。利用程序的局部性原理，让数据在磁盘和真实的物理内存之间以页的形式换入换出，以此节约了成本，提高了硬件的利用率。

Stack 是进程地址空间中举足轻重的一环，函数的调用与返回、局部变量的存储都依赖于它。但它的位置与大小却并不总是那么明朗，通过对 Linux 下 stack 分布和大小的探查，我们对 stack 的分配问题如管中窥豹，可见一斑。见诸各种资料上的进程地址空间布局图仅仅只是标明了主线程的 stack，并没有在线程概念普及后予以校正。或许这在计算机专家眼中根本不值一提，但它成功迷惑了我。因此我做了这些许微小的工作，以俟夫究察者得焉！

有了这些对于 stack 的基础认识，会更容易理解 Go 语言中协程的 stack，我们下一篇文章见😙~

在撰写此文时，深深感受到自身学识之匮乏、能力之浅薄，因此行文中定有疏漏讹误之处，也请阅读此文的朋友热情斧正。

***参考文献***

1. [操作系统导论](https://book.douban.com/subject/33463930/)
2. [深入理解计算机系统](https://book.douban.com/subject/26912767/)
3. [程序员的自我修养](https://book.douban.com/subject/3652388/)
4. [Linux/UNIX系统编程手册](https://book.douban.com/subject/25809330/)
5. [深入 Linux 内核架构](https://book.douban.com/subject/4843567/)
6. [现代操作系统](https://book.douban.com/subject/27096665/)
7. [mm](https://www.kernel.org/doc/Documentation/x86/x86_64/mm.txt)
8. [How to mmap the stack for the clone() system call on linux?](https://stackoverflow.com/questions/1083172/how-to-mmap-the-stack-for-the-clone-system-call-on-linux)
9. [allocatestack.c](https://github.com/lattera/glibc/blob/a2f34833b1042d5d8eeb263b4cf4caaea138c4ad/nptl/allocatestack.c)
10.  [Understanding glibc malloc](https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/comment-page-1/)
