---
title: enq之TM-contention解决之道——外键无索引导致锁争用（下）
tags: lock
categories: oracle 
date: 2016-07-27 16:21
---

上篇文章简要介绍了一下当外键无索引时，更新删除主表的数据会造成子表的锁定，如果此时子表上有事务，那么进行更新删除的session变会等待，等待事件就是`enq: TM - contention`

外键与 TM enqueue lock 的主要问题是 在早期版本中(9i之前) 当 子表child table上 的外键没有索引时 ， 若发生 父表 parent table 上记录被delete 或 update时 ， 会在child table上加 share lock， 这会 阻塞 child table 上的DML。
但是从 9i以后的当 子表child table上 的外键没有索引时， 父表parent table上的delete 、update 只在 实际这个DML执行的过程中要求`share (TM lmode=4)` lock，而不会在整个事务中 都要求保持 child table上的 share lock。

**还是先了解一下oracle中的锁模式吧，TM锁和TX锁都属于DML锁，这里介绍的是TM的锁模式**

```
Value   Name(s)                    Table method (TM lock)
    0   No lock                    n/a

    1   Null lock (NL)             Used during some parallel DML operations (e.g. update) by
                                   the pX slaves while the QC is holding an exclusive lock.

    2   Sub-share (SS)             Until 9.2.0.5/6 "select for update"
        Row-share (RS)             Since 9.2.0.1/2 used at opposite end of RI during DML
                                   Lock table in row share mode
                                   Lock table in share update mode

    3   Sub-exclusive(SX)          Update (also "select for update" from 9.2.0.5/6)
        Row-exclusive(RX)          Lock table in row exclusive mode
                                   Since 11.1 used at opposite end of RI during DML

    4   Share (S)                  Lock table in share mode
                                   Can appear during parallel DML with id2 = 1, in the PX slave sessions
                                   Common symptom of "foreign key locking" (missing index) problem

    5   share sub exclusive (SSX)  Lock table in share row exclusive mode
        share row exclusive (SRX)  Less common symptom of "foreign key locking" but likely to be more
                                   frequent if the FK constraint is defined with "on delete cascade."

    6   Exclusive (X)              Lock table in exclusive mode
```


share lock就是mode为4的S锁

**Summary of Locks Obtained by DML Statements**

![Summary of Locks Obtained by DML Statements](http://qiniu.liupzmin.com/Summary-of-Locks-Obtained-by-DML-Statements.jpg)

**TM 锁在下列场景中被申请：**
- 在OPS(早期的RAC)中LGWR会以ID1=0 &  ID2=0去申请该队列锁来检查 DML_LOCKS 在所有实例中是全0还是全非0
- 当一个单表或分区 需要做不同的表/分区操作时，ORACLE需要协调这些操作，所以需要申请该队列锁。包括：
- 启用参考约束 referential constraints
- 修改约束从DIASABLE NOVALIDATE 到DISABLE VALIDATE
- 重建IOT
- 创建视图或者修改ALTER视图时可能需要申请该队列锁
- 分析表统计信息或validate structure时
- 一些PDML并行DML操作
- 所有可能调用kkdllk()函数的操作
- 太多太多了。。。

**下面是各种锁之间的兼容性:**

![lock](http://qiniu.liupzmin.com/lock-comp.png)

好了，开始动手做个试验吧，试验中我会引用KST trace的内容，关于KST，本文不做介绍，只拿来使用

首先，准备环境，本实验均在11.2.0.4环境下

```sql
SQL> conn lp/lp
Connected.
SQL> create table prim(a int,b varchar2(10));

Table created.

SQL> alter table prim add constraint PK_PRIM primary key(a);

Table altered.

SQL> create table child (ca int,cb varchar2(10));

Table created.

SQL> alter table child add constraint FK_CHILD_CA foreign key (ca) references prim(a);

Table altered.

SQL> insert into prim values(1,'asdasd');

1 row created.

SQL> insert into prim values(2,'asdasd');

1 row created.

SQL> insert into prim values(3,'asdasd');

1 row created.

SQL> commit;

Commit complete.
```

这里要说一下，在外键是否存在`on delete cascade`时锁的获取还有区别，所以我们分别来测试，首先是没有索引没有cascade的情况下，各个语句的锁获取情况

## 无索引，无cascade

```sql
SQL> select distinct sid from v$mystat;

       SID
----------
        17

SQL> select pid,spid from v$process where addr = ( select paddr from v$session where sid=(select distinct sid from v$mystat));

       PID SPID
---------- ------------
        36   2761 

SQL> alter system set "_trace_events"='10000-10999:255:36';

System altered.
```

insert 父表：
```sql
insert into prim values(5,'asdasd');
```

查看kst信息:

```sql
select kst.event,kst.sid,kst.pid,kst.function,kst.data from x$trace kst where pid=36 and sid=17;

10704 17 36 ksqgtlctx ksqgtl: acquire TM-0001563c-00000000 mode=SX flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10704 17 36 ksqgtlctx ksqgtl: acquire TM-0001563e-00000000 mode=SX flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10813 17 36 ktubnd ktubnd: Bind usn 3 nax 1 nbx 0 lng 0 par 0
10813 17 36 ktubnd ktubnd: Txn Bound xid: 3.3.1150
10704 17 36 ksqgtlctx ksqgtl: acquire TX-00030003-0000047e mode=X flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10811 17 36 ktbgcl1 3301000100000000 0000000000000000 69dd140000000000 0200000000000000
10811 17 36 ktbgcl1 3301000100000000 0000000000000000 0de3140000000000 0810c76a177f0000
10005 17 36 kslwtbctx KSL WAIT BEG [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=173 seq_num=180 snap_id=1
10005 17 36 kslwtectx KSL WAIT END [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=173 seq_num=180 snap_id=1
10005 17 36 kslwtectx KSL WAIT END wait times (usecs) - snap=4, exc=4, tot=4
10005 17 36 kslwtbctx KSL WAIT BEG [SQL*Net message from client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=174 seq_num=181 snap_id=1
```
可见父表上的插入会获取父表和子表`mode`为`3`的`TM锁`，TM后跟的是object_id的十六进制，一个TX锁，让我们验证一下:

```sql
SQL> select * from v$lock where type in('TM','TX');

ADDR              KADDR           SID  TY        ID1         ID2    IMODE   REQUEST    CTIME     BLOCK
----------------  -------------- ----  -- ---------- ----------- -------- --------- -------- ---------
00007FD97FFE54E8  00007FD97FFE5548 17  TM      87614           0        3         0        8         0
00000000ACE37CD0  00000000ACE37D48 17  Tx     262162         851        6         0        8         0
00007FD97EFE54E8  00007FD97FFE5548 17  TM      87612           0        3         0        8         0
```

我们来commit一下

```
10704 17 36 ksqrcli ksqrcl: release TX-00030003-0000047e mode=X
10813 17 36 ktudnx ktudnx: dec cnt xid:3.3.1150 nax:0 nbx:0
10704 17 36 ksqrcli ksqrcl: release TM-0001563e-00000000 mode=SX
10704 17 36 ksqrcli ksqrcl: release TM-0001563c-00000000 mode=SX
10021 17 36 kcrf_commit_force 2ee3140000000000 2fe3140000000000
10005 17 36 kslwtbctx KSL WAIT BEG [log file sync] 7416/0x1cf8 1368878/0x14e32e 0/0x0 wait_id=175 seq_num=182 snap_id=1
10005 17 36 ksliwat KSL FACILITY WAIT fac#=3 time_waited_csecs=1
10005 17 36 ksliwat KSL POST RCVD poster=11 num=76 loc='ksl2.h LINE:2374 ID:kslpsr' id1=138 id2=0 name=EV type=0 fac#=3 posted=0x3 may_be_posted=1
10005 17 36 kslwtectx KSL WAIT END [log file sync] 7416/0x1cf8 1368878/0x14e32e 0/0x0 wait_id=175 seq_num=182 snap_id=1
10005 17 36 kslwtectx KSL WAIT END wait times (usecs) - snap=11126, exc=11126, tot=11126
10005 17 36 kslwtbctx KSL WAIT BEG [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=176 seq_num=183 snap_id=1
10005 17 36 kslwtectx KSL WAIT END [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=176 seq_num=183 snap_id=1
10005 17 36 kslwtectx KSL WAIT END wait times (usecs) - snap=3, exc=3, tot=3
10005 17 36 kslwtbctx KSL WAIT BEG [SQL*Net message from client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=177 seq_num=184 snap_id=1
```

可见获得的锁全部一一释放

insert子表：
```sql
insert into child values(2,'sadsada');

10704 17 36 ksqgtlctx ksqgtl: acquire TM-0001563c-00000000 mode=SX flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10704 17 36 ksqgtlctx ksqgtl: acquire TM-0001563e-00000000 mode=SX flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10813 17 36 ktubnd ktubnd: Bind usn 7 nax 1 nbx 0 lng 0 par 0
10813 17 36 ktubnd ktubnd: Txn Bound xid: 7.17.835
10704 17 36 ksqgtlctx ksqgtl: acquire TX-00070011-00000343 mode=X flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10005 17 36 kslwtbctx KSL WAIT BEG [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=186 seq_num=193 snap_id=1
10005 17 36 kslwtectx KSL WAIT END [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=186 seq_num=193 snap_id=1
10005 17 36 kslwtectx KSL WAIT END wait times (usecs) - snap=3, exc=3, tot=3
10005 17 36 kslwtbctx KSL WAIT BEG [SQL*Net message from client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=187 seq_num=194 snap_id=1
```

可见子表上的插入也会获取父表和子表`mode`为`3`的`TM锁`

update父表：

```sql
update prim set a=1 where a=1;

10704 17 36 ksqgtlctx ksqgtl: acquire TM-0001563c-00000000 mode=SX flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10704 17 36 ksqgtlctx ksqgtl: acquire TM-0001563e-00000000 mode=S flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10704 17 36 ksqrcli ksqrcl: release TM-0001563e-00000000 mode=S
10704 17 36 ksqrcli ksqrcl: SUCCESS
10811 17 36 ktbgcl1 2c01000100000000 0000000000000000 0fe7140000000000 0200000000000000
10811 17 36 ktbgcl1 2c01000100000000 0000000000000000 27e9140000000000 b80fb86a177f0000
10813 17 36 ktubnd ktubnd: Bind usn 4 nax 1 nbx 0 lng 0 par 0
10813 17 36 ktubnd ktubnd: Txn Bound xid: 4.10.851
10704 17 36 ksqgtlctx ksqgtl: acquire TX-0004000a-00000353 mode=X flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10005 17 36 kslwtbctx KSL WAIT BEG [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=204 seq_num=211 snap_id=1
10005 17 36 kslwtectx KSL WAIT END [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=204 seq_num=211 snap_id=1
10005 17 36 kslwtectx KSL WAIT END wait times (usecs) - snap=5, exc=5, tot=5
10005 17 36 kslwtbctx KSL WAIT BEG [SQL*Net message from client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=205 seq_num=212 snap_id=1
```

可以很清楚的看到在执行语句期间，注意仅仅是语句的执行期间，会附加一个mode为4的S锁到子表上，很快便释放了

delete父表：
```sql
delete from prim where a=4

10704 17 36 ksqgtlctx ksqgtl: acquire TM-0001563c-00000000 mode=SX flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10704 17 36 ksqgtlctx ksqgtl: acquire TM-0001563e-00000000 mode=S flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10704 17 36 ksqrcli ksqrcl: release TM-0001563e-00000000 mode=S
10704 17 36 ksqrcli ksqrcl: SUCCESS
10813 17 36 ktubnd ktubnd: Bind usn 2 nax 1 nbx 0 lng 0 par 0
10813 17 36 ktubnd ktubnd: Txn Bound xid: 2.0.1117
10704 17 36 ksqgtlctx ksqgtl: acquire TX-00020000-0000045d mode=X flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10704 17 36 ksqgtlctx ksqgtl: acquire TM-0001563e-00000000 mode=S flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10704 17 36 ksqrcli ksqrcl: release TM-0001563e-00000000 mode=S
10704 17 36 ksqrcli ksqrcl: SUCCESS
10812 17 36 ktrgcm 3b01000100000000 0000000000000000 bae9140000000000
10812 17 36 ktrgcm 0200000000000000 0000000000000000 5d04000000000000
10812 17 36 ktrgcm 9b12c00000000000 4401000000000000 0200000000000000
10812 17 36 ktrgcm 3b01000100000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 3c01000100000000 0000000000000000 bae9140000000000
10812 17 36 ktrgcm 0200000000000000 0000000000000000 5d04000000000000
10812 17 36 ktrgcm 9b12c00000000000 4401000000000000 0200000000000000
10812 17 36 ktrgcm 3c01000100000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 3d01000100000000 0000000000000000 bae9140000000000
10812 17 36 ktrgcm 0200000000000000 0000000000000000 5d04000000000000
10812 17 36 ktrgcm 9b12c00000000000 4401000000000000 0200000000000000
10812 17 36 ktrgcm 3d01000100000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 3e01000100000000 0000000000000000 bae9140000000000
10812 17 36 ktrgcm 0200000000000000 0000000000000000 5d04000000000000
10812 17 36 ktrgcm 9b12c00000000000 4401000000000000 0200000000000000
10812 17 36 ktrgcm 3e01000100000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 3f01000100000000 0000000000000000 bae9140000000000
10812 17 36 ktrgcm 0200000000000000 0000000000000000 5d04000000000000
10812 17 36 ktrgcm 9b12c00000000000 4401000000000000 0200000000000000
10812 17 36 ktrgcm 3f01000100000000 0000000000000000 0000000000000000
10005 17 36 kslwtbctx KSL WAIT BEG [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=227 seq_num=234 snap_id=1
10005 17 36 kslwtectx KSL WAIT END [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=227 seq_num=234 snap_id=1
10005 17 36 kslwtectx KSL WAIT END wait times (usecs) - snap=3, exc=3, tot=3
10005 17 36 kslwtbctx KSL WAIT BEG [SQL*Net message from client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=228 seq_num=235 snap_id=1
```

delete跟update比多了一次S锁的获取和释放，为何呢，是否和删除的行数有关？我们再多删一行试试

```sql
SQL> delete from prim where a=4 or a=5;

10704 17 36 ksqgtlctx ksqgtl: acquire TM-0001563c-00000000 mode=SX flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10704 17 36 ksqgtlctx ksqgtl: acquire TM-0001563e-00000000 mode=S flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10704 17 36 ksqrcli ksqrcl: release TM-0001563e-00000000 mode=S
10704 17 36 ksqrcli ksqrcl: SUCCESS
10813 17 36 ktubnd ktubnd: Bind usn 6 nax 1 nbx 0 lng 0 par 0
10813 17 36 ktubnd ktubnd: Txn Bound xid: 6.15.1290
10704 17 36 ksqgtlctx ksqgtl: acquire TX-0006000f-0000050a mode=X flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10704 17 36 ksqgtlctx ksqgtl: acquire TM-0001563e-00000000 mode=S flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10704 17 36 ksqrcli ksqrcl: release TM-0001563e-00000000 mode=S
10704 17 36 ksqrcli ksqrcl: SUCCESS
10812 17 36 ktrgcm 3b01000100000000 0000000000000000 5eea140000000000
10812 17 36 ktrgcm 0600000000000000 0f00000000000000 0a05000000000000
10812 17 36 ktrgcm bc06c00000000000 0e01000000000000 2200000000000000
10812 17 36 ktrgcm 3b01000100000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 3c01000100000000 0000000000000000 5eea140000000000
10812 17 36 ktrgcm 0600000000000000 0f00000000000000 0a05000000000000
10812 17 36 ktrgcm bc06c00000000000 0e01000000000000 2200000000000000
10812 17 36 ktrgcm 3c01000100000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 3d01000100000000 0000000000000000 5eea140000000000
10812 17 36 ktrgcm 0600000000000000 0f00000000000000 0a05000000000000
10812 17 36 ktrgcm bc06c00000000000 0e01000000000000 2200000000000000
10812 17 36 ktrgcm 3d01000100000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 3e01000100000000 0000000000000000 5eea140000000000
10812 17 36 ktrgcm 0600000000000000 0f00000000000000 0a05000000000000
10812 17 36 ktrgcm bc06c00000000000 0e01000000000000 2200000000000000
10812 17 36 ktrgcm 3e01000100000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 3f01000100000000 0000000000000000 5eea140000000000
10812 17 36 ktrgcm 0600000000000000 0f00000000000000 0a05000000000000
10812 17 36 ktrgcm bc06c00000000000 0e01000000000000 2200000000000000
10812 17 36 ktrgcm 3f01000100000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 3301000100000000 0000000000000000 5eea140000000000
10812 17 36 ktrgcm 0000000000000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 0000000000000000 0000000000000000 0000000000000000
10811 17 36 ktbgcl1 3301000100000000 0000000000000000 5eea140000000000 0100000000000000
10811 17 36 ktbgcl1 3301000100000000 0000000000000000 5eea140000000000 e00fc76a177f0000
10812 17 36 kturCRBackoutOneChg 0100000000000000 bc06c00000000000 0e01000000000000 2200000000000000
10812 17 36 ktrgcm 3301000100000000 0100000000000000 0100000000000000
10704 17 36 ksqgtlctx ksqgtl: acquire TM-0001563e-00000000 mode=S flags=GLOBAL|XACT why="contention"
10704 17 36 ksqgtlctx ksqgtl: SUCCESS
10704 17 36 ksqrcli ksqrcl: release TM-0001563e-00000000 mode=S
10704 17 36 ksqrcli ksqrcl: SUCCESS
10812 17 36 ktrgcm 3b01000100000000 0000000000000000 5fea140000000000
10812 17 36 ktrgcm 0600000000000000 0f00000000000000 0a05000000000000
10812 17 36 ktrgcm bc06c00000000000 0e01000000000000 2400000000000000
10812 17 36 ktrgcm 3b01000100000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 3c01000100000000 0000000000000000 5fea140000000000
10812 17 36 ktrgcm 0600000000000000 0f00000000000000 0a05000000000000
10812 17 36 ktrgcm bc06c00000000000 0e01000000000000 2400000000000000
10812 17 36 ktrgcm 3c01000100000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 3d01000100000000 0000000000000000 5fea140000000000
10812 17 36 ktrgcm 0600000000000000 0f00000000000000 0a05000000000000
10812 17 36 ktrgcm bc06c00000000000 0e01000000000000 2400000000000000
10812 17 36 ktrgcm 3d01000100000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 3e01000100000000 0000000000000000 5fea140000000000
10812 17 36 ktrgcm 0600000000000000 0f00000000000000 0a05000000000000
10812 17 36 ktrgcm bc06c00000000000 0e01000000000000 2400000000000000
10812 17 36 ktrgcm 3e01000100000000 0000000000000000 0000000000000000
10812 17 36 ktrgcm 3f01000100000000 0000000000000000 5fea140000000000
10812 17 36 ktrgcm 0600000000000000 0f00000000000000 0a05000000000000
10812 17 36 ktrgcm bc06c00000000000 0e01000000000000 2400000000000000
10812 17 36 ktrgcm 3f01000100000000 0000000000000000 0000000000000000
10005 17 36 kslwtbctx KSL WAIT BEG [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=236 seq_num=243 snap_id=1
10005 17 36 kslwtectx KSL WAIT END [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=236 seq_num=243 snap_id=1
10005 17 36 kslwtectx KSL WAIT END wait times (usecs) - snap=4, exc=4, tot=4
10005 17 36 kslwtbctx KSL WAIT BEG [SQL*Net message from client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=237 seq_num=244 snap_id=1
```

可以发现除了语句执行时需要获取一次`S`锁之外，删多少行就要获取多少次`S`锁，从之前的锁兼容列表就可发现`S`锁和`SX（RX）`锁是不兼容的，而`SX（RX）`是insert update delete获取的锁模式，可以想象如果此时子表上有事务，或者S锁获得了尚未释放的时候，子表要进行事务获取`mode`为3的`SX（RX）`锁时，**session都会产生等待**。

看一下此时session获取的锁，记住这次结果，后面会有对比。

```sql
SQL> select *  from v$lock where type in('TM','TX');

ADDR             KADDR                   SID TY        ID1        ID2      LMODE    REQUEST      CTIME      BLOCK
---------------- ---------------- ---------- -- ---------- ---------- ---------- ---------- ---------- ----------
00000000ACE37CD0 00000000ACE37D48         17 TX     393231       1290          6          0        195          0
00007FD97FFE6520 00007FD97FFE6580         17 TM      87612          0          3          0        195          0
```

可见语句执行完，已不持有子表上的任何锁

下面来模拟一下等待，

```sql
sid:31
SQL> insert into child values(2,'12312');

1 row created.

SQL> select distinct sid from v$mystat;

       SID
----------
        31
        
sid:1169

SQL> update prim set a=1 where a=1;

SQL> select *  from v$lock where type in('TM','TX');

ADDR             KADDR                   SID TY        ID1        ID2      LMODE    REQUEST      CTIME      BLOCK
---------------- ---------------- ---------- -- ---------- ---------- ---------- ---------- ---------- ----------
00007FD97FFE54E8 00007FD97FFE5548         31 TM      87614          0          3          0        143          1
00000000ABE6BE80 00000000ABE6BEF8         31 TX     327713       1114          6          0        143          0
00007FD97FFE54E8 00007FD97FFE5548       1169 TM      87612          0          3          0         76          0
00007FD97FFE54E8 00007FD97FFE5548         31 TM      87612          0          3          0        143          0
00007FD97FFE54E8 00007FD97FFE5548       1169 TM      87614          0          0          4         76          0
```

此时查一下等待链

```sql
SQL>  --锁源头查找,带对象和sql以及event
SQL>  WITH sessions AS
  2   (SELECT /*+materialize*/
  3     sid,
  4     blocking_session,
  5     blocking_instance,
  6     row_wait_obj#,
  7     sql_id,
  8     inst_id,
  9     event
 10      FROM gv$session)
 11  SELECT LPAD(' ', 4 * (level - 1)) || s.inst_id || '.' || sid sid,
 12         object_name,
 13         substr(sql_text, 1, 40) sql_text,
 14         event
 15    FROM sessions s
 16    LEFT OUTER JOIN dba_objects d
 17      ON (object_id = row_wait_obj#)
 18    LEFT OUTER JOIN gv$sql q
 19      ON (s.sql_id = q.SQL_ID and s.inst_id = q.INST_ID)
 20   WHERE sid IN (SELECT blocking_session FROM sessions)
 21      OR blocking_session IS NOT NULL
 22  CONNECT BY PRIOR sid = blocking_session
 23   START WITH blocking_session IS NULL;

SID        OBJECT_NAM SQL_TEXT                                 EVENT
---------- ---------- ---------------------------------------- ------------------------------
1.31                                                           SQL*Net message from client
    1.1169 CHILD      update prim set a=1 where a=1            enq: TM - contention
```

从上面的分析我们知道，无论插入父表和子表，都会获取两张表上的`mode`为`3`的锁，而`mode`为`3`的锁和`mode`为`4`的锁是不兼容的，也就是说此时父表上连插入都无法进行


再开第三个session

```sql
sid:1167
SQL> insert into prim values(7,'dasd'); --hang住了

SQL> select *  from v$lock where type in('TM','TX');

ADDR             KADDR                SID TY        ID1        ID2      LMODE    REQUEST      CTIME      BLOCK
---------------- ---------------- ------- -- ---------- ---------- ---------- ---------- ---------- ----------
00007FD97D39F2A8 00007FD97D39F308      31 TM      87614          0          3          0        476          1
00007FD97D39F2A8 00007FD97D39F308    1167 TM      87614          0          0          3         85          0
00000000ABE6BE80 00000000ABE6BEF8      31 TX     327713       1114          6          0        476          0
00007FD97D39F2A8 00007FD97D39F308    1169 TM      87612          0          3          0        409          0
00007FD97D39F2A8 00007FD97D39F308      31 TM      87612          0          3          0        476          0
00007FD97D39F2A8 00007FD97D39F308    1169 TM      87614          0          0          4        409          0
00007FD97D39F2A8 00007FD97D39F308    1167 TM      87612          0          3          0         85          0    
    
SID                  OBJECT_NAME                    SQL_TEXT                                 EVENT
-------------------- ------------------------------ ---------------------------------------- ----------------------------------------
1.31                                                                                         SQL*Net message from client
    1.1169           CHILD                          update prim set a=1 where a=1            enq: TM - contention
        1.1167       CHILD                          insert into prim values(7,'dasd')        enq: TM - contention
```

 可见`1167`的session被阻塞了。

 ## 无索引，有cascade
 ```sql
 SQL> alter table child drop constraint FK_CHILD_CA;

Table altered.

SQL> alter table child add constraint FK_CHILD_CA foreign key (ca) references prim(a) on delete cascade;

Table altered.
```

有cascade的时候，仅在delete语句上有所区别，下面仅列出delete语句

```sql
SQL> delete from prim where a=2 or a=4;

2 rows deleted.

 10704      17         36 ksqgtlctx            ksqgtl: acquire TM-0001563c-00000000 mode=SX flags=GLOBAL|XACT why="contention"
 10704      17         36 ksqgtlctx            ksqgtl: SUCCESS
 10704      17         36 ksqgtlctx            ksqgtl: acquire TM-0001563e-00000000 mode=SSX flags=GLOBAL|XACT why="contention"
 10704      17         36 ksqgtlctx            ksqgtl: SUCCESS
 10704      17         36 ksqcnv               ksqcnv: convert TM-0001563e-00000000 from=SSX to=SX flags=
 10704      17         36 ksqcnv               ksqcnv: SUCCESS
 10813      17         36 ktubnd               ktubnd: Bind usn 4 nax 1 nbx 0 lng 0 par 0
 10813      17         36 ktubnd               ktubnd: Txn Bound xid: 4.20.852
 10704      17         36 ksqgtlctx            ksqgtl: acquire TX-00040014-00000354 mode=X flags=GLOBAL|XACT why="contention"
 10704      17         36 ksqgtlctx            ksqgtl: SUCCESS
 10704      17         36 ksqcnv               ksqcnv: convert TM-0001563e-00000000 from=SX to=SSX flags=
 10704      17         36 ksqcnv               ksqcnv: SUCCESS
 10704      17         36 ksqcnv               ksqcnv: convert TM-0001563e-00000000 from=SSX to=SX flags=
 10704      17         36 ksqcnv               ksqcnv: SUCCESS
 10812      17         36 ktrgcm               3b01000100000000 0000000000000000 bef2140000000000
 10812      17         36 ktrgcm               0400000000000000 1400000000000000 5403000000000000
 10812      17         36 ktrgcm               ef00c00000000000 2801000000000000 0a00000000000000
 10812      17         36 ktrgcm               3b01000100000000 0000000000000000 0000000000000000
 10812      17         36 ktrgcm               3c01000100000000 0000000000000000 bef2140000000000
 10812      17         36 ktrgcm               0400000000000000 1400000000000000 5403000000000000
 10812      17         36 ktrgcm               ef00c00000000000 2801000000000000 0a00000000000000
 10812      17         36 ktrgcm               3c01000100000000 0000000000000000 0000000000000000
 10812      17         36 ktrgcm               3d01000100000000 0000000000000000 bef2140000000000
 10812      17         36 ktrgcm               0400000000000000 1400000000000000 5403000000000000
 10812      17         36 ktrgcm               ef00c00000000000 2801000000000000 0a00000000000000
 10812      17         36 ktrgcm               3d01000100000000 0000000000000000 0000000000000000
 10812      17         36 ktrgcm               3e01000100000000 0000000000000000 bef2140000000000
 10812      17         36 ktrgcm               0400000000000000 1400000000000000 5403000000000000
 10812      17         36 ktrgcm               ef00c00000000000 2801000000000000 0a00000000000000
 10812      17         36 ktrgcm               3e01000100000000 0000000000000000 0000000000000000
 10812      17         36 ktrgcm               3f01000100000000 0000000000000000 bef2140000000000
 10812      17         36 ktrgcm               0400000000000000 1400000000000000 5403000000000000
 10812      17         36 ktrgcm               ef00c00000000000 2801000000000000 0a00000000000000
 10812      17         36 ktrgcm               3f01000100000000 0000000000000000 0000000000000000
 10812      17         36 ktrgcm               3301000100000000 0000000000000000 bef2140000000000
 10812      17         36 ktrgcm               0000000000000000 0000000000000000 0000000000000000
 10812      17         36 ktrgcm               0000000000000000 0000000000000000 0000000000000000
 10811      17         36 ktbgcl1              3301000100000000 0000000000000000 bef2140000000000 0100000000000000
 10811      17         36 ktbgcl1              3301000100000000 0000000000000000 bff2140000000000 e00fc76a177f0000
 10812      17         36 kturCRBackoutOneChg  0100000000000000 ef00c00000000000 2801000000000000 0a00000000000000
 10812      17         36 ktrgcm               3301000100000000 0100000000000000 0100000000000000
 10704      17         36 ksqcnv               ksqcnv: convert TM-0001563e-00000000 from=SX to=SSX flags=
 10704      17         36 ksqcnv               ksqcnv: SUCCESS
 10704      17         36 ksqcnv               ksqcnv: convert TM-0001563e-00000000 from=SSX to=SX flags=
 10704      17         36 ksqcnv               ksqcnv: SUCCESS
 10812      17         36 ktrgcm               3b01000100000000 0000000000000000 c0f2140000000000
 10812      17         36 ktrgcm               0400000000000000 1400000000000000 5403000000000000
 10812      17         36 ktrgcm               ef00c00000000000 2801000000000000 0e00000000000000
 10812      17         36 ktrgcm               3b01000100000000 0000000000000000 0000000000000000
 10812      17         36 ktrgcm               3c01000100000000 0000000000000000 c0f2140000000000
 10812      17         36 ktrgcm               0400000000000000 1400000000000000 5403000000000000
 10812      17         36 ktrgcm               ef00c00000000000 2801000000000000 0e00000000000000
 10812      17         36 ktrgcm               3c01000100000000 0000000000000000 0000000000000000
 10812      17         36 ktrgcm               3d01000100000000 0000000000000000 c0f2140000000000
 10812      17         36 ktrgcm               0400000000000000 1400000000000000 5403000000000000
 10812      17         36 ktrgcm               ef00c00000000000 2801000000000000 0e00000000000000
 10812      17         36 ktrgcm               3d01000100000000 0000000000000000 0000000000000000
 10812      17         36 ktrgcm               3e01000100000000 0000000000000000 c0f2140000000000
 10812      17         36 ktrgcm               0400000000000000 1400000000000000 5403000000000000
 10812      17         36 ktrgcm               ef00c00000000000 2801000000000000 0e00000000000000
 10812      17         36 ktrgcm               3e01000100000000 0000000000000000 0000000000000000
 10812      17         36 ktrgcm               3f01000100000000 0000000000000000 c0f2140000000000
 10812      17         36 ktrgcm               0400000000000000 1400000000000000 5403000000000000
 10812      17         36 ktrgcm               ef00c00000000000 2801000000000000 0e00000000000000
 10812      17         36 ktrgcm               3f01000100000000 0000000000000000 0000000000000000
 10005      17         36 kslwtbctx            KSL WAIT BEG [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=425 seq_num=432 snap_id=1
 10005      17         36 kslwtectx            KSL WAIT END [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=425 seq_num=432 snap_id=1
 10005      17         36 kslwtectx            KSL WAIT END wait times (usecs) - snap=4, exc=4, tot=4
 10005      17         36 kslwtbctx            KSL WAIT BEG [SQL*Net message from client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=426 seq_num=433 snap_id=1
 ```

 此时会申请一个mode为5的SSX锁，随后即转换为mode为3的SX锁，这也是在语句执行期间获取和转换的，并非事务期间，同样删除多少行就涉及到多少次获取转换，看一下此时锁获得情况

```sql
SQL> select *  from v$lock where type in('TM','TX');

ADDR             KADDR                SID TY        ID1        ID2      LMODE    REQUEST      CTIME      BLOCK
---------------- ---------------- ------- -- ---------- ---------- ---------- ---------- ---------- ----------
00007FD97FFE6520 00007FD97FFE6580      17 TM      87614          0          3          0        131          0
00000000ACE37CD0 00000000ACE37D48      17 TX     262164        852          6          0        131          0
00007FD97FFE6520 00007FD97FFE6580      17 TM      87612          0          3          0        131          0
```

是不是和没有cascade的时候不同了，这次最终会持有子表上的mode为3的锁，我们再深入的思考一点，SSX锁和SX锁是不兼容的，这样是否就意味着后进行的delete会被先进行的delete阻塞（不同session），好，现在就来模拟一下：

```sql

sid:1169

SQL> delete from prim where a=1;

0 rows deleted.

SQL> select *  from v$lock where type in('TM','TX');

ADDR             KADDR                SID TY        ID1        ID2      LMODE    REQUEST      CTIME      BLOCK
---------------- ---------------- ------- -- ---------- ---------- ---------- ---------- ---------- ----------
00007FD97FFE54E8 00007FD97FFE5548    1169 TM      87612          0          3          0         53          0
00007FD97FFE54E8 00007FD97FFE5548    1169 TM      87614          0          3          0         53          0

sid:1167

SQL> delete from prim where a=2; --session hang住了

--查询此刻锁的持有情况
SQL> select *  from v$lock where type in('TM','TX');

ADDR             KADDR                SID TY        ID1        ID2      LMODE    REQUEST      CTIME      BLOCK
---------------- ---------------- ------- -- ---------- ---------- ---------- ---------- ---------- ----------
00007FD97FFE6520 00007FD97FFE6580    1169 TM      87612          0          3          0         85          0
00007FD97FFE6520 00007FD97FFE6580    1167 TM      87612          0          3          0         11          0
00007FD97FFE6520 00007FD97FFE6580    1167 TM      87614          0          0          5         11          0
00007FD97FFE6520 00007FD97FFE6580    1169 TM      87614          0          3          0         85          1
```

可见1167在请求mode为5的锁，且已被阻塞

查看等待链

```sql
SID                  OBJECT_NAME                    SQL_TEXT                                 EVENT
-------------------- ------------------------------ ---------------------------------------- ----------------------------------------
1.1169                                                                                       SQL*Net message from client
    1.1167           CHILD                          delete from prim where a=2               enq: TM - contention
```

因为delete完毕会持有子表上的SX锁，而SX锁与S锁不兼容，所以delete父表的session也会阻塞update父表的session，因为update会去请求子表的S锁，而此时子表上有SX锁，类似于子表上有事务在进行，这里就不在论述了，徒占篇幅。

## 有索引，无cascade
我看到有资料说，`如果有索引时,对父表的操作,会级联加一个TM RS锁(level 2)到子表上`。但我在试验中并未看到，也许是版本差异，我也未去求证，有索引时insert与无索引时在获取锁方面没有区别，这里仅列出update和delete

创建索引：
```sql
SQL> alter table child drop constraint FK_CHILD_CA;

Table altered.

SQL> alter table child add constraint FK_CHILD_CA foreign key (ca) references prim(a);

Table altered.

SQL> create  index ind_child_ca on child(ca);

Index created.
```
update父表：
```sql
SQL> update prim set a=6 where a=6;

1 row updated.

 10704      17         36 ksqgtlctx            ksqgtl: acquire TM-0001563c-00000000 mode=SX flags=GLOBAL|XACT why="contention"
 10704      17         36 ksqgtlctx            ksqgtl: SUCCESS
 10704      17         36 ksqgtlctx            ksqgtl: acquire TM-0001563e-00000000 mode=SX flags=GLOBAL|XACT why="contention"
 10704      17         36 ksqgtlctx            ksqgtl: SUCCESS
 10811      17         36 ktbgcl1              2b01000100000000 0000000000000000 fadc140000000000 0200000000000000
 10811      17         36 ktbgcl1              2b01000100000000 0000000000000000 cdf9140000000000 10dfb76a177f0000
 10813      17         36 ktubnd               ktubnd: Bind usn 3 nax 1 nbx 0 lng 0 par 0
 10813      17         36 ktubnd               ktubnd: Txn Bound xid: 3.28.1117
 10704      17         36 ksqgtlctx            ksqgtl: acquire TX-0003001c-0000045d mode=X flags=GLOBAL|XACT why="contention"
 10704      17         36 ksqgtlctx            ksqgtl: SUCCESS
 10005      17         36 kslwtbctx            KSL WAIT BEG [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=471 seq_num=478 snap_id=1
 10005      17         36 kslwtectx            KSL WAIT END [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=471 seq_num=478 snap_id=1
 10005      17         36 kslwtectx            KSL WAIT END wait times (usecs) - snap=3, exc=3, tot=3
 10005      17         36 kslwtbctx            KSL WAIT BEG [SQL*Net message from client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=472 seq_num=479 snap_id=1
 ```

可见有了索引之后，不再需要在语句级别获取子表上的S锁了

delete父表：

```sql
SQL> delete from prim where a=9;

1 row deleted.

 10704      17         36 ksqgtlctx            ksqgtl: acquire TM-0001563c-00000000 mode=SX flags=GLOBAL|XACT why="contention"
 10704      17         36 ksqgtlctx            ksqgtl: SUCCESS
 10704      17         36 ksqgtlctx            ksqgtl: acquire TM-0001563e-00000000 mode=SX flags=GLOBAL|XACT why="contention"
 10704      17         36 ksqgtlctx            ksqgtl: SUCCESS
 10811      17         36 ktbgcl1              2c01000100000000 0000000000000000 66f1140000000000 0200000000000000
 10811      17         36 ktbgcl1              2c01000100000000 0000000000000000 58fa140000000000 78efb76a177f0000
 10813      17         36 ktubnd               ktubnd: Bind usn 4 nax 1 nbx 0 lng 0 par 0
 10813      17         36 ktubnd               ktubnd: Txn Bound xid: 4.18.854
 10704      17         36 ksqgtlctx            ksqgtl: acquire TX-00040012-00000356 mode=X flags=GLOBAL|XACT why="contention"
 10704      17         36 ksqgtlctx            ksqgtl: SUCCESS
 10811      17         36 ktbgcl1              3301000100000000 0000000000000000 c0f2140000000000 0200000000000000
 10811      17         36 ktbgcl1              3301000100000000 0000000000000000 58fa140000000000 90eec66a177f0000
 10005      17         36 kslwtbctx            KSL WAIT BEG [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=476 seq_num=483 snap_id=1
 10005      17         36 kslwtectx            KSL WAIT END [SQL*Net message to client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=476 seq_num=483 snap_id=1
 10005      17         36 kslwtectx            KSL WAIT END wait times (usecs) - snap=4, exc=4, tot=4
 10005      17         36 kslwtbctx            KSL WAIT BEG [SQL*Net message from client] 1650815232/0x62657100 1/0x1 0/0x0 wait_id=477 seq_num=484 snap_id=1
 ```

与update相同，都持有了子表上的SX锁，而SX与SX是相容的，所以不会再产生锁定问题

```sql
SQL> select *  from v$lock where type in('TM','TX');

ADDR             KADDR                SID TY        ID1        ID2      LMODE    REQUEST      CTIME      BLOCK
---------------- ---------------- ------- -- ---------- ---------- ---------- ---------- ---------- ----------
00007FD97FFE54E8 00007FD97FFE5548      17 TM      87614          0          3          0        182          0
00000000ABD4E7C0 00000000ABD4E838      17 TX     262162        854          6          0        182          0
00007FD97FFE54E8 00007FD97FFE5548      17 TM      87612          0          3          0        182          0
```

## 有索引，有cascade

`表现与无cascade时相同`

## 总结

- 外键无索引锁无cascade时，update/delete父表，会在语句级别级联一个mode为4的S锁到子表，其中delete多少行就会级联多少次
- 外键无索引有cascade时，update父表仍会在语句级别级联mode为4的S锁到子表，delete时会先获取mode为5的SSX锁，在将其转换成mode为3的SX锁，而且删除多少行就会涉及到多少次转换
- 外键有索引无cascade时，update/delete不会在语句级级联锁到子表，最终会持有父表和子表上的mode为3的SX锁（无索引时只有有cascade的delete时最终会持有子表上的SX锁）
- 外键有索引有cascade时，与无cascade表现相同