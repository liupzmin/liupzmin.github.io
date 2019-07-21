---
title: enq之TM-contention解决之道——外键无索引导致锁争用（上）
tags: oracle 
date: 2016-07-25 16:21
---

近日，开发负责人反映某生产环境业务处理缓慢，主要业务操作就是修改会员信息，登录查询后发现大量的session正在等待enq: TM - contention，且waiting的语句几乎都是update
session的即时信息没有保留，现在附上ash视图的一些统计信息，可以大概了解一下当时争用的场景

```sql
SQL> @ash_wait_chains.sql username||':'||program2||event2 session_type='FOREGROUND' sysdate-6/24 sysdate-5/24

-- Display ASH Wait Chain Signatures script v0.2 BETA by Tanel Poder ( http://blog.tanelpoder.com )

%This     SECONDS        AAS
------ ---------- ----------
WAIT_CHAIN
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  72%       20073        5.6
-> JSCHPROD:(JDBC Thin Client) ON CPU

   8%        2293         .6
-> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) ON CPU

   8%        2141         .6
-> JSCHPROD:(JDBC Thin Client) log file sync  -> SYS:(LGWR) log file parallel write

   7%        1879         .5
-> JSCHPROD:(JDBC Thin Client) log file sync  -> SYS:(LGWR) LGWR-LNS wait on channel

   2%         654         .2
-> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) ON CPU

   1%         288         .1
-> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) ON CPU

   1%         149          0
-> JSCHPROD:(JDBC Thin Client) log file sync  -> SYS:(LGWR) ON CPU

   0%         128          0
-> JSCHPROD:(JDBC Thin Client) enq: TM - contention

   0%         112          0
-> JSCHPROD:(JDBC Thin Client) log file sync

   0%          86          0
-> JSCHPROD:(JDBC Thin Client) db file scattered read

   0%          43          0
-> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) ON CPU

   0%          37          0
-> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) enq: TM - contention

   0%          25          0
-> JSCHPROD:(JDBC Thin Client) log file sync  -> SYS:(LGWR) LGWR wait on LNS

   0%          13          0
-> JSCHPROD:(plsqldev.exe) ON CPU

   0%          11          0
-> SYS:(plsqldev.exe) ON CPU

   0%          10          0
-> JSCHPROD:(JDBC Thin Client) SQL*Net more data from client

   0%           9          0
-> JSCHPROD:(JDBC Thin Client) db file sequential read

   0%           9          0
-> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) ON
CPU

   0%           6          0
-> JSCHPROD:(JDBC Thin Client) read by other session  -> JSCHPROD:(JDBC Thin Client) ON CPU

   0%           4          0
-> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) enq: TM - contention  -> JSCHPROD:(JDBC Thin Client) enq: TM - contention

   0%           3          0
-> JSCHPROD:(JDBC Thin Client) SQL*Net more data to client

   0%           3          0
-> JSCHPROD:(JDBC Thin Client) buffer busy waits [data block]

   0%           3          0
-> SYS:(oraagent.bin) Disk file operations I/O

   0%           3          0
-> JSCHPROD:(JDBC Thin Client) log file sync  -> SYS:(LGWR) LGWR wait for redo copy

   0%           2          0
-> JSCHPROD:(JDBC Thin Client) enq: TX - row lock contention

   0%           2          0
-> JSCHPROD:(JDBC Thin Client) log file sync  -> SYS:(LGWR) LGWR wait for redo copy  -> JSCHPROD:(JDBC Thin Client) ON CPU

   0%           2          0
-> JSCHPROD:(JDBC Thin Client) enq: TX - index contention  -> JSCHPROD:(JDBC Thin Client) ON CPU

   0%           1          0
-> JSCHPROD:(plsqldev.exe) log file sync  -> SYS:(LGWR) log file parallel write

   0%           1          0
-> SYS:(plsqldev.exe) Disk file operations I/O

   0%           1          0
-> JSCHPROD:(JDBC Thin Client) enq: TX - row lock contention  -> JSCHPROD:(JDBC Thin Client) ON CPU


30 rows selected.

```

可以看到，TM锁的争用很多，再看一份当时awr报告的top10

![](https://liupzminblog.files.wordpress.com/2016/12/qq20161218-02x.png)

虽然占DBTIME不多，但本来是很快的操作，短时间内给人的感觉就是业务处理缓慢
查一下当时等待事件的p1,p2,p3的值

```sql
select ash.SAMPLE_TIME,
       ash.EVENT,
       ash.SESSION_ID,
       ash.BLOCKING_SESSION,
       ash.P1TEXT,
       ash.P1,
       ash.P2TEXT,
       ash.p2,
       ash.p3text,
       ash.p3,
       ash.SESSION_STATE,
       ash.SQL_OPNAME,
       ash.SQL_ID
       --ash.*
  from v$active_session_history ash
 where ash.SAMPLE_TIME >
       to_date('20160425 10:00:00', 'yyyymmdd HH24:MI:SS')
   and ash.SAMPLE_TIME <
       to_date('20160425 12:10:00', 'yyyymmdd HH24:MI:SS')
   and ash.WAIT_CLASS <> 'Idle'
   and ash.EVENT like 'enq: TM - contention'
 order by sample_time desc;
```

下面是部分结果

```
enq: TM - contention	1828	2401	name|mode	1414332421	object #	110417	table/partition	0	WAITING	UPDATE	ak25v8q8p6fzd
enq: TM - contention	1873	2504	name|mode	1414332419	object #	110415	table/partition	0	WAITING	INSERT	7w0tma5up32wt
enq: TM - contention	2504	1828	name|mode	1414332421	object #	110415	table/partition	0	WAITING	UPDATE	ak25v8q8p6fzd
enq: TM - contention	772	4	name|mode	1414332421	object #	110417	table/partition	0	WAITING	UPDATE	ak25v8q8p6fzd
enq: TM - contention	1781	1828	name|mode	1414332419	object #	110415	table/partition	0	WAITING	INSERT	7w0tma5up32wt
enq: TM - contention	1828	772	name|mode	1414332421	object #	110415	table/partition	0	WAITING	UPDATE	ak25v8q8p6fzd
enq: TM - contention	2401	1828	name|mode	1414332419	object #	110415	table/partition	0	WAITING	INSERT	7w0tma5up32wt
enq: TM - contention	2504	772	name|mode	1414332421	object #	110415	table/partition	0	WAITING	UPDATE	ak25v8q8p6fzd
enq: TM - contention	4	772	name|mode	1414332419	object #	110415	table/partition	0	WAITING	INSERT	7w0tma5up32wt
enq: TM - contention	148	1781	name|mode	1414332420	object #	110428	table/partition	0	WAITING	UPDATE	9gd6xhd0xyhph
enq: TM - contention	772	148	name|mode	1414332421	object #	110415	table/partition	0	WAITING	UPDATE	ak25v8q8p6fzd
enq: TM - contention	1828	148	name|mode	1414332421	object #	110415	table/partition	0	WAITING	UPDATE	ak25v8q8p6fzd
enq: TM - contention	1873	772	name|mode	1414332419	object #	110415	table/partition	0	WAITING	INSERT	7w0tma5up32wt
```

可以看到p2的值为产生TM争用的对象id，经过查证，这些object均是session正在更新的表的子表，而且通过v$sql查看update语句均更改了主表的主键，问题到这里已经很明朗了，由于外键没加索引，导致了主表在更新主表主键或删除主表记录时对子表的锁定，而且这张主表被大量的子表引用，此时子表上也同时进行事务处理，所以造成了更新主表的session 不时hang住。

通过对所有子表的外键加索引，消除了争用，检测未加索引的外键语句：

```sql
SELECT TABLE_NAME,
       CONSTRAINT_NAME,
       CNAME1 || NVL2(CNAME2, ',' || CNAME2, NULL) ||
       NVL2(CNAME3, ',' || CNAME3, NULL) ||
       NVL2(CNAME4, ',' || CNAME4, NULL) ||
       NVL2(CNAME5, ',' || CNAME5, NULL) ||
       NVL2(CNAME6, ',' || CNAME6, NULL) ||
       NVL2(CNAME7, ',' || CNAME7, NULL) ||
       NVL2(CNAME8, ',' || CNAME8, NULL) COLUMNS
  FROM (SELECT B.TABLE_NAME,
               B.CONSTRAINT_NAME,
               MAX(DECODE(POSITION, 1, COLUMN_NAME, NULL)) CNAME1,
               MAX(DECODE(POSITION, 2, COLUMN_NAME, NULL)) CNAME2,
               MAX(DECODE(POSITION, 3, COLUMN_NAME, NULL)) CNAME3,
               MAX(DECODE(POSITION, 4, COLUMN_NAME, NULL)) CNAME4,
               MAX(DECODE(POSITION, 5, COLUMN_NAME, NULL)) CNAME5,
               MAX(DECODE(POSITION, 6, COLUMN_NAME, NULL)) CNAME6,
               MAX(DECODE(POSITION, 7, COLUMN_NAME, NULL)) CNAME7,
               MAX(DECODE(POSITION, 8, COLUMN_NAME, NULL)) CNAME8,
               COUNT(*) COL_CNT
          FROM (SELECT SUBSTR(TABLE_NAME, 1, 30) TABLE_NAME,
                       SUBSTR(CONSTRAINT_NAME, 1, 30) CONSTRAINT_NAME,
                       SUBSTR(COLUMN_NAME, 1, 30) COLUMN_NAME,
                       POSITION
                  FROM USER_CONS_COLUMNS) A,
               USER_CONSTRAINTS B
         WHERE A.CONSTRAINT_NAME = B.CONSTRAINT_NAME
           AND B.CONSTRAINT_TYPE = 'R'
         GROUP BY B.TABLE_NAME, B.CONSTRAINT_NAME) CONS
 WHERE COL_CNT > ALL
 (SELECT COUNT(*)
          FROM USER_IND_COLUMNS I
         WHERE I.TABLE_NAME = CONS.TABLE_NAME
           AND I.COLUMN_NAME IN (CNAME1, CNAME2, CNAME3, CNAME4, CNAME5,
                CNAME6, CNAME7, CNAME8)
           AND I.COLUMN_POSITION <= CONS.COL_CNT
         GROUP BY I.INDEX_NAME);
```

这是摘自TOM大师的语句，外键不加索引也是导致死锁的常见原因之一，因此对于主表经常进行更新删除操作的情况，外键一定要加索引。
至于外键未加索引是如何导致锁定的，以及为何加了索引后争用就消失了? 敬请关注enq: TM - contention解决之道——外键无索引导致锁争用 （下）
