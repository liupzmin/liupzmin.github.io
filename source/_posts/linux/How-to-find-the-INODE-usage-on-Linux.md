---
tags: linux
title: Linux inode 小记
date: 2019-08-05 13:59:15
categories: linux
---

## 什么是inode

理解inode，要从文件储存说起。

文件储存在硬盘上，硬盘的最小存储单位叫做"扇区"（Sector）。每个扇区储存512字节（相当于0.5KB）。

操作系统读取硬盘的时候，不会一个个扇区地读取，这样效率太低，而是一次性连续读取多个扇区，即一次性读取一个"块"（block）。这种由多个扇区组成的"块"，是文件存取的最小单位。"块"的大小，最常见的是4KB，即连续八个 sector组成一个 block。

文件数据都储存在"块"中，那么很显然，我们还必须找到一个地方储存文件的元信息，比如文件的创建者、文件的创建日期、文件的大小等等。这种储存文件元信息的区域就叫做inode，中文译名为"索引节点"。

每一个文件都有对应的inode，里面包含了与该文件有关的一些信息。

## inode的内容

inode包含文件的元信息，具体来说有以下内容：

- 文件的字节数
- 文件拥有者的User ID
- 文件的Group ID
- 文件的读、写、执行权限
- 文件的时间戳
- 链接数，即有多少文件名指向这个inode
- 文件数据block的位置

使用`stat`命令查看某个文件的`inode信息`：

```bash
 $ stat proxy.sh 
  File: proxy.sh
  Size: 89        	Blocks: 8          IO Block: 4096   regular file
Device: 802h/2050d	Inode: 9714171     Links: 1
Access: (0755/-rwxr-xr-x)  Uid: ( 1000/ liupeng)   Gid: ( 1000/ liupeng)
Access: 2019-07-29 10:25:57.880429139 +0800
Modify: 2019-07-29 10:25:57.880429139 +0800
Change: 2019-07-29 10:26:05.910683677 +0800
 Birth: 2019-07-29 10:25:57.880429139 +0800
```

## inode的大小

inode也会消耗硬盘空间，所以硬盘格式化的时候，操作系统自动将硬盘分成两个区域。一个是数据区，存放文件数据；另一个是inode区（inode table），存放inode所包含的信息。

每个inode节点的大小，一般是128字节或256字节，甚至可以手动指定到2K，inode节点的总数，在格式化时就给定，以ext3/ext4为例：
- 每个 inode 大小为 256byte，block 大小为 4k byte；
- 根据 block count 和 inode count，我们也可以算出 16k bytes-per-inode（15728384*4096/3932160）

也就是文件系统在创建的时候每16k空间自动划分一个inode，如果你需要存储的是大量的小文件，那么你应该在格式化分区的时候手动修改`bytes-per-inode`的值，例如：
```shell
mkfs.ext4 -i 8192 /dev/sda1
```
而在`xfs`文件系统中`-i inode_options`中的`maxpct=value`描述如下：

```shell
This  specifies  the  maximum percentage of space in the filesystem that can be allocated to inodes. The default value is 25% for filesystems under 1TB, 5% for filesystems under 50TB and 1% for filesystems over 50TB.
```
可见默认情况下`xfs`文件系统要比`ext`文件系统分配更多的inode

可以使用`df -i`查看inode的大小和使用率
```shell
$ df -i
Filesystem       Inodes   IUsed    IFree IUse% Mounted on
dev             1015658     414  1015244    1% /dev
run             1017797     680  1017117    1% /run
/dev/sda2      14057472 1003259 13054213    8% /
tmpfs           1017797     129  1017668    1% /dev/shm
tmpfs           1017797      18  1017779    1% /sys/fs/cgroup
tmpfs           1017797     629  1017168    1% /tmp
/dev/sda1             0       0        0     - /boot/efi
tmpfs           1017797      37  1017760    1% /run/user/1000
```

## inode号

每个inode都有一个号码，操作系统用inode号码来识别不同的文件。

这里值得重复一遍，Unix/Linux系统内部不使用文件名，而使用inode号码来识别文件。对于系统来说，文件名只是inode号码便于识别的别称或者绰号。

表面上，用户通过文件名，打开文件。实际上，系统内部这个过程分成三步：首先，系统找到这个文件名对应的inode号码；其次，通过inode号码，获取inode信息；最后，根据inode信息，找到文件数据所在的block，读出数据。

使用`ls -i`命令，可以看到文件名对应的inode号码：
```shell
$ ls -i proxy.sh 
9714171 proxy.sh
```

## 目录文件

Unix/Linux系统中，目录（directory）也是一种文件。打开目录，实际上就是打开目录文件。

目录文件的结构非常简单，就是一系列目录项（dirent）的列表。每个目录项，由两部分组成：所包含文件的文件名，以及该文件名对应的inode号码。

## 查看目录的inode

通过介绍我们知道，通常情况下每个文件对应一个inode，那么如果想查找某个目录使用的inode数量，则可以使用如下命令：

```shell
clear;echo "Detailed Inode usage: $(pwd)" ; for d in `find -maxdepth 1 -type d |cut -d\/ -f2 |grep -xv . |sort`; do c=$(find $d |wc -l) ; printf "$c\t\t- $d\n" ; done ; printf "Total: \t\t$(find $(pwd) | wc -l)\n"
```

输出如下：

```shell
Detailed Inode usage: /home/liupeng/liupzmin.github.io
312		    - .git
8049		- node_modules
294		    - public
4		    - scaffolds
45		    - source
409		    - themes
Total: 		    9120
```

要清理inode，只要找到包含大量文件的目录删除之即可。


**参考文献：**

1. [理解inode](http://www.ruanyifeng.com/blog/2011/12/inode.html)
2. [How to find the INODE usage on Linux](https://thegeeksalive.com/how-to-find-the-inode-usage-on-linux/)