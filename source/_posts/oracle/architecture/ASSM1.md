---
tags: oracle
date: 2015-09-25 14:51
categories: oracle
title: ASSM三级位图结构与高水位的探究（上）
---

#ASSM三级位图结构与高水位的探究
>三级位图

Oracle9i在本地表空管理(LMT)的基础上，对段空间管理也引入了位图管理（Segment Space Management Auto）来取代原来的freelist管理方式（Segment Space Management Manual）。
但是默认system和undo表空间仍然是MSSM的管理方式，本文主要探究ASSM管理方式下，段的三级位图结构和高水位推进的关系 先来看
ASSM管理方式下的三级位图结构图：

![](http://qiniu.liupzmin.com/%E4%B8%89%E7%BA%A7%E4%BD%8D%E5%9B%BE.jpg)
*注：此图来源于网络*
一个段被创建之后，段头其实是一个L3块，在我的试验中，一个1M固定区大小，block为8K的表空间，创建的第一个段第0个区，前两个块是L1（128,129号块），第三个是L2（130号块），第四个是L3（131号块），前128个块是数据文件头，本文不讨论。
当数据被插入的时候，oracle根据连接进来的session，做hash运算，随机选择一个L3（如果有多个的话），再随机选择一个L2，接下来在该L2下随机选择一个L1，再从L1中管理的block里面随机选择一个块将数据插入，不同的session经过hash之后，最后落到block时已经是很分散的了，不会产生很多和会话同时往一个block中插数据申请独占buffer pin，而造成大量buffer busy waits的情况，这就是ASSM号称支持大并发插入的原理所在，但是事实上真的如此么？我们来一探究竟！

>实验环境

```
OS:REDHAT6.5 X64
DB VERSION:11.2.0.4
```
>首先，创建一个1M区大小的表空间，并在上面创建一张表，此表创建了4个空间很大的字段，目的是要一行占满一个block，便于观察结果

```sql
SQL> select file#,NAME,BYTES/1024/1204 from v$datafile;

     FILE# NAME                                                                             BYTES/1024/1204
---------- -------------------------------------------------------------------------------- ---------------
         1 +DATA/min/datafile/system.256.854775095                                               637.873754
         2 +DATA/min/datafile/sysaux.257.854775097                                               484.784053
         3 +DATA/min/datafile/undotbs1.258.854775097                                             187.109635
         4 +DATA/min/datafile/users.259.854775097                                                103.122924

SQL> create tablespace lp datafile '+DATA/min/datafile/lp.dbf' size 2048M uniform size 1m;

Tablespace created.

SQL> select file#,NAME,BYTES/1024/1204 from v$datafile;

     FILE# NAME                                                                             BYTES/1024/1204
---------- -------------------------------------------------------------------------------- ---------------
         1 +DATA/min/datafile/system.256.854775095                                               637.873754
         2 +DATA/min/datafile/sysaux.257.854775097                                               484.784053
         3 +DATA/min/datafile/undotbs1.258.854775097                                             187.109635
         4 +DATA/min/datafile/users.259.854775097                                                103.122924
         5 +DATA/min/datafile/lp.dbf                                                              1741.8206

SQL> create table lp (id number,des1 char(2000),des2 char(2000),des3 char(2000),des4 char(500)) tablespace lp;

Table created.
```

>观察新建的表所占区

```sql
SQL> col SEGMENT_NAME for a30
SQL> col des1 for a1
SQL> col des2 for a1
SQL> col des3 for a1
SQL> col des4 for a1
SQL> set line 200
SQL> select SEGMENT_NAME,EXTENT_ID,FILE_ID,BLOCK_ID, BLOCKS from dba_extents where segment_name='LP';

SEGMENT_NAME                    EXTENT_ID    FILE_ID   BLOCK_ID     BLOCKS
------------------------------ ---------- ---------- ---------- ----------
LP                                      0          5        128        128
```

>此表是5号文件，拥有1个区，block从128号开始，接下来我们插入一条数据，并dump出段头进行观察

```sql
SQL> insert into lp values(1,'a','a','a','a');

1 row created.

SQL> commit;

Commit complete.

SQL> select segment_name ,HEADER_FILE,HEADER_BLOCK from dba_segments where segment_name='LP';

SEGMENT_NAME                   HEADER_FILE HEADER_BLOCK
------------------------------ ----------- ------------
LP                                       5          131


SQL> alter system checkpoint;

System altered.
```

>LP的段头是5号文件，131号块，进行dump

```sql
SQL> alter system dump datafile 5 block 131;

SQL> col value for a65
SQL> select *  from v$diag_info;

   INST_ID NAME                                                                             VALUE
---------- -------------------------------------------------------------------------------- -----------------------------------------------------------------
         1 Diag Enabled                                                                     TRUE
         1 ADR Base                                                                         /u01/app/oracle
         1 ADR Home                                                                         /u01/app/oracle/diag/rdbms/min/min
         1 Diag Trace                                                                       /u01/app/oracle/diag/rdbms/min/min/trace
         1 Diag Alert                                                                       /u01/app/oracle/diag/rdbms/min/min/alert
         1 Diag Incident                                                                    /u01/app/oracle/diag/rdbms/min/min/incident
         1 Diag Cdump                                                                       /u01/app/oracle/diag/rdbms/min/min/cdump
         1 Health Monitor                                                                   /u01/app/oracle/diag/rdbms/min/min/hm
         1 Default Trace File                                                               /u01/app/oracle/diag/rdbms/min/min/trace/min_ora_2835.trc
         1 Active Problem Count                                                             0
         1 Active Incident Count                                                            0

11 rows selected.
```

>观察trc文件min_ora_2835.trc

```
Block dump from disk:
buffer tsn: 8 rdba: 0x01400083 (5/131)
scn: 0x0000.00140d51 seq: 0x01 flg: 0x04 tail: 0x0d512301
frmt: 0x02 chkval: 0xfd8a type: 0x23=PAGETABLE SEGMENT HEADER
Hex dump of block: st=0, typ_found=1



Extent Control Header
  -----------------------------------------------------------------
  Extent Header:: spare1: 0      spare2: 0      #extents: 1      #blocks: 128   
                  last map  0x00000000  #maps: 0      offset: 2716  
      Highwater::  0x014000c0  ext#: 0      blk#: 64     ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 60    
  mapblk  0x00000000  offset: 0     
                   Unlocked
  --------------------------------------------------------
  Low HighWater Mark :
      Highwater::  0x01400084  ext#: 0      blk#: 4      ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 0     
  mapblk  0x00000000  offset: 0     
  Level 1 BMB for High HWM block: 0x01400080
  Level 1 BMB for Low HWM block: 0x01400080
  --------------------------------------------------------
  Segment Type: 1 nl2: 1      blksz: 8192   fbsz: 0      
  L2 Array start offset:  0x00001434
  First Level 3 BMB:  0x00000000
  L2 Hint for inserts:  0x01400082
  Last Level 1 BMB:  0x01400081
  Last Level II BMB:  0x01400082
  Last Level III BMB:  0x00000000
     Map Header:: next  0x00000000  #extents: 1    obj#: 87596  flag: 0x10000000
  Inc # 0
  Extent Map
  -----------------------------------------------------------------
   0x01400080  length: 128   

  Auxillary Map
  --------------------------------------------------------
   Extent 0     :  L1 dba:  0x01400080 Data dba:  0x01400084
  --------------------------------------------------------

Second Level Bitmap block DBAs
   --------------------------------------------------------
   DBA 1:   0x01400082

End dump data blocks tsn: 8 file#: 5 minblk 131 maxblk 131
```

>很容易发现这是一个PAGETABLE SEGMENT HEADER块，Extent Map中只有一个区，另外在尾部有一个指向二级位图块的地址0x01400082，这是数据块地址的二进制用十六进制来表示，我们来将其转化成直观一点的信息

```sql
SQL> select dbms_utility.data_block_address_file(to_number('01400082', 'xxxxxxxx')) file#,
  2 dbms_utility.data_block_address_block(to_number('01400082', 'xxxxxxxx')) block#
  3 from dual;

     FILE# BLOCK#
---------- ----------
         5 130
```

> 依样将5号文件130号块的内容dump出来

```
Block dump from disk:
buffer tsn: 8 rdba: 0x01400082 (5/130)
scn: 0x0000.00140a45 seq: 0x02 flg: 0x04 tail: 0x0a452102
frmt: 0x02 chkval: 0xd119 type: 0x21=SECOND LEVEL BITMAP BLOCK
Hex dump of block: st=0, typ_found=1


Dump of Second Level Bitmap Block
   number: 2       nfree: 2       ffree: 0      pdba:     0x01400083
   Inc #: 0 Objd: 87596
  opcode:0
 xid:
  L1 Ranges :
  --------------------------------------------------------
   0x01400080  Free: 5 Inst: 1
   0x01400081  Free: 5 Inst: 1

  --------------------------------------------------------
End dump data blocks tsn: 8 file#: 5 minblk 130 maxblk 130
```

>可以发现这是一个二级位图块，并从其中找到了2个L1块的地址，我们来看第一个L1

```
Block dump from disk:
buffer tsn: 8 rdba: 0x01400080 (5/128)
scn: 0x0000.00140d51 seq: 0x03 flg: 0x04 tail: 0x0d512003
frmt: 0x02 chkval: 0xe697 type: 0x20=FIRST LEVEL BITMAP BLOCK
Hex dump of block: st=0, typ_found=1

DBA Ranges :
  --------------------------------------------------------
   0x01400080  Length: 64     Offset: 0      

   0:Metadata   1:Metadata   2:Metadata   3:Metadata
   4:unformatted   5:unformatted   6:unformatted   7:unformatted
   8:unformatted   9:unformatted   10:unformatted   11:unformatted
   12:unformatted   13:unformatted   14:unformatted   15:unformatted
   16:75-100% free   17:75-100% free   18:75-100% free   19:75-100% free
   20:75-100% free   21:75-100% free   22:75-100% free   23:75-100% free
   24:75-100% free   25:75-100% free   26:75-100% free   27:75-100% free
   28:75-100% free   29:75-100% free   30:75-100% free   31:0-25% free
   32:unformatted   33:unformatted   34:unformatted   35:unformatted
   36:unformatted   37:unformatted   38:unformatted   39:unformatted
   40:unformatted   41:unformatted   42:unformatted   43:unformatted
   44:unformatted   45:unformatted   46:unformatted   47:unformatted
   48:unformatted   49:unformatted   50:unformatted   51:unformatted
   52:unformatted   53:unformatted   54:unformatted   55:unformatted
   56:unformatted   57:unformatted   58:unformatted   59:unformatted
   60:unformatted   61:unformatted   62:unformatted   63:unformatted
  --------------------------------------------------------
End dump data blocks tsn: 8 file#: 5 minblk 128 maxblk 128
```

>无疑这是一个L1块，里面挂了64个数据块，因为我之前插入了一行数据，oracle已经格式化了一部分，并向其中一个块插入了数据，但是没达到我们的效果，被插入的块还显示0-25%的空闲，我们要的是full的状态，说明我们构造的数据不够大，没关系，我们修改一下pctfree就可以了，另外我们可以发现这个区是1M，有2个L1，每个L1里有64个数据块，接下来修改pctfree

```sql
SQL> alter table lp pctfree 24;

Table altered.

SQL> select TABLE_NAME,PCT_FREE from dba_tables where table_name='LP';

TABLE_NAME                       PCT_FREE
------------------------------ ----------
LP                                     24

SQL> insert into lp values(2,'a','a','a','a');

1 row created.

SQL> commit;

Commit complete.
--手动触发检查点将内存中的块刷新到磁盘，本文每次dump之前都会做这个操作，以保证磁盘和内存的内容一致
SQL> alter system checkpoint;

System altered.
```

>我们再看128号块

```
DBA Ranges :
  --------------------------------------------------------
   0x01400080  Length: 64     Offset: 0      

   0:Metadata   1:Metadata   2:Metadata   3:Metadata
   4:unformatted   5:unformatted   6:unformatted   7:unformatted
   8:unformatted   9:unformatted   10:unformatted   11:unformatted
   12:unformatted   13:unformatted   14:unformatted   15:unformatted
   16:75-100% free   17:75-100% free   18:75-100% free   19:75-100% free
   20:75-100% free   21:75-100% free   22:75-100% free   23:75-100% free
   24:75-100% free   25:75-100% free   26:75-100% free   27:75-100% free
   28:75-100% free   29:75-100% free   30:75-100% free   31:0-25% free
   32:75-100% free   33:75-100% free   34:75-100% free   <font color='red'>35:FULL</font>
   36:75-100% free   37:75-100% free   38:75-100% free   39:75-100% free
   40:75-100% free   41:75-100% free   42:75-100% free   43:75-100% free
   44:75-100% free   45:75-100% free   46:75-100% free   47:75-100% free
   48:unformatted   49:unformatted   50:unformatted   51:unformatted
   52:unformatted   53:unformatted   54:unformatted   55:unformatted
   56:unformatted   57:unformatted   58:unformatted   59:unformatted
   60:unformatted   61:unformatted   62:unformatted   63:unformatted
  --------------------------------------------------------
End dump data blocks tsn: 8 file#: 5 minblk 128 maxblk 128
```

>已经是我们想的结果了，到这里我们再来看一下另外一个L1吧，dump129号块

```
DBA Ranges :
  --------------------------------------------------------
   0x014000c0  Length: 64     Offset: 0      

   0:unformatted   1:unformatted   2:unformatted   3:unformatted
   4:unformatted   5:unformatted   6:unformatted   7:unformatted
   8:unformatted   9:unformatted   10:unformatted   11:unformatted
   12:unformatted   13:unformatted   14:unformatted   15:unformatted
   16:unformatted   17:unformatted   18:unformatted   19:unformatted
   20:unformatted   21:unformatted   22:unformatted   23:unformatted
   24:unformatted   25:unformatted   26:unformatted   27:unformatted
   28:unformatted   29:unformatted   30:unformatted   31:unformatted
   32:unformatted   33:unformatted   34:unformatted   35:unformatted
   36:unformatted   37:unformatted   38:unformatted   39:unformatted
   40:unformatted   41:unformatted   42:unformatted   43:unformatted
   44:unformatted   45:unformatted   46:unformatted   47:unformatted
   48:unformatted   49:unformatted   50:unformatted   51:unformatted
   52:unformatted   53:unformatted   54:unformatted   55:unformatted
   56:unformatted   57:unformatted   58:unformatted   59:unformatted
   60:unformatted   61:unformatted   62:unformatted   63:unformatted
  --------------------------------------------------------
End dump data blocks tsn: 8 file#: 5 minblk 129 maxblk 129
```

>全是未格式化的块，试一试并发插入，这里不模拟真正的并发了，只是开不同的session来进行插入

```sql
SQL> insert into lp values(3,'a','a','a','a');

1 row created.

SQL> insert into lp values(4,'a','a','a','a');

1 row created.

SQL> insert into lp values(5,'a','a','a','a');

1 row created.

SQL> insert into lp values(6,'a','a','a','a');

1 row created.

SQL> insert into lp values(7,'a','a','a','a');

1 row created.

--单独session5条

SQL> select dbms_rowid.ROWID_RELATIVE_FNO(rowid),dbms_rowid.ROWID_BLOCK_NUMBER(rowid) from lp;

DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID)
------------------------------------ ------------------------------------
                                   5                                  158
                                   5                                  159
                                   5                                  163
                                   5                                  167
                                   5                                  171
                                   5                                  172
                                   5                                  174
                                   5                                  175
                                   5                                  179
                                   5                                  183
                                   5                                  187
                                   5                                  191

12 rows selected.
```

>一个单独的session里插入了5条，另外5个单独的session里各插一条，发现规律了么，插入的块只在132和192之间，我们分别看看两个L1

```
SQL> alter system checkpoint;

System altered.


  DBA Ranges :
  --------------------------------------------------------
   0x01400080  Length: 64     Offset: 0      

   0:Metadata   1:Metadata   2:Metadata   3:Metadata
   4:unformatted   5:unformatted   6:unformatted   7:unformatted
   8:unformatted   9:unformatted   10:unformatted   11:unformatted
   12:unformatted   13:unformatted   14:unformatted   15:unformatted
   16:75-100% free   17:75-100% free   18:75-100% free   19:75-100% free
   20:75-100% free   21:75-100% free   22:75-100% free   23:75-100% free
   24:75-100% free   25:75-100% free   26:75-100% free   27:75-100% free
   28:75-100% free   29:75-100% free   30:FULL   31:0-25% free
   32:75-100% free   33:75-100% free   34:75-100% free   35:FULL
   36:75-100% free   37:75-100% free   38:75-100% free   39:FULL
   40:75-100% free   41:75-100% free   42:75-100% free   43:FULL
   44:FULL   45:75-100% free   46:FULL   47:FULL
   48:75-100% free   49:75-100% free   50:75-100% free   51:FULL
   52:75-100% free   53:75-100% free   54:75-100% free   55:FULL
   56:75-100% free   57:75-100% free   58:75-100% free   59:FULL
   60:75-100% free   61:75-100% free   62:75-100% free   63:FULL
  --------------------------------------------------------
End dump data blocks tsn: 8 file#: 5 minblk 128 maxblk 128

********************************************************************
看一下第二个L1

DBA Ranges :
  --------------------------------------------------------
   0x014000c0  Length: 64     Offset: 0      

   0:unformatted   1:unformatted   2:unformatted   3:unformatted
   4:unformatted   5:unformatted   6:unformatted   7:unformatted
   8:unformatted   9:unformatted   10:unformatted   11:unformatted
   12:unformatted   13:unformatted   14:unformatted   15:unformatted
   16:unformatted   17:unformatted   18:unformatted   19:unformatted
   20:unformatted   21:unformatted   22:unformatted   23:unformatted
   24:unformatted   25:unformatted   26:unformatted   27:unformatted
   28:unformatted   29:unformatted   30:unformatted   31:unformatted
   32:unformatted   33:unformatted   34:unformatted   35:unformatted
   36:unformatted   37:unformatted   38:unformatted   39:unformatted
   40:unformatted   41:unformatted   42:unformatted   43:unformatted
   44:unformatted   45:unformatted   46:unformatted   47:unformatted
   48:unformatted   49:unformatted   50:unformatted   51:unformatted
   52:unformatted   53:unformatted   54:unformatted   55:unformatted
   56:unformatted   57:unformatted   58:unformatted   59:unformatted
   60:unformatted   61:unformatted   62:unformatted   63:unformatted
  --------------------------------------------------------
End dump data blocks tsn: 8 file#: 5 minblk 129 maxblk 129
```

>这说明只在第一个L1里面随机，那为什么L2下挂了2个L1，我插入了10条却没有一条随机到第二个L1块呢？答案是高水位，这是vage大师已经论证过的了，偶在这里仅为见证一下~~，好了，还记得L3里的高水位信息么？


```
Extent Control Header
  -----------------------------------------------------------------
  Extent Header:: spare1: 0      spare2: 0      #extents: 101    #blocks: 12928
                  last map  0x00000000  #maps: 0      offset: 2716  
      Highwater::  0x014000c0  ext#: 0      blk#: 64     ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 60    
  mapblk  0x00000000  offset: 0
```

>Highwater::  0x014000c0是哪个块呢？

```sql
  SQL> select dbms_utility.data_block_address_file(to_number('014000c0', 'xxxxxxxx')) file#,
  2 dbms_utility.data_block_address_block(to_number('014000c0', 'xxxxxxxx')) block#
  3 from dual;

     FILE# BLOCK#
---------- ----------
         5 192
```

 >这是第二个L1管理的第一个块，可见数据的插入只在高水位之下进行（直接路径插入除外），此时的状态如下：


![](http://qiniu.liupzmin.com/0%E4%B8%AAextent.png)

>那么L1里面只会有64个块么？高水位永远都会指向L1的最后一个块么？我们继续剖析！
首先给LP表多分配一些区

```sql
SQL> alter table lp allocate extent(size 100m);

Table altered.

SQL> select SEGMENT_NAME,EXTENT_ID,FILE_ID,BLOCK_ID, BLOCKS from dba_extents where segment_name='LP';

SEGMENT_NAME                    EXTENT_ID    FILE_ID   BLOCK_ID     BLOCKS
------------------------------ ---------- ---------- ---------- ----------
LP                                      0          5        128        128
LP                                      1          5        256        128
LP                                      2          5        384        128
LP                                      3          5        512        128
LP                                      4          5        640        128
LP                                      5          5        768        128
LP                                      6          5        896        128
LP                                      7          5       1024        128
LP                                      8          5       1152        128
LP                                      9          5       1280        128
LP                                     10          5       1408        128
LP                                     11          5       1536        128
LP                                     12          5       1664        128
LP                                     13          5       1792        128
LP                                     14          5       1920        128
LP                                     15          5       2048        128
LP                                     16          5       2176        128
LP                                     17          5       2304        128
LP                                     18          5       2432        128
LP                                     19          5       2560        128
LP                                     20          5       2688        128
LP                                     21          5       2816        128
LP                                     22          5       2944        128
LP                                     23          5       3072        128
LP                                     24          5       3200        128
LP                                     25          5       3328        128
LP                                     26          5       3456        128
LP                                     27          5       3584        128
LP                                     28          5       3712        128
LP                                     29          5       3840        128
LP                                     30          5       3968        128
LP                                     31          5       4096        128
LP                                     32          5       4224        128
LP                                     33          5       4352        128
LP                                     34          5       4480        128
LP                                     35          5       4608        128
LP                                     36          5       4736        128
LP                                     37          5       4864        128
LP                                     38          5       4992        128
LP                                     39          5       5120        128
LP                                     40          5       5248        128
LP                                     41          5       5376        128
LP                                     42          5       5504        128
LP                                     43          5       5632        128
LP                                     44          5       5760        128
LP                                     45          5       5888        128
LP                                     46          5       6016        128
LP                                     47          5       6144        128
LP                                     48          5       6272        128
LP                                     49          5       6400        128
LP                                     50          5       6528        128
LP                                     51          5       6656        128
LP                                     52          5       6784        128
LP                                     53          5       6912        128
LP                                     54          5       7040        128
LP                                     55          5       7168        128
LP                                     56          5       7296        128
LP                                     57          5       7424        128
LP                                     58          5       7552        128
LP                                     59          5       7680        128
LP                                     60          5       7808        128
LP                                     61          5       7936        128
LP                                     62          5       8064        128
LP                                     63          5       8192        128
LP                                     64          5       8320        128
LP                                     65          5       8448        128
LP                                     66          5       8576        128
LP                                     67          5       8704        128
LP                                     68          5       8832        128
LP                                     69          5       8960        128
LP                                     70          5       9088        128
LP                                     71          5       9216        128
LP                                     72          5       9344        128
LP                                     73          5       9472        128
LP                                     74          5       9600        128
LP                                     75          5       9728        128
LP                                     76          5       9856        128
LP                                     77          5       9984        128
LP                                     78          5      10112        128
LP                                     79          5      10240        128
LP                                     80          5      10368        128
LP                                     81          5      10496        128
LP                                     82          5      10624        128
LP                                     83          5      10752        128
LP                                     84          5      10880        128
LP                                     85          5      11008        128
LP                                     86          5      11136        128
LP                                     87          5      11264        128
LP                                     88          5      11392        128
LP                                     89          5      11520        128
LP                                     90          5      11648        128
LP                                     91          5      11776        128
LP                                     92          5      11904        128
LP                                     93          5      12032        128
LP                                     94          5      12160        128
LP                                     95          5      12288        128
LP                                     96          5      12416        128
LP                                     97          5      12544        128
LP                                     98          5      12672        128
LP                                     99          5      12800        128
LP                                    100          5      12928        128

101 rows selected.
```

>我分配了100个区，加上原来的一共101个，现在我们看看L3段头里面的内容

```
Extent Control Header
  -----------------------------------------------------------------
  Extent Header:: spare1: 0      spare2: 0      #extents: 101    #blocks: 12928
                  last map  0x00000000  #maps: 0      offset: 2716  
      Highwater::  0x014000c0  ext#: 0      blk#: 64     ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 60    
  mapblk  0x00000000  offset: 0     


Extent Map
  -----------------------------------------------------------------
   0x01400080  length: 128   
   0x01400100  length: 128   
   0x01400180  length: 128   
   0x01400200  length: 128   
   0x01400280  length: 128   
   0x01400300  length: 128   
   0x01400380  length: 128   
   0x01400400  length: 128   
   0x01400480  length: 128   
   0x01400500  length: 128   
   0x01400580  length: 128   
   0x01400600  length: 128   
   0x01400680  length: 128   
   0x01400700  length: 128   
   0x01400780  length: 128   
   0x01400800  length: 128   
   0x01400880  length: 128   
   0x01400900  length: 128   
   0x01400980  length: 128   
   0x01400a00  length: 128   
   0x01400a80  length: 128   
   0x01400b00  length: 128   
   0x01400b80  length: 128   
   0x01400c00  length: 128   
   0x01400c80  length: 128   
   0x01400d00  length: 128   
   0x01400d80  length: 128   
   0x01400e00  length: 128   
   0x01400e80  length: 128   
   0x01400f00  length: 128   
   0x01400f80  length: 128   
   0x01401000  length: 128   
   0x01401080  length: 128   
   0x01401100  length: 128   
   0x01401180  length: 128   
   0x01401200  length: 128   
   0x01401280  length: 128   
   0x01401300  length: 128   
   0x01401380  length: 128   
   0x01401400  length: 128   
   0x01401480  length: 128   
   0x01401500  length: 128   
   0x01401580  length: 128   
   0x01401600  length: 128   
   0x01401680  length: 128   
   0x01401700  length: 128   
   0x01401780  length: 128   
   0x01401800  length: 128   
   0x01401880  length: 128   
   0x01401900  length: 128   
   0x01401980  length: 128   
   0x01401a00  length: 128   
   0x01401a80  length: 128   
   0x01401b00  length: 128   
   0x01401b80  length: 128   
   0x01401c00  length: 128   
   0x01401c80  length: 128   
   0x01401d00  length: 128   
   0x01401d80  length: 128   
   0x01401e00  length: 128   
   0x01401e80  length: 128   
   0x01401f00  length: 128   
   0x01401f80  length: 128   
   0x01402000  length: 128   
   0x01402080  length: 128   
   0x01402100  length: 128   
   0x01402180  length: 128   
   0x01402200  length: 128   
   0x01402280  length: 128   
   0x01402300  length: 128   
   0x01402380  length: 128   
   0x01402400  length: 128   
   0x01402480  length: 128   
   0x01402500  length: 128   
   0x01402580  length: 128   
   0x01402600  length: 128   
   0x01402680  length: 128   
   0x01402700  length: 128   
   0x01402780  length: 128   
   0x01402800  length: 128   
   0x01402880  length: 128   
   0x01402900  length: 128   
   0x01402980  length: 128   
   0x01402a00  length: 128   
   0x01402a80  length: 128   
   0x01402b00  length: 128   
   0x01402b80  length: 128   
   0x01402c00  length: 128   
   0x01402c80  length: 128   
   0x01402d00  length: 128   
   0x01402d80  length: 128   
   0x01402e00  length: 128   
   0x01402e80  length: 128   
   0x01402f00  length: 128   
   0x01402f80  length: 128   
   0x01403000  length: 128   
   0x01403080  length: 128   
   0x01403100  length: 128   
   0x01403180  length: 128   
   0x01403200  length: 128   
   0x01403280  length: 128   

  Auxillary Map
  --------------------------------------------------------
   Extent 0     :  L1 dba:  0x01400080 Data dba:  0x01400084
   Extent 1     :  L1 dba:  0x01400100 Data dba:  0x01400102
   Extent 2     :  L1 dba:  0x01400180 Data dba:  0x01400182
   Extent 3     :  L1 dba:  0x01400200 Data dba:  0x01400202
   Extent 4     :  L1 dba:  0x01400280 Data dba:  0x01400282
   Extent 5     :  L1 dba:  0x01400300 Data dba:  0x01400302
   Extent 6     :  L1 dba:  0x01400380 Data dba:  0x01400382
   Extent 7     :  L1 dba:  0x01400400 Data dba:  0x01400402
   Extent 8     :  L1 dba:  0x01400480 Data dba:  0x01400482
   Extent 9     :  L1 dba:  0x01400500 Data dba:  0x01400502
   Extent 10    :  L1 dba:  0x01400580 Data dba:  0x01400582
   Extent 11    :  L1 dba:  0x01400600 Data dba:  0x01400602
   Extent 12    :  L1 dba:  0x01400680 Data dba:  0x01400682
   Extent 13    :  L1 dba:  0x01400700 Data dba:  0x01400702
   Extent 14    :  L1 dba:  0x01400780 Data dba:  0x01400782
   Extent 15    :  L1 dba:  0x01400800 Data dba:  0x01400802
   Extent 16    :  L1 dba:  0x01400880 Data dba:  0x01400882
   Extent 17    :  L1 dba:  0x01400900 Data dba:  0x01400902
   Extent 18    :  L1 dba:  0x01400980 Data dba:  0x01400982
   Extent 19    :  L1 dba:  0x01400a00 Data dba:  0x01400a02
   Extent 20    :  L1 dba:  0x01400a80 Data dba:  0x01400a82
   Extent 21    :  L1 dba:  0x01400b00 Data dba:  0x01400b02
   Extent 22    :  L1 dba:  0x01400b80 Data dba:  0x01400b82
   Extent 23    :  L1 dba:  0x01400c00 Data dba:  0x01400c02
   Extent 24    :  L1 dba:  0x01400c80 Data dba:  0x01400c82
   Extent 25    :  L1 dba:  0x01400d00 Data dba:  0x01400d02
   Extent 26    :  L1 dba:  0x01400d80 Data dba:  0x01400d82
   Extent 27    :  L1 dba:  0x01400e00 Data dba:  0x01400e02
   Extent 28    :  L1 dba:  0x01400e80 Data dba:  0x01400e82
   Extent 29    :  L1 dba:  0x01400f00 Data dba:  0x01400f02
   Extent 30    :  L1 dba:  0x01400f80 Data dba:  0x01400f82
   Extent 31    :  L1 dba:  0x01401000 Data dba:  0x01401002
   Extent 32    :  L1 dba:  0x01401080 Data dba:  0x01401082
   Extent 33    :  L1 dba:  0x01401100 Data dba:  0x01401102
   Extent 34    :  L1 dba:  0x01401180 Data dba:  0x01401182
   Extent 35    :  L1 dba:  0x01401200 Data dba:  0x01401202
   Extent 36    :  L1 dba:  0x01401280 Data dba:  0x01401282
   Extent 37    :  L1 dba:  0x01401300 Data dba:  0x01401302
   Extent 38    :  L1 dba:  0x01401380 Data dba:  0x01401382
   Extent 39    :  L1 dba:  0x01401400 Data dba:  0x01401402
   Extent 40    :  L1 dba:  0x01401480 Data dba:  0x01401482
   Extent 41    :  L1 dba:  0x01401500 Data dba:  0x01401502
   Extent 42    :  L1 dba:  0x01401580 Data dba:  0x01401582
   Extent 43    :  L1 dba:  0x01401600 Data dba:  0x01401602
   Extent 44    :  L1 dba:  0x01401680 Data dba:  0x01401682
   Extent 45    :  L1 dba:  0x01401700 Data dba:  0x01401702
   Extent 46    :  L1 dba:  0x01401780 Data dba:  0x01401782
   Extent 47    :  L1 dba:  0x01401800 Data dba:  0x01401802
   Extent 48    :  L1 dba:  0x01401880 Data dba:  0x01401882
   Extent 49    :  L1 dba:  0x01401900 Data dba:  0x01401902
   Extent 50    :  L1 dba:  0x01401980 Data dba:  0x01401982
   Extent 51    :  L1 dba:  0x01401a00 Data dba:  0x01401a02
   Extent 52    :  L1 dba:  0x01401a80 Data dba:  0x01401a82
   Extent 53    :  L1 dba:  0x01401b00 Data dba:  0x01401b02
   Extent 54    :  L1 dba:  0x01401b80 Data dba:  0x01401b82
   Extent 55    :  L1 dba:  0x01401c00 Data dba:  0x01401c02
   Extent 56    :  L1 dba:  0x01401c80 Data dba:  0x01401c82
   Extent 57    :  L1 dba:  0x01401d00 Data dba:  0x01401d02
   Extent 58    :  L1 dba:  0x01401d80 Data dba:  0x01401d82
   Extent 59    :  L1 dba:  0x01401e00 Data dba:  0x01401e02
   Extent 60    :  L1 dba:  0x01401e80 Data dba:  0x01401e82
   Extent 61    :  L1 dba:  0x01401f00 Data dba:  0x01401f02
   Extent 62    :  L1 dba:  0x01401f80 Data dba:  0x01401f82
   Extent 63    :  L1 dba:  0x01402000 Data dba:  0x01402001
   Extent 64    :  L1 dba:  0x01402000 Data dba:  0x01402080
   Extent 65    :  L1 dba:  0x01402100 Data dba:  0x01402101
   Extent 66    :  L1 dba:  0x01402100 Data dba:  0x01402180
   Extent 67    :  L1 dba:  0x01402200 Data dba:  0x01402201
   Extent 68    :  L1 dba:  0x01402200 Data dba:  0x01402280
   Extent 69    :  L1 dba:  0x01402300 Data dba:  0x01402301
   Extent 70    :  L1 dba:  0x01402300 Data dba:  0x01402380
   Extent 71    :  L1 dba:  0x01402400 Data dba:  0x01402401
   Extent 72    :  L1 dba:  0x01402400 Data dba:  0x01402480
   Extent 73    :  L1 dba:  0x01402500 Data dba:  0x01402501
   Extent 74    :  L1 dba:  0x01402500 Data dba:  0x01402580
   Extent 75    :  L1 dba:  0x01402600 Data dba:  0x01402601
   Extent 76    :  L1 dba:  0x01402600 Data dba:  0x01402680
   Extent 77    :  L1 dba:  0x01402700 Data dba:  0x01402701
   Extent 78    :  L1 dba:  0x01402700 Data dba:  0x01402780
   Extent 79    :  L1 dba:  0x01402800 Data dba:  0x01402801
   Extent 80    :  L1 dba:  0x01402800 Data dba:  0x01402880
   Extent 81    :  L1 dba:  0x01402900 Data dba:  0x01402901
   Extent 82    :  L1 dba:  0x01402900 Data dba:  0x01402980
   Extent 83    :  L1 dba:  0x01402a00 Data dba:  0x01402a01
   Extent 84    :  L1 dba:  0x01402a00 Data dba:  0x01402a80
   Extent 85    :  L1 dba:  0x01402b00 Data dba:  0x01402b01
   Extent 86    :  L1 dba:  0x01402b00 Data dba:  0x01402b80
   Extent 87    :  L1 dba:  0x01402c00 Data dba:  0x01402c01
   Extent 88    :  L1 dba:  0x01402c00 Data dba:  0x01402c80
   Extent 89    :  L1 dba:  0x01402d00 Data dba:  0x01402d01
   Extent 90    :  L1 dba:  0x01402d00 Data dba:  0x01402d80
   Extent 91    :  L1 dba:  0x01402e00 Data dba:  0x01402e01
   Extent 92    :  L1 dba:  0x01402e00 Data dba:  0x01402e80
   Extent 93    :  L1 dba:  0x01402f00 Data dba:  0x01402f01
   Extent 94    :  L1 dba:  0x01402f00 Data dba:  0x01402f80
   Extent 95    :  L1 dba:  0x01403000 Data dba:  0x01403001
   Extent 96    :  L1 dba:  0x01403000 Data dba:  0x01403080
   Extent 97    :  L1 dba:  0x01403100 Data dba:  0x01403101
   Extent 98    :  L1 dba:  0x01403100 Data dba:  0x01403180
   Extent 99    :  L1 dba:  0x01403200 Data dba:  0x01403201
   Extent 100   :  L1 dba:  0x01403200 Data dba:  0x01403280
  --------------------------------------------------------

   Second Level Bitmap block DBAs
   --------------------------------------------------------
   DBA 1:   0x01400082

End dump data blocks tsn: 8 file#: 5 minblk 131 maxblk 131   
```

>这里我们主要看Auxillary Map，里面记录了每个区所属的L1，不难发现从第64个区开始，64、65两个区的L1相同，这说明什么，说明L1下面挂了整整两个区的数据块，验证一下，将0x01402000块dump出来之后的内容：

```sql
  01402000  十进制 》20979712
  SQL> select dbms_utility.data_block_address_file(20979712) Rfile#,dbms_utility.data_block_address_block(20979712) "Block#" from dual;

    RFILE#     Block#
---------- ----------
         5       8192
```

```
          DBA Ranges :
  --------------------------------------------------------
   0x01402000  Length: 128    Offset: 0      
   0x01402080  Length: 128    Offset: 128    

   0:Metadata   1:unformatted   2:unformatted   3:unformatted
   4:unformatted   5:unformatted   6:unformatted   7:unformatted
   8:unformatted   9:unformatted   10:unformatted   11:unformatted
   12:unformatted   13:unformatted   14:unformatted   15:unformatted
   16:unformatted   17:unformatted   18:unformatted   19:unformatted
   20:unformatted   21:unformatted   22:unformatted   23:unformatted
   24:unformatted   25:unformatted   26:unformatted   27:unformatted
   28:unformatted   29:unformatted   30:unformatted   31:unformatted
   32:unformatted   33:unformatted   34:unformatted   35:unformatted
   36:unformatted   37:unformatted   38:unformatted   39:unformatted
   40:unformatted   41:unformatted   42:unformatted   43:unformatted
   44:unformatted   45:unformatted   46:unformatted   47:unformatted
   48:unformatted   49:unformatted   50:unformatted   51:unformatted
   52:unformatted   53:unformatted   54:unformatted   55:unformatted
   56:unformatted   57:unformatted   58:unformatted   59:unformatted
   60:unformatted   61:unformatted   62:unformatted   63:unformatted
   64:unformatted   65:unformatted   66:unformatted   67:unformatted
   68:unformatted   69:unformatted   70:unformatted   71:unformatted
   72:unformatted   73:unformatted   74:unformatted   75:unformatted
   76:unformatted   77:unformatted   78:unformatted   79:unformatted
   80:unformatted   81:unformatted   82:unformatted   83:unformatted
   84:unformatted   85:unformatted   86:unformatted   87:unformatted
   88:unformatted   89:unformatted   90:unformatted   91:unformatted
   92:unformatted   93:unformatted   94:unformatted   95:unformatted
   96:unformatted   97:unformatted   98:unformatted   99:unformatted
   100:unformatted   101:unformatted   102:unformatted   103:unformatted
   104:unformatted   105:unformatted   106:unformatted   107:unformatted
   108:unformatted   109:unformatted   110:unformatted   111:unformatted
   112:unformatted   113:unformatted   114:unformatted   115:unformatted
   116:unformatted   117:unformatted   118:unformatted   119:unformatted
   120:unformatted   121:unformatted   122:unformatted   123:unformatted
   124:unformatted   125:unformatted   126:unformatted   127:unformatted
   128:unformatted   129:unformatted   130:unformatted   131:unformatted
   132:unformatted   133:unformatted   134:unformatted   135:unformatted
   136:unformatted   137:unformatted   138:unformatted   139:unformatted
   140:unformatted   141:unformatted   142:unformatted   143:unformatted
   144:unformatted   145:unformatted   146:unformatted   147:unformatted
   148:unformatted   149:unformatted   150:unformatted   151:unformatted
   152:unformatted   153:unformatted   154:unformatted   155:unformatted
   156:unformatted   157:unformatted   158:unformatted   159:unformatted
   160:unformatted   161:unformatted   162:unformatted   163:unformatted
   164:unformatted   165:unformatted   166:unformatted   167:unformatted
   168:unformatted   169:unformatted   170:unformatted   171:unformatted
   172:unformatted   173:unformatted   174:unformatted   175:unformatted
   176:unformatted   177:unformatted   178:unformatted   179:unformatted
   180:unformatted   181:unformatted   182:unformatted   183:unformatted
   184:unformatted   185:unformatted   186:unformatted   187:unformatted
   188:unformatted   189:unformatted   190:unformatted   191:unformatted
   192:unformatted   193:unformatted   194:unformatted   195:unformatted
   196:unformatted   197:unformatted   198:unformatted   199:unformatted
   200:unformatted   201:unformatted   202:unformatted   203:unformatted
   204:unformatted   205:unformatted   206:unformatted   207:unformatted
   208:unformatted   209:unformatted   210:unformatted   211:unformatted
   212:unformatted   213:unformatted   214:unformatted   215:unformatted
   216:unformatted   217:unformatted   218:unformatted   219:unformatted
   220:unformatted   221:unformatted   222:unformatted   223:unformatted
   224:unformatted   225:unformatted   226:unformatted   227:unformatted
   228:unformatted   229:unformatted   230:unformatted   231:unformatted
   232:unformatted   233:unformatted   234:unformatted   235:unformatted
   236:unformatted   237:unformatted   238:unformatted   239:unformatted
   240:unformatted   241:unformatted   242:unformatted   243:unformatted
   244:unformatted   245:unformatted   246:unformatted   247:unformatted
   248:unformatted   249:unformatted   250:unformatted   251:unformatted
   252:unformatted   253:unformatted   254:unformatted   255:unformatted
  --------------------------------------------------------
End dump data blocks tsn: 8 file#: 5 minblk 8192 maxblk 8192
```

>256个块，2M 2个区，可见L1中块的数量会随着段大小的改变而做调整的，可以增大到1024个甚至更多，我在这里不去再做验证，将重点放在高水位上，根据我们之前观察到的现象，高水位在第二个L1的地一个块，也可说第一个L1的末端，那么现在的L1里有256个块，高水位是不是也会推进到这个L1的末端呢？很容易进行验证，将数据插满到这个L1的前几个块再去观察段头高水位的变化就行了，那么插多少条呢？再来计算一下，当前的L1是8192号块，前面一共63个区，每个区里面两个L1，加上第一个区里面的一个L2和L3，我们需要插满8192-63*2-2=8064

```sql
declare
i number;
begin
for i in 1..8064 loop
insert into lp values(i,'a','a','a','a');
end loop;
end;
/
```

>观察62号区的第二个L1，第一个L1的地址是0x01401f80，那么第二个就是01401f81，5号文件8065号块，dump出来的结果

```
DBA Ranges :
  --------------------------------------------------------
   0x01401fc0  Length: 64     Offset: 0      

   0:FULL   1:FULL   2:FULL   3:FULL
   4:FULL   5:FULL   6:FULL   7:FULL
   8:FULL   9:FULL   10:FULL   11:FULL
   12:FULL   13:FULL   14:FULL   15:FULL
   16:FULL   17:FULL   18:FULL   19:FULL
   20:FULL   21:FULL   22:FULL   23:FULL
   24:FULL   25:FULL   26:FULL   27:FULL
   28:FULL   29:FULL   30:FULL   31:FULL
   32:FULL   33:FULL   34:FULL   35:FULL
   36:FULL   37:FULL   38:FULL   39:FULL
   40:FULL   41:FULL   42:FULL   43:FULL
   44:FULL   45:FULL   46:FULL   47:FULL
   48:FULL   49:FULL   50:FULL   51:FULL
   52:FULL   53:FULL   54:FULL   55:FULL
   56:FULL   57:FULL   58:FULL   59:FULL
   60:FULL   61:FULL   62:FULL   63:FULL
  --------------------------------------------------------
End dump data blocks tsn: 8 file#: 5 minblk 8065 maxblk 8065
```

>居然全满了？再看8192号L1呢

```
DBA Ranges :
  --------------------------------------------------------
   0x01402000  Length: 128    Offset: 0      
   0x01402080  Length: 128    Offset: 128    

   0:Metadata   1:FULL   2:FULL   3:FULL
   4:FULL   5:FULL   6:FULL   7:FULL
   8:FULL   9:FULL   10:FULL   11:FULL
   12:FULL   13:FULL   14:FULL   15:FULL
   16:FULL   17:FULL   18:FULL   19:FULL
   20:FULL   21:FULL   22:FULL   23:FULL
   24:FULL   25:FULL   26:FULL   27:FULL
   28:FULL   29:FULL   30:FULL   31:FULL
   32:FULL   33:FULL   34:FULL   35:FULL
   36:FULL   37:FULL   38:FULL   39:FULL
   40:FULL   41:FULL   42:FULL   43:FULL
   44:FULL   45:FULL   46:FULL   47:FULL
   48:FULL   49:FULL   50:FULL   51:FULL
   52:FULL   53:FULL   54:FULL   55:FULL
   56:FULL   57:FULL   58:FULL   59:FULL
   60:FULL   61:FULL   62:FULL   63:FULL
   64:FULL   65:FULL   66:FULL   67:FULL
   68:FULL   69:FULL   70:FULL   71:FULL
   72:FULL   73:FULL   74:FULL   75:FULL
   76:FULL   77:FULL   78:FULL   79:FULL
   80:FULL   81:FULL   82:FULL   83:FULL
   84:FULL   85:FULL   86:FULL   87:FULL
   88:FULL   89:FULL   90:FULL   91:FULL
   92:FULL   93:FULL   94:FULL   95:FULL
   96:FULL   97:FULL   98:FULL   99:FULL
   100:FULL   101:FULL   102:FULL   103:FULL
   104:FULL   105:FULL   106:FULL   107:FULL
   108:FULL   109:FULL   110:FULL   111:FULL
   112:FULL   113:FULL   114:FULL   115:FULL
   116:FULL   117:FULL   118:FULL   119:FULL
   120:FULL   121:FULL   122:FULL   123:FULL
   124:FULL   125:FULL   126:FULL   127:FULL
   128:unformatted   129:unformatted   130:unformatted   131:unformatted
   132:unformatted   133:unformatted   134:unformatted   135:unformatted
   136:unformatted   137:unformatted   138:unformatted   139:unformatted
   140:unformatted   141:unformatted   142:unformatted   143:unformatted
   144:75-100% free   145:75-100% free   146:75-100% free   147:75-100% free
   148:75-100% free   149:75-100% free   150:75-100% free   151:75-100% free
   152:75-100% free   153:75-100% free   154:75-100% free   155:FULL
   156:75-100% free   157:75-100% free   158:75-100% free   159:75-100% free
   160:75-100% free   161:75-100% free   162:75-100% free   163:FULL
   164:75-100% free   165:75-100% free   166:75-100% free   167:75-100% free
   168:75-100% free   169:75-100% free   170:75-100% free   171:75-100% free
   172:75-100% free   173:75-100% free   174:75-100% free   175:75-100% free
   176:unformatted   177:unformatted   178:unformatted   179:unformatted
   180:unformatted   181:unformatted   182:unformatted   183:unformatted
   184:unformatted   185:unformatted   186:unformatted   187:unformatted
   188:unformatted   189:unformatted   190:unformatted   191:unformatted
   192:unformatted   193:unformatted   194:unformatted   195:unformatted
   196:unformatted   197:unformatted   198:unformatted   199:unformatted
   200:unformatted   201:unformatted   202:unformatted   203:unformatted
   204:unformatted   205:unformatted   206:unformatted   207:unformatted
   208:unformatted   209:unformatted   210:unformatted   211:unformatted
   212:unformatted   213:unformatted   214:unformatted   215:unformatted
   216:unformatted   217:unformatted   218:unformatted   219:unformatted
   220:unformatted   221:unformatted   222:unformatted   223:unformatted
   224:unformatted   225:unformatted   226:unformatted   227:unformatted
   228:unformatted   229:unformatted   230:unformatted   231:unformatted
   232:unformatted   233:unformatted   234:unformatted   235:unformatted
   236:unformatted   237:unformatted   238:unformatted   239:unformatted
   240:unformatted   241:unformatted   242:unformatted   243:unformatted
   244:unformatted   245:unformatted   246:unformatted   247:unformatted
   248:unformatted   249:unformatted   250:unformatted   251:unformatted
   252:unformatted   253:unformatted   254:unformatted   255:unformatted
  --------------------------------------------------------
End dump data blocks tsn: 8 file#: 5 minblk 8192 maxblk 8192
```

>完蛋了，计算错误，L1下面的第二个区也已经被格式化了，看段头信息

```
Extent Control Header
  -----------------------------------------------------------------
  Extent Header:: spare1: 0      spare2: 0      #extents: 101    #blocks: 12928
                  last map  0x00000000  #maps: 0      offset: 2716  
      Highwater::  0x01402100  ext#: 64     blk#: 128    ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 8191  
  mapblk  0x00000000  offset: 64   
```

>果然高水位指到了第66个区的L1块地址，这会影响我们的判断，到底哪里出问题了呢？观察8192号块内容是后128个块里被插入了2条，前128个块全满，也就是说我多插了130个块，后来猛然醒悟，我使用DBA来计算的，忘记减去128个块的文件头了，这么一算结果相查差还是不大的，太马虎了，要不是因为马虎哥也能上清华北大了；然后该怎么办呢？重新来过？不用！既然这个L1推过头了，索性把它堆满，然后我们去观察下一个L1，这下就好计算了；

```sql
declare
i number;
begin
for i in 1..126 loop
insert into lp values(i,'a','a','a','a');
end loop;
end;
/
```

>观察8192号块

```
 DBA Ranges :
  --------------------------------------------------------
   0x01402000  Length: 128    Offset: 0      
   0x01402080  Length: 128    Offset: 128    

   0:Metadata   1:FULL   2:FULL   3:FULL
   4:FULL   5:FULL   6:FULL   7:FULL
   8:FULL   9:FULL   10:FULL   11:FULL
   12:FULL   13:FULL   14:FULL   15:FULL
   16:FULL   17:FULL   18:FULL   19:FULL
   20:FULL   21:FULL   22:FULL   23:FULL
   24:FULL   25:FULL   26:FULL   27:FULL
   28:FULL   29:FULL   30:FULL   31:FULL
   32:FULL   33:FULL   34:FULL   35:FULL
   36:FULL   37:FULL   38:FULL   39:FULL
   40:FULL   41:FULL   42:FULL   43:FULL
   44:FULL   45:FULL   46:FULL   47:FULL
   48:FULL   49:FULL   50:FULL   51:FULL
   52:FULL   53:FULL   54:FULL   55:FULL
   56:FULL   57:FULL   58:FULL   59:FULL
   60:FULL   61:FULL   62:FULL   63:FULL
   64:FULL   65:FULL   66:FULL   67:FULL
   68:FULL   69:FULL   70:FULL   71:FULL
   72:FULL   73:FULL   74:FULL   75:FULL
   76:FULL   77:FULL   78:FULL   79:FULL
   80:FULL   81:FULL   82:FULL   83:FULL
   84:FULL   85:FULL   86:FULL   87:FULL
   88:FULL   89:FULL   90:FULL   91:FULL
   92:FULL   93:FULL   94:FULL   95:FULL
   96:FULL   97:FULL   98:FULL   99:FULL
   100:FULL   101:FULL   102:FULL   103:FULL
   104:FULL   105:FULL   106:FULL   107:FULL
   108:FULL   109:FULL   110:FULL   111:FULL
   112:FULL   113:FULL   114:FULL   115:FULL
   116:FULL   117:FULL   118:FULL   119:FULL
   120:FULL   121:FULL   122:FULL   123:FULL
   124:FULL   125:FULL   126:FULL   127:FULL
   128:FULL   129:FULL   130:FULL   131:FULL
   132:FULL   133:FULL   134:FULL   135:FULL
   136:FULL   137:FULL   138:FULL   139:FULL
   140:FULL   141:FULL   142:FULL   143:FULL
   144:FULL   145:FULL   146:FULL   147:FULL
   148:FULL   149:FULL   150:FULL   151:FULL
   152:FULL   153:FULL   154:FULL   155:FULL
   156:FULL   157:FULL   158:FULL   159:FULL
   160:FULL   161:FULL   162:FULL   163:FULL
   164:FULL   165:FULL   166:FULL   167:FULL
   168:FULL   169:FULL   170:FULL   171:FULL
   172:FULL   173:FULL   174:FULL   175:FULL
   176:FULL   177:FULL   178:FULL   179:FULL
   180:FULL   181:FULL   182:FULL   183:FULL
   184:FULL   185:FULL   186:FULL   187:FULL
   188:FULL   189:FULL   190:FULL   191:FULL
   192:FULL   193:FULL   194:FULL   195:FULL
   196:FULL   197:FULL   198:FULL   199:FULL
   200:FULL   201:FULL   202:FULL   203:FULL
   204:FULL   205:FULL   206:FULL   207:FULL
   208:FULL   209:FULL   210:FULL   211:FULL
   212:FULL   213:FULL   214:FULL   215:FULL
   216:FULL   217:FULL   218:FULL   219:FULL
   220:FULL   221:FULL   222:FULL   223:FULL
   224:FULL   225:FULL   226:FULL   227:FULL
   228:FULL   229:FULL   230:FULL   231:FULL
   232:FULL   233:FULL   234:FULL   235:FULL
   236:FULL   237:FULL   238:FULL   239:FULL
   240:FULL   241:FULL   242:FULL   243:FULL
   244:FULL   245:FULL   246:FULL   247:FULL
   248:FULL   249:FULL   250:FULL   251:FULL
   252:FULL   253:FULL   254:FULL   255:FULL
  --------------------------------------------------------
End dump data blocks tsn: 8 file#: 5 minblk 8192 maxblk 8192
```

>已经全满了，这个时候我们再去看一下段头的高水位

```
 Extent Control Header
  -----------------------------------------------------------------
  Extent Header:: spare1: 0      spare2: 0      #extents: 101    #blocks: 12928
                  last map  0x00000000  #maps: 0      offset: 2716  
      Highwater::  0x01402100  ext#: 64     blk#: 128    ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 8191  
  mapblk  0x00000000  offset: 64
```

>没有动，看来oracle也是事不到眼前不知道着急啊，来看一看下一个L1

```sql
SQL> select dbms_utility.data_block_address_file(to_number('01402100', 'xxxxxxxx')) file#,
  2 dbms_utility.data_block_address_block(to_number('01402100', 'xxxxxxxx')) block#
  3 from dual;

     FILE# BLOCK#
---------- ----------
         5 8448
```

>这个L1是5号文件8448号块，dump出来

```
 DBA Ranges :
  --------------------------------------------------------
   0x01402100  Length: 128    Offset: 0      
   0x01402180  Length: 128    Offset: 128    

   0:Metadata   1:unformatted   2:unformatted   3:unformatted
   4:unformatted   5:unformatted   6:unformatted   7:unformatted
   8:unformatted   9:unformatted   10:unformatted   11:unformatted
   12:unformatted   13:unformatted   14:unformatted   15:unformatted
   16:unformatted   17:unformatted   18:unformatted   19:unformatted
   20:unformatted   21:unformatted   22:unformatted   23:unformatted
   24:unformatted   25:unformatted   26:unformatted   27:unformatted
   28:unformatted   29:unformatted   30:unformatted   31:unformatted
   32:unformatted   33:unformatted   34:unformatted   35:unformatted
   36:unformatted   37:unformatted   38:unformatted   39:unformatted
   40:unformatted   41:unformatted   42:unformatted   43:unformatted
   44:unformatted   45:unformatted   46:unformatted   47:unformatted
   48:unformatted   49:unformatted   50:unformatted   51:unformatted
   52:unformatted   53:unformatted   54:unformatted   55:unformatted
   56:unformatted   57:unformatted   58:unformatted   59:unformatted
   60:unformatted   61:unformatted   62:unformatted   63:unformatted
   64:unformatted   65:unformatted   66:unformatted   67:unformatted
   68:unformatted   69:unformatted   70:unformatted   71:unformatted
   72:unformatted   73:unformatted   74:unformatted   75:unformatted
   76:unformatted   77:unformatted   78:unformatted   79:unformatted
   80:unformatted   81:unformatted   82:unformatted   83:unformatted
   84:unformatted   85:unformatted   86:unformatted   87:unformatted
   88:unformatted   89:unformatted   90:unformatted   91:unformatted
   92:unformatted   93:unformatted   94:unformatted   95:unformatted
   96:unformatted   97:unformatted   98:unformatted   99:unformatted
   100:unformatted   101:unformatted   102:unformatted   103:unformatted
   104:unformatted   105:unformatted   106:unformatted   107:unformatted
   108:unformatted   109:unformatted   110:unformatted   111:unformatted
   112:unformatted   113:unformatted   114:unformatted   115:unformatted
   116:unformatted   117:unformatted   118:unformatted   119:unformatted
   120:unformatted   121:unformatted   122:unformatted   123:unformatted
   124:unformatted   125:unformatted   126:unformatted   127:unformatted
   128:unformatted   129:unformatted   130:unformatted   131:unformatted
   132:unformatted   133:unformatted   134:unformatted   135:unformatted
   136:unformatted   137:unformatted   138:unformatted   139:unformatted
   140:unformatted   141:unformatted   142:unformatted   143:unformatted
   144:unformatted   145:unformatted   146:unformatted   147:unformatted
   148:unformatted   149:unformatted   150:unformatted   151:unformatted
   152:unformatted   153:unformatted   154:unformatted   155:unformatted
   156:unformatted   157:unformatted   158:unformatted   159:unformatted
   160:unformatted   161:unformatted   162:unformatted   163:unformatted
   164:unformatted   165:unformatted   166:unformatted   167:unformatted
   168:unformatted   169:unformatted   170:unformatted   171:unformatted
   172:unformatted   173:unformatted   174:unformatted   175:unformatted
   176:unformatted   177:unformatted   178:unformatted   179:unformatted
   180:unformatted   181:unformatted   182:unformatted   183:unformatted
   184:unformatted   185:unformatted   186:unformatted   187:unformatted
   188:unformatted   189:unformatted   190:unformatted   191:unformatted
   192:unformatted   193:unformatted   194:unformatted   195:unformatted
   196:unformatted   197:unformatted   198:unformatted   199:unformatted
   200:unformatted   201:unformatted   202:unformatted   203:unformatted
   204:unformatted   205:unformatted   206:unformatted   207:unformatted
   208:unformatted   209:unformatted   210:unformatted   211:unformatted
   212:unformatted   213:unformatted   214:unformatted   215:unformatted
   216:unformatted   217:unformatted   218:unformatted   219:unformatted
   220:unformatted   221:unformatted   222:unformatted   223:unformatted
   224:unformatted   225:unformatted   226:unformatted   227:unformatted
   228:unformatted   229:unformatted   230:unformatted   231:unformatted
   232:unformatted   233:unformatted   234:unformatted   235:unformatted
   236:unformatted   237:unformatted   238:unformatted   239:unformatted
   240:unformatted   241:unformatted   242:unformatted   243:unformatted
   244:unformatted   245:unformatted   246:unformatted   247:unformatted
   248:unformatted   249:unformatted   250:unformatted   251:unformatted
   252:unformatted   253:unformatted   254:unformatted   255:unformatted
  --------------------------------------------------------
End dump data blocks tsn: 8 file#: 5 minblk 8448 maxblk 8448
```

>256个块，除了第一个是L1元数据之外，没有一个格式化的，这就达到了我们的目的，我现在要插入一条数据，迫使高水位推动，猜一下高水位会在哪？在L1管理的最后一个块后面么？
看插入一行数据之后的8448号块

```
DBA Ranges :
  --------------------------------------------------------
   0x01402100  Length: 128    Offset: 0      
   0x01402180  Length: 128    Offset: 128    

   0:Metadata   1:unformatted   2:unformatted   3:unformatted
   4:unformatted   5:unformatted   6:unformatted   7:unformatted
   8:unformatted   9:unformatted   10:unformatted   11:unformatted
   12:unformatted   13:unformatted   14:unformatted   15:unformatted
   16:75-100% free   17:75-100% free   18:75-100% free   19:75-100% free
   20:75-100% free   21:75-100% free   22:75-100% free   23:75-100% free
   24:75-100% free   25:75-100% free   26:75-100% free   27:75-100% free
   28:FULL   29:75-100% free   30:75-100% free   31:75-100% free
   32:unformatted   33:unformatted   34:unformatted   35:unformatted
   36:unformatted   37:unformatted   38:unformatted   39:unformatted
   40:unformatted   41:unformatted   42:unformatted   43:unformatted
   44:unformatted   45:unformatted   46:unformatted   47:unformatted
   48:unformatted   49:unformatted   50:unformatted   51:unformatted
   52:unformatted   53:unformatted   54:unformatted   55:unformatted
   56:unformatted   57:unformatted   58:unformatted   59:unformatted
   60:unformatted   61:unformatted   62:unformatted   63:unformatted
   64:unformatted   65:unformatted   66:unformatted   67:unformatted
   68:unformatted   69:unformatted   70:unformatted   71:unformatted
   72:unformatted   73:unformatted   74:unformatted   75:unformatted
   76:unformatted   77:unformatted   78:unformatted   79:unformatted
   80:unformatted   81:unformatted   82:unformatted   83:unformatted
   84:unformatted   85:unformatted   86:unformatted   87:unformatted
   88:unformatted   89:unformatted   90:unformatted   91:unformatted
   92:unformatted   93:unformatted   94:unformatted   95:unformatted
   96:unformatted   97:unformatted   98:unformatted   99:unformatted
   100:unformatted   101:unformatted   102:unformatted   103:unformatted
   104:unformatted   105:unformatted   106:unformatted   107:unformatted
   108:unformatted   109:unformatted   110:unformatted   111:unformatted
   112:unformatted   113:unformatted   114:unformatted   115:unformatted
   116:unformatted   117:unformatted   118:unformatted   119:unformatted
   120:unformatted   121:unformatted   122:unformatted   123:unformatted
   124:unformatted   125:unformatted   126:unformatted   127:unformatted
   128:unformatted   129:unformatted   130:unformatted   131:unformatted
   132:unformatted   133:unformatted   134:unformatted   135:unformatted
   136:unformatted   137:unformatted   138:unformatted   139:unformatted
   140:unformatted   141:unformatted   142:unformatted   143:unformatted
   144:unformatted   145:unformatted   146:unformatted   147:unformatted
   148:unformatted   149:unformatted   150:unformatted   151:unformatted
   152:unformatted   153:unformatted   154:unformatted   155:unformatted
   156:unformatted   157:unformatted   158:unformatted   159:unformatted
   160:unformatted   161:unformatted   162:unformatted   163:unformatted
   164:unformatted   165:unformatted   166:unformatted   167:unformatted
   168:unformatted   169:unformatted   170:unformatted   171:unformatted
   172:unformatted   173:unformatted   174:unformatted   175:unformatted
   176:unformatted   177:unformatted   178:unformatted   179:unformatted
   180:unformatted   181:unformatted   182:unformatted   183:unformatted
   184:unformatted   185:unformatted   186:unformatted   187:unformatted
   188:unformatted   189:unformatted   190:unformatted   191:unformatted
   192:unformatted   193:unformatted   194:unformatted   195:unformatted
   196:unformatted   197:unformatted   198:unformatted   199:unformatted
   200:unformatted   201:unformatted   202:unformatted   203:unformatted
   204:unformatted   205:unformatted   206:unformatted   207:unformatted
   208:unformatted   209:unformatted   210:unformatted   211:unformatted
   212:unformatted   213:unformatted   214:unformatted   215:unformatted
   216:unformatted   217:unformatted   218:unformatted   219:unformatted
   220:unformatted   221:unformatted   222:unformatted   223:unformatted
   224:unformatted   225:unformatted   226:unformatted   227:unformatted
   228:unformatted   229:unformatted   230:unformatted   231:unformatted
   232:unformatted   233:unformatted   234:unformatted   235:unformatted
   236:unformatted   237:unformatted   238:unformatted   239:unformatted
   240:unformatted   241:unformatted   242:unformatted   243:unformatted
   244:unformatted   245:unformatted   246:unformatted   247:unformatted
   248:unformatted   249:unformatted   250:unformatted   251:unformatted
   252:unformatted   253:unformatted   254:unformatted   255:unformatted
  --------------------------------------------------------
End dump data blocks tsn: 8 file#: 5 minblk 8448 maxblk 8448
```

>有一个块已经满了，揭晓答案的时候来了，dump出段头

```
Extent Control Header
  -----------------------------------------------------------------
  Extent Header:: spare1: 0      spare2: 0      #extents: 101    #blocks: 12928
                  last map  0x00000000  #maps: 0      offset: 2716  
      Highwater::  0x01402180  ext#: 65     blk#: 128    ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 8318  
  mapblk  0x00000000  offset: 65
```

>高水位是0x01402180，这个地址在哪呢？0x01402180比0x01402100多了十六进制80个块，就是十进制的128个块，仅仅是推动了一个区的大小，并不是一个L1的大小，这说明常规路径下插入高水位的推动是以L1和区中小的那个单位来推动的，就是L1小就推动L1里面所有块的大小，区小就推动一个区的大小，可见所谓的高并发并非像宣传的那样，还是受高水位的限制的，这个时候的区块大概像下面这个样子：

![](http://qiniu.liupzmin.com/%E6%89%A9%E5%B1%95%E7%9A%84L1.png)

>开几个session做下插入试试

```sql
SQL> select dbms_rowid.ROWID_RELATIVE_FNO(rowid),dbms_rowid.ROWID_BLOCK_NUMBER(rowid) from lp where trim(des1)='b';

                                   5                                 8474
                                   5                                 8482
                                   5                                 8484
                                   5                                 8488
                                   5                                 8489
                                   5                                 8490
                                   5                                 8491
                                   5                                 8492
                                   5                                 8493
                                   5                                 8494
                                   5                                 8495
                                   5                                 8496
                                   5                                 8498
                                   5                                 8500
                                   5                                 8504
                                   5                                 8506
                                   5                                 8512
                                   5                                 8514
                                   5                                 8520
                                   5                                 8522

20 rows selected.
```

>全部是在高水位之下
这个实验有一点值得思考，就是PCTFREE如何设置，在插入删除频繁的段，PCTFREE的值应该要远离三个百分比线，就是25%/50%/75%，避免频繁插入删除的时候块状态频繁改变，由full变为非full，如此就会修改L1块的内容，有可能造成buffer busy waits。
而大家熟知的直接路径下的插入是直接在高水位之上插入并且绕过cache buffer的，这种情况下的高水位是如何推动的呢？我们再慢慢分析.
