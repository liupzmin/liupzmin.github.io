---
date: 2015-09-30 14:31
tags: oracle
title: ASSM三级位图结构与高水位的探究（下）
categories: oracle
---

上一篇文章介绍了ASSM三级位图管理下的数据插入和高水位推进的关系，我们得出的结论是随着数据不断的插入，段大小的不断增长，L1中挂载的数据块的数量也会增长，而高水位是以L1中数据块大小的总和与区大小之间最小的一方为单位进行推进，而且我们知道要想应对大并发插入，就要使高水位之下的空闲块的数量尽可能多，但是如此一来就有可能造成空间的浪费，而oracle的初心也是本着节省空间的目的来设计的。
之前我们用的是常规路径插入，能以L1和区的单位推动高水位，前提是要填满L1或区，而我们熟知的直接路径插入是在高水位之上插入，那是不是可以用直接路径插入快速增加高水位呢？我们来一探究竟。

>首先构造测试的表空间和表，跟上一篇一样

```sql
SQL> drop tablespace lp including contents and datafiles;

Tablespace dropped.

SQL> create tablespace lp datafile '/home/oracle/app/oracle/oradata/DG43/lp.dbf' size 2048M uniform size 1m;
create table lp (id number,des1 char(2000),des2 char(2000),des3 char(2000),des4 char(500)) tablespace lp;

Tablespace created.

SQL>
Table created.

SQL> alter table lp pctfree 24;

Table altered.


SQL> select object_id,object_name from dba_objects  where object_name='LP';

 OBJECT_ID OBJECT_NAME
---------- ------------------------------
     78760 LP
```

>首先看一下，78760这个对象在buffer里的数据块有哪些

```sql
SQL> select file#,BLOCK#,DIRTY,OBJD from v$bh where objd='78760';

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        131 N      78760
         5        129 N      78760
         5        130 N      78760
         5        128 N      78760
```
>有没有眼熟呢，128、129、130、131四个块刚好是两个L1、L2、L3
我们定位段头块，看一下此时的高水位

```sql
SQL> select segment_name ,HEADER_FILE,HEADER_BLOCK from dba_segments where segment_name='LP';

SEGMENT_NAME                   HEADER_FILE HEADER_BLOCK
------------------------------ ----------- ------------
LP                                       5          131

Extent Control Header
  -----------------------------------------------------------------
  Extent Header:: spare1: 0      spare2: 0      #extents: 1      #blocks: 128   
                  last map  0x00000000  #maps: 0      offset: 2716  
      Highwater::  0x01400084  ext#: 0      blk#: 4      ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 0     
  mapblk  0x00000000  offset: 0     

SQL> select dbms_utility.data_block_address_file(to_number('01400084', 'xxxxxxxx')) file#,
  2                      dbms_utility.data_block_address_block(to_number('01400084', 'xxxxxxxx')) block#
  3  from dual;

     FILE#     BLOCK#
---------- ----------
         5        132
```

>可以看出在没有任何数据的情况下，高水位是在第一个数据块上的，下面常规插入一行数据


```sql
insert into lp values(1,'a','a','a','a');
```

>看一下高水位和buffer中的块


```sql
SQL> select file#,BLOCK#,DIRTY,OBJD from v$bh where objd='78760';

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        165 Y      78760
         5        131 Y      78760
         5        173 Y      78760
         5        160 Y      78760
         5        168 Y      78760
         5        163 Y      78760
         5        129 N      78760
         5        171 Y      78760
         5        166 Y      78760
         5        174 Y      78760
         5        161 Y      78760

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        169 Y      78760
         5        164 Y      78760
         5        130 N      78760
         5        172 Y      78760
         5        167 Y      78760
         5        175 Y      78760
         5        162 Y      78760
         5        128 Y      78760
         5        170 Y      78760

20 rows selected.

SQL> select dbms_rowid.ROWID_RELATIVE_FNO(rowid),dbms_rowid.ROWID_BLOCK_NUMBER(rowid) from lp;

DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID)
------------------------------------ ------------------------------------
                                   5                                  160
```

```
Extent Control Header
-----------------------------------------------------------------
Extent Header:: spare1: 0 spare2: 0 #extents: 1 #blocks: 128
last map 0x00000000 #maps: 0 offset: 2716
Highwater:: 0x014000c0 ext#: 0 blk#: 64 ext size: 128
#blocks in seg. hdr's freelists: 0
#blocks below: 60
mapblk 0x00000000 offset: 0
```

>发现数据插入到160号块中，buffer中却多了16个块，高水位已移动到第二个L1中的第一个块上，看一下第一个L1

```
HWM Flag: HWM Set
      Highwater::  0x014000c0  ext#: 0      blk#: 64     ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 60    
  mapblk  0x00000000  offset: 0     
  --------------------------------------------------------
  DBA Ranges :
  --------------------------------------------------------
   0x01400080  Length: 64     Offset: 0      

   0:Metadata   1:Metadata   2:Metadata   3:Metadata
   4:unformatted   5:unformatted   6:unformatted   7:unformatted
   8:unformatted   9:unformatted   10:unformatted   11:unformatted
   12:unformatted   13:unformatted   14:unformatted   15:unformatted
   16:unformatted   17:unformatted   18:unformatted   19:unformatted
   20:unformatted   21:unformatted   22:unformatted   23:unformatted
   24:unformatted   25:unformatted   26:unformatted   27:unformatted
   28:unformatted   29:unformatted   30:unformatted   31:unformatted
   32:FULL   33:75-100% free   34:75-100% free   35:75-100% free
   36:75-100% free   37:75-100% free   38:75-100% free   39:75-100% free
   40:75-100% free   41:75-100% free   42:75-100% free   43:75-100% free
   44:75-100% free   45:75-100% free   46:75-100% free   47:75-100% free
   48:unformatted   49:unformatted   50:unformatted   51:unformatted
   52:unformatted   53:unformatted   54:unformatted   55:unformatted
   56:unformatted   57:unformatted   58:unformatted   59:unformatted
   60:unformatted   61:unformatted   62:unformatted   63:unformatted
```

```sql
   SQL> select dbms_utility.data_block_address_file(to_number('014000c0', 'xxxxxxxx')) file#,
  2                      dbms_utility.data_block_address_block(to_number('014000c0', 'xxxxxxxx')) block#
  3  from dual;

     FILE#     BLOCK#
---------- ----------
         5        192
```

>可以发现，buffer中新增的16个块是一次性格式化的这16个块，而高水位指在192号块上，直接路径插入一行

```sql
SQL> select file#,BLOCK#,DIRTY,OBJD from v$bh where objd='78760';

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        165 N      78760
         5        131 N      78760
         5        173 N      78760
         5        160 N      78760
         5        168 N      78760
         5        163 N      78760
         5        129 N      78760
         5        171 N      78760
         5        166 N      78760
         5        174 N      78760
         5        161 N      78760

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        169 N      78760
         5        164 N      78760
         5        130 N      78760
         5        172 N      78760
         5        167 N      78760
         5        175 N      78760
         5        162 N      78760
         5        128 N      78760
         5        170 N      78760

20 rows selected.


SQL> insert /*+ append_values(lp)*/ into lp values(2,'b','b','b','b');

1 row created.

SQL> commit;

Commit complete.

SQL> select dbms_rowid.ROWID_RELATIVE_FNO(rowid),dbms_rowid.ROWID_BLOCK_NUMBER(rowid) from lp;

DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID)
------------------------------------ ------------------------------------
                                   5                                  160
                                   5                                  192

SQL> select file#,BLOCK#,DIRTY,OBJD from v$bh where objd='78760';

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        165 N      78760
         5        131 Y      78760
         5        173 N      78760
         5        160 N      78760
         5        168 N      78760
         5        163 N      78760
         5        129 Y      78760
         5        171 N      78760
         5        192 Y      78760
         5        166 N      78760
         5        174 N      78760

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        161 N      78760
         5        169 N      78760
         5        164 N      78760
         5        130 N      78760
         5        172 N      78760
         5        167 N      78760
         5        175 N      78760
         5        162 N      78760
         5        128 Y      78760
         5        170 N      78760

21 rows selected.

SQL> alter system checkpoint;

System altered.
```

>可以发现确实插入到了192号块，并且192号块已在buffer中，看一下此时的高水位

```
Extent Control Header
  -----------------------------------------------------------------
  Extent Header:: spare1: 0      spare2: 0      #extents: 1      #blocks: 128   
                  last map  0x00000000  #maps: 0      offset: 2716  
      Highwater::  0x014000c1  ext#: 0      blk#: 65     ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 65    
  mapblk  0x00000000  offset: 0  

HWM Flag: HWM Set
      Highwater::  0x014000c1  ext#: 0      blk#: 65     ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 65    
  mapblk  0x00000000  offset: 0     
  --------------------------------------------------------
  DBA Ranges :
  --------------------------------------------------------
   0x014000c0  Length: 64     Offset: 0      

   0:FULL   1:unformatted   2:unformatted   3:unformatted
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
End dump data blocks tsn: 5 file#: 5 minblk 129 maxblk 129
```

>发现高水位仅仅移动了1个block，再试一遍

```sql
SQL> insert /*+ append_values(lp)*/ into lp values(3,'c','c','c','c');

1 row created.

SQL> commit;

Commit complete.

SQL> select dbms_rowid.ROWID_RELATIVE_FNO(rowid),dbms_rowid.ROWID_BLOCK_NUMBER(rowid) from lp;

DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID)
------------------------------------ ------------------------------------
                                   5                                  160
                                   5                                  192
                                   5                                  193

SQL> select file#,BLOCK#,DIRTY,OBJD from v$bh where objd='78760';

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        165 N      78760
         5        131 Y      78760
         5        173 N      78760
         5        160 N      78760
         5        168 N      78760
         5        163 N      78760
         5        129 Y      78760
         5        171 N      78760
         5        192 N      78760
         5        166 N      78760
         5        174 N      78760

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        161 N      78760
         5        169 N      78760
         5        164 N      78760
         5        130 N      78760
         5        172 N      78760
         5        193 Y      78760
         5        167 N      78760
         5        175 N      78760
         5        162 N      78760
         5        128 N      78760
         5        170 N      78760

22 rows selected.
```

```
Extent Header:: spare1: 0      spare2: 0      #extents: 1      #blocks: 128   
                  last map  0x00000000  #maps: 0      offset: 2716  
      Highwater::  0x014000c2  ext#: 0      blk#: 66     ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 66    
  mapblk  0x00000000  offset: 0     
```

  >仍然是只推动了一个block，但是发现为何每次直接路径插入的block都会出现在buffer里呢？难道是11g这个append_values新的hint的原因，再来试一下传统的append

```sql
  SQL> insert /*+ append*/ into lp select *  from lp where trim(des1)='a';

1 row created.

SQL> commit;

Commit complete.

SQL> select dbms_rowid.ROWID_RELATIVE_FNO(rowid),dbms_rowid.ROWID_BLOCK_NUMBER(rowid) from lp;

DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID)
------------------------------------ ------------------------------------
                                   5                                  160
                                   5                                  192
                                   5                                  193
                                   5                                  194

SQL> select file#,BLOCK#,DIRTY,OBJD from v$bh where objd='78760';

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        165 N      78760
         5        131 Y      78760
         5        173 N      78760
         5        194 Y      78760
         5        160 N      78760
         5        168 N      78760
         5        163 N      78760
         5        129 Y      78760
         5        171 N      78760
         5        192 N      78760
         5        166 N      78760

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        174 N      78760
         5        161 N      78760
         5        169 N      78760
         5        164 N      78760
         5        130 N      78760
         5        172 N      78760
         5        193 N      78760
         5        167 N      78760
         5        175 N      78760
         5        162 N      78760
         5        128 N      78760

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        170 N      78760

23 rows selected.

SQL> alter system checkpoint;

System altered.
```

```
  Extent Control Header
  -----------------------------------------------------------------
  Extent Header:: spare1: 0      spare2: 0      #extents: 1      #blocks: 128   
                  last map  0x00000000  #maps: 0      offset: 2716  
      Highwater::  0x014000c3  ext#: 0      blk#: 67     ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 67    
  mapblk  0x00000000  offset: 0
```

>依然如故，不知道大家有没有发现，每次我执行完插入，都会执行一条select dbms_rowid.ROWID_RELATIVE_FNO(rowid),dbms_rowid.ROWID_BLOCK_NUMBER(rowid) from lp;
这其实是全表扫描，本来块没在buffer里，一个FTS把块全给整进来了（小表buffer读，大表在11g中通常情况是直接路径读），知道原因了，再试一遍

```sql
SQL> insert /*+ append*/ into lp select *  from lp where trim(des1)='b';

1 row created.

SQL> commit;

Commit complete.

SQL> select file#,BLOCK#,DIRTY,OBJD from v$bh where objd='78760';

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        165 N      78760
         5        131 Y      78760
         5        173 N      78760
         5        194 N      78760
         5        160 N      78760
         5        168 N      78760
         5        163 N      78760
         5        129 Y      78760
         5        171 N      78760
         5        192 N      78760
         5        166 N      78760

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        174 N      78760
         5        161 N      78760
         5        169 N      78760
         5        164 N      78760
         5        130 N      78760
         5        172 N      78760
         5        193 N      78760
         5        167 N      78760
         5        175 N      78760
         5        162 N      78760
         5        128 N      78760

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        170 N      78760

23 rows selected.

SQL> select dbms_rowid.ROWID_RELATIVE_FNO(rowid),dbms_rowid.ROWID_BLOCK_NUMBER(rowid) from lp;

DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID)
------------------------------------ ------------------------------------
                                   5                                  160
                                   5                                  192
                                   5                                  193
                                   5                                  194
                                   5                                  195

SQL> select file#,BLOCK#,DIRTY,OBJD from v$bh where objd='78760';

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        165 N      78760
         5        131 Y      78760
         5        173 N      78760
         5        194 N      78760
         5        160 N      78760
         5        168 N      78760
         5        163 N      78760
         5        129 Y      78760
         5        171 N      78760
         5        192 N      78760
         5        166 N      78760

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        174 N      78760
         5        195 Y      78760
         5        161 N      78760
         5        169 N      78760
         5        164 N      78760
         5        130 N      78760
         5        172 N      78760
         5        193 N      78760
         5        167 N      78760
         5        175 N      78760
         5        162 N      78760

     FILE#     BLOCK# D       OBJD
---------- ---------- - ----------
         5        128 N      78760
         5        170 N      78760

24 rows selected.
```

>终于看到了想要的结果，总结一下吧，直接路径下高水位的推进大概如下图的样子：


![](http://qiniu.liupzmin.com/direct.png)

>不知道大家有没有注意到，直接路径插入完后再读入buffer中居然是个脏块，这是不是跟延迟提交清除有关呢？我们以后再慢慢分析！
