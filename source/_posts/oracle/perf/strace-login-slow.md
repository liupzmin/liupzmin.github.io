---
date: 2016-01-05 21:29
categories: strace
tags: strace
title: strace解决sqlplus登陆缓慢的问题一例
---

**公司一台数据库测试服务器，某天开始sqlplus登陆缓慢，要卡四五秒才能登陆成功，而且是sqlplus / as  sysdba**

**感觉问题排查无法下手，又不是什么大问题，库也不重要，一直没有去管**

**今天闲来无事，决定着手解决这个问题，排查此类问题的终极大杀器，祭出strace**
>[oracle@test8 ~]$ strace -T -t -o /tmp/nohost sqlplus / as sysdba

>查看/tmp/nohost内容，此处省略部分内容

```
10:05:39 open("/etc/hosts", O_RDONLY|O_CLOEXEC) = 9 <0.000086>
10:05:39 fstat(9, {st_mode=S_IFREG|0644, st_size=208, ...}) = 0 <0.000061>
10:05:39 mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fb66c9a2000 <0.000063>
10:05:39 read(9, "127.0.0.1 localhost localhost."..., 4096) = 208 <0.000069>
10:05:39 read(9, "", 4096) = 0 <0.000059>
10:05:39 close(9) = 0 <0.000125>
10:05:39 munmap(0x7fb66c9a2000, 4096) = 0 <0.000078>
10:05:39 open("/home/app/oracle/product/11.2.0/dbhome_1/lib/libnss_dns.so.2", O_RDONLY) = -1 ENOENT (No such file or directory) <0.000099>
10:05:39 open("/etc/ld.so.cache", O_RDONLY) = 9 <0.000085>
10:05:39 fstat(9, {st_mode=S_IFREG|0644, st_size=404774, ...}) = 0 <0.000059>
10:05:39 mmap(NULL, 404774, PROT_READ, MAP_PRIVATE, 9, 0) = 0x7fb66c71c000 <0.000069>
10:05:39 close(9) = 0 <0.000058>
10:05:39 open("/lib64/libnss_dns.so.2", O_RDONLY) = 9 <0.000090>
10:05:39 read(9, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\20\0\0\0\0\0\0"..., 832) = 832 <0.000062>
10:05:39 fstat(9, {st_mode=S_IFREG|0755, st_size=27424, ...}) = 0 <0.000060>
10:05:39 mmap(NULL, 2117880, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 9, 0) = 0x7fb66c13b000 <0.000067>
10:05:39 mprotect(0x7fb66c140000, 2093056, PROT_NONE) = 0 <0.000074>
10:05:39 mmap(0x7fb66c33f000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 9, 0x4000) = 0x7fb66c33f000 <0.000071>
10:05:39 close(9) = 0 <0.000058>
10:05:39 open("/home/app/oracle/product/11.2.0/dbhome_1/lib/libresolv.so.2", O_RDONLY) = -1 ENOENT (No such file or directory) <0.000100>
10:05:39 open("/lib64/libresolv.so.2", O_RDONLY) = 9 <0.000089>
10:05:39 read(9, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\00009\300E>\0\0\0"..., 832) = 832 <0.000060>
10:05:39 fstat(9, {st_mode=S_IFREG|0755, st_size=113952, ...}) = 0 <0.000060>
10:05:39 mmap(0x3e45c00000, 2202248, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 9, 0) = 0x3e45c00000 <0.000066>
10:05:39 mprotect(0x3e45c16000, 2097152, PROT_NONE) = 0 <0.000067>
10:05:39 mmap(0x3e45e16000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 9, 0x16000) = 0x3e45e16000 <0.000077>
10:05:39 mmap(0x3e45e18000, 6792, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x3e45e18000 <0.000072>
10:05:39 close(9) = 0 <0.000058>
10:05:39 mprotect(0x3e45e16000, 4096, PROT_READ) = 0 <0.000123>
10:05:39 mprotect(0x7fb66c33f000, 4096, PROT_READ) = 0 <0.000069>
10:05:39 munmap(0x7fb66c71c000, 404774) = 0 <0.000082>
10:05:39 socket(PF_INET, SOCK_DGRAM|SOCK_NONBLOCK, IPPROTO_IP) = 9 <0.000094>
10:05:39 connect(9, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("221.228.255.1")}, 16) = 0 <0.000076>
10:05:39 poll([{fd=9, events=POLLOUT}], 1, 0) = 1 ([{fd=9, revents=POLLOUT}]) <0.000061>
10:05:39 sendto(9, "\23\232\1\0\0\1\0\0\0\0\0\0\5test8\0\0\1\0\1", 23, MSG_NOSIGNAL, NULL, 0) = 23 <0.000201>
10:05:39 poll([{fd=9, events=POLLIN|POLLOUT}], 1, 5000) = 1 ([{fd=9, revents=POLLOUT}]) <0.000062>
10:05:39 sendto(9, "\3d\1\0\0\1\0\0\0\0\0\0\5test8\0\0\34\0\1", 23, MSG_NOSIGNAL, NULL, 0) = 23 <0.000098>
10:05:39 poll([{fd=9, events=POLLIN}], 1, 4998) = 1 ([{fd=9, revents=POLLIN}]) <0.006345>
10:05:39 ioctl(9, FIONREAD, [39]) = 0 <0.000063>
10:05:39 recvfrom(9, "\23\232\201\200\0\1\0\1\0\0\0\0\5test8\0\0\1\0\1\300\f\0\1\0\1\0\0\0"..., 2048, 0, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("221.228.255.1")}, [16]) = 39 <0.000096>
10:05:39 poll([{fd=9, events=POLLIN}], 1, 4991) = 0 (Timeout) <4.996143>
10:05:44 poll([{fd=9, events=POLLOUT}], 1, 0) = 1 ([{fd=9, revents=POLLOUT}]) <0.000100>
10:05:44 sendto(9, "\23\232\1\0\0\1\0\0\0\0\0\0\5test8\0\0\1\0\1", 23, MSG_NOSIGNAL, NULL, 0) = 23 <0.000198>
10:05:44 poll([{fd=9, events=POLLIN}], 1, 5000) = 1 ([{fd=9, revents=POLLIN}]) <0.006851>
10:05:44 ioctl(9, FIONREAD, [39]) = 0 <0.000151>
10:05:44 recvfrom(9, "\23\232\201\200\0\1\0\1\0\0\0\0\5test8\0\0\1\0\1\300\f\0\1\0\1\0\0\0"..., 2048, 0, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("221.228.255.1")}, [16]) = 39 <0.000107>
10:05:44 poll([{fd=9, events=POLLOUT}], 1, 4990) = 1 ([{fd=9, revents=POLLOUT}]) <0.000088>
10:05:44 sendto(9, "\3d\1\0\0\1\0\0\0\0\0\0\5test8\0\0\34\0\1", 23, MSG_NOSIGNAL, NULL, 0) = 23 <0.000120>
10:05:44 poll([{fd=9, events=POLLIN}], 1, 4990) = 1 ([{fd=9, revents=POLLIN}]) <0.008139>
10:05:44 ioctl(9, FIONREAD, [98]) = 0 <0.000074>
10:05:44 recvfrom(9, "\3d\201\203\0\1\0\0\0\1\0\0\5test8\0\0\34\0\1\0\0\6\0\1\0\0\35\17"..., 2009, 0, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("221.228.255.1")}, [16]) = 98 <0.000084>
10:05:44 close(9) = 0 <0.000143>
10:05:44 open("/etc/hostid", O_RDONLY) = -1 ENOENT (No such file or directory) <0.000095>
10:05:44 uname({sys="Linux", node="test8", ...}) = 0 <0.000061>
10:05:44 open("/etc/hosts", O_RDONLY|O_CLOEXEC) = 9 <0.000114>
10:05:44 fstat(9, {st_mode=S_IFREG|0644, st_size=208, ...}) = 0 <0.000062>
10:05:44 mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fb66c9a2000 <0.000071>
10:05:44 read(9, "127.0.0.1 localhost localhost."..., 4096) = 208 <0.000089>
10:05:44 read(9, "", 4096) = 0 <0.000059>
10:05:44 close(9) = 0 <0.000098>
10:05:44 munmap(0x7fb66c9a2000, 4096) = 0 <0.000090>
10:05:44 socket(PF_INET, SOCK_DGRAM|SOCK_NONBLOCK, IPPROTO_IP) = 9 <0.000092>
10:05:44 connect(9, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("221.228.255.1")}, 16) = 0 <0.000072>
10:05:44 poll([{fd=9, events=POLLOUT}], 1, 0) = 1 ([{fd=9, revents=POLLOUT}]) <0.000059>
10:05:44 sendto(9, "\333\360\1\0\0\1\0\0\0\0\0\0\5test8\0\0\1\0\1", 23, MSG_NOSIGNAL, NULL, 0) = 23 <0.000159>
10:05:44 poll([{fd=9, events=POLLIN}], 1, 5000) = 1 ([{fd=9, revents=POLLIN}]) <0.008444>
10:05:44 ioctl(9, FIONREAD, [39]) = 0 <0.000067>
10:05:44 recvfrom(9, "\333\360\201\200\0\1\0\1\0\0\0\0\5test8\0\0\1\0\1\300\f\0\1\0\1\0\0\0"..., 1024, 0, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("221.228.255.1")}, [16]) = 39 <0.000077>
10:05:44 close(9) = 0 <0.000086>
10:05:44 write(10, "\2x\0\0\6\0\0\0\0\0\3s\3\0\0\0\0\0\0\0\0\0\0\0\0!\0\0\0\376\377\377"..., 632) = 632 <0.000124>
10:05:44 read(11, "\6\331\0\0\6\0\0\0\0\0\10&\0\23\0\0\0\23AUTH_VERSION_S"..., 8208) = 1753 <0.013111>
10:05:44 open("/home/app/oracle/product/11.2.0/dbhome_1//rdbms/mesg/oraus.msb", O_RDONLY) = 9 <0.000142>
10:05:44 fcntl(9, F_SETFD, FD_CLOEXEC) = 0 <0.000060>
10:05:44 lseek(9, 0, SEEK_SET) = 0 <0.000062>
10:05:44 read(9, "\25\23\"\1\23\3\t\t\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 256) = 256 <0.000091>
10:05:44 lseek(9, 512, SEEK_SET) = 512 <0.000060>
```
>其中*10:05:39 poll([{fd=9, events=POLLIN}], 1, 4991) = 0 (Timeout) <4.996143>*是耗时最长的一步
>而且卡在此处足足有5s，而且最终超时，根据上下文判断，此处是一个socket连接，而且还涉及到dns服务器的地址，由此推断应该和解析有关

查看/etc/resolv.conf

```
[root@test8 etc]# cat /etc/resolv.conf
# Generated by NetworkManager
#nameserver 221.228.255.1
nameserver 221.228.255.1
nameserver 218.2.135.1
```
>可见配置了dns，其实我将dns注释掉之后sqlplus就恢复了正常，但dns不应该成为导致此问题的根本原因，此处明显是去dns上解析本机了,再来看/etc/hosts

```
[root@test8 etc]# cat /etc/hosts
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.8 test8.com
192.168.1.29 oem.oracle.com
```
>在重新查阅了hosts的正确配置之后发现我的hosts配置方法是错误的，以下为正确格式

```
格式：
　　一般情况下hosts的内容关于主机名(hostname)的定义，每行为一个主机，每行由三部份组成，每个部份由空格隔开。其中#号开头的行做说明，不被系统解释。
　　第一部份：网络IP地址；
　　第二部份：主机名.域名，注意主机名和域名之间有个半角的点，比如 localhost.localdomain
　　第二部份：主机名(主机名别名） ，其实就是主机名；
```
>我的主机名是test8，在我测试邮件的时候，为了骗过邮件服务器，我把hosts里解析随意加了个域名，却从未正式了解过hosts的配置方法，现在正确配置一下

```
[root@test8 etc]# cat /etc/hosts
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.8 test8.com test8
192.168.1.29 oem.oracle.com
```
>再测试sqlplus，已经恢复正常


>然而有些问题仍然比较困惑，正解析与反解析何时会进行？为什么会进行？为什么hosts和dns都不存在时候反而没问题，而dns存在hosts配置不正确的时候为什么会出问题？这些疑惑以我现在的能力还无法参破，就连strace的内容也只看个皮毛，期待来日解决吧！ 