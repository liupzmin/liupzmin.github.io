---
date: 2015-09-23 11:04
tags: oracle backup
title: RMAN异机异目录完全恢复
categories: oracle backup
---

>开始

先说一下为什么要写这个异机完全恢复的blog，作为一个年轻的DBA，相信大多数都是备份长做而恢复不常做，我会经常担心我的生产库的服务器挂了怎么办，我的备份完好，但我们又没有容灾，我能否保证我的数据不丢失呢，所以就要rman利用备份重建，而如果此时我能找到完好无损的在线日志，我是不是可以完全恢复了呢，理论上大家都说可以，确从未在网上见到过完全恢复的事例（完全其实很简单，只是网上阐述了太多的不完全恢复，会让我觉得set until time等等是必备的其中一步），甚至oracle的官方网文档上也使用了“SET UNTIL SCN 123456;” 的案例，有鉴于此，我做了这个实验，并记录下来，跟大家分享，不见得这篇原创有多高深的技术和见解O(∩_∩)O

```
环境：
    源库：
        os：redhat 6.3 x64
        db: 11gr2 11.2.0.4
        存储方式：ASM
    目标库：
        os：redhat 6.5 x64
        db: 11gr2 11.2.0.4
        存储方式：文件系统
```

## 1. 首先，源库进行rman备份
```sql
--手动0级全库备份
run {
    CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
    CONFIGURE BACKUP OPTIMIZATION ON;
    CONFIGURE CONTROLFILE AUTOBACKUP ON;
    CONFIGURE DEVICE TYPE DISK PARALLELISM 4 BACKUP TYPE TO BACKUPSET;
    CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '+DATA/backup/ctl_atuo_%F';
    ALLOCATE CHANNEL c1 TYPE DISK;
    ALLOCATE CHANNEL c2 TYPE DISK;
    CROSSCHECK ARCHIVELOG ALL;
    BACKUP INCREMENTAL LEVEL 0
    DATABASE FORMAT '+DATA/backup/db_%U' TAG 'db_rman';
    SQL 'ALTER SYSTEM ARCHIVE LOG CURRENT';
    BACKUP ARCHIVELOG ALL FORMAT '+DATA/backup/arc_%U' TAG 'ARC_rman'
    DELETE INPUT;
    DELETE NOPROMPT OBSOLETE;
    RELEASE CHANNEL c1;
    RELEASE CHANNEL c2;
}
```
<!--more-->
### 观察备份的文件，此处是ASM分别用asmcmd和rman查看
```sql
ASMCMD> ls
arc_0cqhdnqq_1_1
arc_0dqhdnqq_1_1
ctl_atuo_c-2496627597-20150917-00
ctl_atuo_c-2496627597-20150917-01
db_09qhdno6_1_1
db_0aqhdno6_1_1
ASMCMD> pwd
+Data/backup
ASMCMD>

RMAN> list backup summary;


List of Backups
===============
Key     TY LV S Device Type Completion Time #Pieces #Copies Compressed Tag
------- -- -- - ----------- --------------- ------- ------- ---------- ---
9       B  0  A DISK        17-SEP-15       1       1       NO         DB_RMAN
10      B  0  A DISK        17-SEP-15       1       1       NO         DB_RMAN
11      B  F  A DISK        17-SEP-15       1       1       NO         TAG20150917T221650
12      B  A  A DISK        17-SEP-15       1       1       NO         ARC_RMAN
13      B  A  A DISK        17-SEP-15       1       1       NO         ARC_RMAN
14      B  F  A DISK        17-SEP-15       1       1       NO         TAG20150917T221703
```
### 为了试验restore之后recover应用归档日志的情况，我在0级备份之后再生成一些归档日志，脚本如下：
```sql
declare
  i integer;
begin
  for i in 1..3000000 loop
    insert into t values(i,'aaaaaaaaaaaaaaaaaaaaa');
    end loop;
end;
```
### 向测试表T中插入200万行数据，看归档的日志：
```
ASMCMD> ls
1_22_854775184.dbf
1_23_854775184.dbf
1_24_854775184.dbf
1_25_854775184.dbf
1_26_854775184.dbf
1_27_854775184.dbf
1_28_854775184.dbf
1_29_854775184.dbf
1_30_854775184.dbf
1_31_854775184.dbf
1_32_854775184.dbf
1_33_854775184.dbf
1_34_854775184.dbf
1_35_854775184.dbf
1_36_854775184.dbf
1_37_854775184.dbf
1_38_854775184.dbf
1_39_854775184.dbf
```
### 查看T表的行数
```sql
SQL> select count(1) from t;

  COUNT(1)
----------
   3000000
```
## 2. 转移备份
现在要把rman备份，备份之后的归档，源库关闭之后的在线redo文件拷贝到备库，这里稍微麻烦一点，涉及到从asm中拷贝文件，我这里使用2中方法，rman的backup as cpoy以及asmcmd的cp命令，如下所示：
>rman copy备份集

```sql
--这里是copy 0级备份
backup as copy backupset 9 format '/u01/backupset/db2.rman';
backup as copy backupset 10 format '/u01/backupset/db2.rman';
backup as copy backupset 11 format '/u01/backupset/ctl1.rman';
backup as copy backupset 12 format '/u01/backupset/arc1.rman';
backup as copy backupset 13 format '/u01/backupset/arc2.rman';
backup as copy backupset 14 format '/u01/backupset/ctl2.rman';
```
>asmcmd转移归档

```
--生成的归档日志
ASMCMD> ls
1_22_854775184.dbf
1_23_854775184.dbf
1_24_854775184.dbf
1_25_854775184.dbf
1_26_854775184.dbf
1_27_854775184.dbf
1_28_854775184.dbf
1_29_854775184.dbf
1_30_854775184.dbf
1_31_854775184.dbf
1_32_854775184.dbf
1_33_854775184.dbf
1_34_854775184.dbf
1_35_854775184.dbf
1_36_854775184.dbf
1_37_854775184.dbf
1_38_854775184.dbf
1_39_854775184.dbf
--copy走，无法使用通配符是一件很痛苦的事情
ASMCMD> cp 1_23_854775184.dbf /u02/backupset
copying +Data/archive_log/1_23_854775184.dbf -> /u02/backupset/1_23_854775184.dbf
ASMCMD> cp 1_24_854775184.dbf /u02/backupset
copying +Data/archive_log/1_24_854775184.dbf -> /u02/backupset/1_24_854775184.dbf
ASMCMD> cp 1_25_854775184.dbf /u02/backupset
copying +Data/archive_log/1_25_854775184.dbf -> /u02/backupset/1_25_854775184.dbf
ASMCMD> cp 1_26_854775184.dbf /u02/backupset
copying +Data/archive_log/1_26_854775184.dbf -> /u02/backupset/1_26_854775184.dbf
ASMCMD> cp 1_27_854775184.dbf /u02/backupset
copying +Data/archive_log/1_27_854775184.dbf -> /u02/backupset/1_27_854775184.dbf
ASMCMD> cp 1_28_854775184.dbf /u02/backupset
copying +Data/archive_log/1_28_854775184.dbf -> /u02/backupset/1_28_854775184.dbf
ASMCMD> cp 1_29_854775184.dbf /u02/backupset
copying +Data/archive_log/1_29_854775184.dbf -> /u02/backupset/1_29_854775184.dbf
ASMCMD> cp 1_3* /u02/backupset
copying +Data/archive_log/1_30_854775184.dbf -> /u02/backupset/1_30_854775184.dbf
ASMCMD> cp 1_31_854775184.dbf /u02/backupset
copying +Data/archive_log/1_31_854775184.dbf -> /u02/backupset/1_31_854775184.dbf
ASMCMD> cp 1_32_854775184.dbf /u02/backupset
copying +Data/archive_log/1_32_854775184.dbf -> /u02/backupset/1_32_854775184.dbf
ASMCMD> cp 1_33_854775184.dbf /u02/backupset
copying +Data/archive_log/1_33_854775184.dbf -> /u02/backupset/1_33_854775184.dbf
ASMCMD> cp 1_34_854775184.dbf /u02/backupset
copying +Data/archive_log/1_34_854775184.dbf -> /u02/backupset/1_34_854775184.dbf
ASMCMD> cp 1_35_854775184.dbf /u02/backupset
copying +Data/archive_log/1_35_854775184.dbf -> /u02/backupset/1_35_854775184.dbf
ASMCMD> cp 1_36_854775184.dbf /u02/backupset
copying +Data/archive_log/1_36_854775184.dbf -> /u02/backupset/1_36_854775184.dbf
ASMCMD> cp 1_37_854775184.dbf /u02/backupset
copying +Data/archive_log/1_37_854775184.dbf -> /u02/backupset/1_37_854775184.dbf
ASMCMD> cp 1_38_854775184.dbf /u02/backupset
copying +Data/archive_log/1_38_854775184.dbf -> /u02/backupset/1_38_854775184.dbf
ASMCMD> cp 1_39_854775184.dbf /u02/backupset
copying +Data/archive_log/1_39_854775184.dbf -> /u02/backupset/1_39_854775184.dbf
```
### 查看当前的redo状态,记住当前redo的seq号
```sql
SQL> select group#,SEQUENCE# ,MEMBERS,ARCHIVED,STATUS from v$log;

    GROUP#  SEQUENCE#    MEMBERS ARC STATUS
---------- ---------- ---------- --- ----------------
         1         40          2 NO  CURRENT
         2         38          2 YES INACTIVE
         3         39          2 YES INACTIVE
 ```
### 查看redo文件
 ```sql
 SQL> select GROUP#,STATUS,MEMBER  from v$logfile;

    GROUP# STATUS  MEMBER
---------- ------- --------------------------------------------------------------------------------
         3         +DATA/min/onlinelog/group_3.266.854775193
         3         +DATA/min/onlinelog/group_3.267.854775195
         2         +DATA/min/onlinelog/group_2.264.854775189
         2         +DATA/min/onlinelog/group_2.265.854775191
         1         +DATA/min/onlinelog/group_1.262.854775185
         1         +DATA/min/onlinelog/group_1.263.854775187
 ```
 >使用asmcmd拷贝并改名，这个时候我已经关闭了源库

```
ASMCMD> cd onlinelog
ASMCMD>
ASMCMD> ls
group_1.262.854775185
group_1.263.854775187
group_2.264.854775189
group_2.265.854775191
group_3.266.854775193
group_3.267.854775195

ASMCMD> cp group_1.262.854775185 /u02/backupset/redo1a.log
copying +Data/min/onlinelog/group_1.262.854775185 -> /u02/backupset/redo1a.log
ASMCMD> cp group_1.263.854775187 /u02/backupset/redo1b.log
copying +Data/min/onlinelog/group_1.263.854775187 -> /u02/backupset/redo1b.log
ASMCMD> cp group_2.264.854775189 /u02/backupset/redo2a.log
copying +Data/min/onlinelog/group_2.264.854775189 -> /u02/backupset/redo2a.log
ASMCMD> cp group_2.265.854775191 /u02/backupset/redo2b.log
copying +Data/min/onlinelog/group_2.265.854775191 -> /u02/backupset/redo2b.log
ASMCMD> cp group_3.266.854775193 /u02/backupset/redo3a.log
copying +Data/min/onlinelog/group_3.266.854775193 -> /u02/backupset/redo3a.log
ASMCMD> cp group_3.267.854775195 /u02/backupset/redo3b.log
copying +Data/min/onlinelog/group_3.267.854775195 -> /u02/backupset/redo3b.log
```
## 3.  目标机器恢复
这一步之前已经在目标主机安装好oracle软件，安装过程不再赘述，直接记录恢复过程
这里直接从备份中恢复pfile（比较懒，不想手动准备），无参数文件，rman会启动一个傻瓜实例，供我们恢复参数文件。
###从备份中恢复参数文件
```sql
RMAN> startup nomount

startup failed: ORA-01078: failure in processing system parameters
LRM-00109: could not open parameter file '/home/oracle/app/oracle/product/11.2.0/dbhome_1/dbs/initmin.ora'

starting Oracle instance without parameter file for retrieval of spfile
Oracle instance started

Total System Global Area    1068937216 bytes

Fixed Size                     2260088 bytes
Variable Size                281019272 bytes
Database Buffers             780140544 bytes
Redo Buffers                   5517312 bytes


RMAN> restore spfile to pfile '/home/oracle/app/oracle/product/11.2.0/dbhome_1/dbs/initmin.ora' from '/u01/rmanbackup/ctl2.rman';

Starting restore at 18-SEP-15
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=171 device type=DISK

channel ORA_DISK_1: restoring spfile from AUTOBACKUP /u01/rmanbackup/ctl2.rman
channel ORA_DISK_1: SPFILE restore from AUTOBACKUP complete
Finished restore at 18-SEP-15

RMAN> shutdown abort

Oracle instance shut down
```
### 创建目录，修改参数文件
```bash
[oracle@rman_newhost u01]$ mkdir -p /u01/app/oracle/admin/min/adump
[oracle@rman_newhost u01]$ mkdir -p /u01/app/oracle/fast_recovery_area
[oracle@rman_newhost u01]$ mkdir -p /u01/app/oracle/archived_log
[oracle@rman_newhost u01]$ mkdir -p /u01/app/oracle/oradata/min
*.audit_file_dest='/u01/app/oracle/admin/min/adump'
*.audit_trail='db'
*.compatible='11.2.0.4.0'
*.control_files='/u01/app/oracle/fast_recovery_area/control01.ctl','/u01/app/oracle/oradata/min/control02.ctl'
*.db_block_size=8192
*.db_domain=''
*.db_name='min'
*.db_recovery_file_dest='/u01/app/oracle/fast_recovery_area'
*.db_recovery_file_dest_size=4385144832
*.diagnostic_dest='/u01/app/oracle'
*.dispatchers='(PROTOCOL=TCP) (SERVICE=minXDB)'
*.log_archive_dest_1='location=/u01/app/oracle/archived_log'
*.log_checkpoints_to_alert=TRUE
*.memory_target=1394606080
*.open_cursors=300
*.processes=1500
*.remote_login_passwordfile='EXCLUSIVE'
*.sessions=1655
*.undo_tablespace='UNDOTBS1'
```
### 启动到nomount状态，恢复控制文件
```sql
RMAN> startup nomount

connected to target database (not started)
Oracle instance started

Total System Global Area    1402982400 bytes

Fixed Size                     2253184 bytes
Variable Size               1275072128 bytes
Database Buffers             117440512 bytes
Redo Buffers                   8216576 bytes

RMAN> restore controlfile from '/u01/rmanbackup/ctl2.rman';

Starting restore at 18-SEP-15
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=1146 device type=DISK

channel ORA_DISK_1: restoring control file
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
output file name=/u01/app/oracle/fast_recovery_area/control01.ctl
output file name=/u01/app/oracle/oradata/min/control02.ctl
Finished restore at 18-SEP-15

RMAN> alter database mount;

database mounted
released channel: ORA_DISK_1
```
### 将redo文件和归档日志移动到对应目录下
```bash
[oracle@rman_newhost rmanbackup]$ mv redo* /u01/app/oracle/oradata/min
[oracle@rman_newhost rmanbackup]$ ll /u01/app/oracle/oradata/min
总用量 316744
-rw-r-----. 1 oracle oinstall 9748480 9月 18 00:34 control02.ctl
-rw-r--r--. 1 oracle oinstall 52429312 9月 17 23:00 redo1a.log
-rw-r--r--. 1 oracle oinstall 52429312 9月 17 23:01 redo1b.log
-rw-r--r--. 1 oracle oinstall 52429312 9月 17 23:01 redo2a.log
-rw-r--r--. 1 oracle oinstall 52429312 9月 17 23:01 redo2b.log
-rw-r--r--. 1 oracle oinstall 52429312 9月 17 23:02 redo3a.log
-rw-r--r--. 1 oracle oinstall 52429312 9月 17 23:02 redo3b.log
[oracle@rman_newhost rmanbackup]$ mv 1_* /u01/app/oracle/archived_log
[oracle@rman_newhost rmanbackup]$ ll /u01/app/oracle/archived_log
总用量 809056
-rw-r--r--. 1 oracle oinstall 46381568 9月 17 22:48 1_22_854775184.dbf
-rw-r--r--. 1 oracle oinstall 39179776 9月 17 22:49 1_23_854775184.dbf
-rw-r--r--. 1 oracle oinstall 46381568 9月 17 22:49 1_24_854775184.dbf
-rw-r--r--. 1 oracle oinstall 46381568 9月 17 22:50 1_25_854775184.dbf
-rw-r--r--. 1 oracle oinstall 46381568 9月 17 22:50 1_26_854775184.dbf
-rw-r--r--. 1 oracle oinstall 48581120 9月 17 22:50 1_27_854775184.dbf
-rw-r--r--. 1 oracle oinstall 46381568 9月 17 22:51 1_28_854775184.dbf
-rw-r--r--. 1 oracle oinstall 42950144 9月 17 22:51 1_29_854775184.dbf
-rw-r--r--. 1 oracle oinstall 46381568 9月 17 22:51 1_30_854775184.dbf
-rw-r--r--. 1 oracle oinstall 50134016 9月 17 22:53 1_31_854775184.dbf
-rw-r--r--. 1 oracle oinstall 46381568 9月 17 22:53 1_32_854775184.dbf
-rw-r--r--. 1 oracle oinstall 46714368 9月 17 22:53 1_33_854775184.dbf
-rw-r--r--. 1 oracle oinstall 44308480 9月 17 22:54 1_34_854775184.dbf
-rw-r--r--. 1 oracle oinstall 46381568 9月 17 22:54 1_35_854775184.dbf
-rw-r--r--. 1 oracle oinstall 46381568 9月 17 22:54 1_36_854775184.dbf
-rw-r--r--. 1 oracle oinstall 46381568 9月 17 22:54 1_37_854775184.dbf
-rw-r--r--. 1 oracle oinstall 46381568 9月 17 22:55 1_38_854775184.dbf
-rw-r--r--. 1 oracle oinstall 46381568 9月 17 22:55 1_39_854775184.dbf
```
### 注册备份集
下面一步的操作是删掉控制文件中对备份的记录信息，因为这是源库备份恢复过来的，我们要删掉老的备份信息，将备份重新注册，不删也没关系，crosscheck以后你会发现，原来的备份状态都是expired的
>利用crosscheck删除控制文件中的备份记录

```sql
RMAN> list backup summary;


List of Backups
===============
Key     TY LV S Device Type Completion Time #Pieces #Copies Compressed Tag
------- -- -- - ----------- --------------- ------- ------- ---------- ---
9       B  0  A DISK        17-SEP-15       1       1       NO         DB_RMAN
10      B  0  A DISK        17-SEP-15       1       1       NO         DB_RMAN
11      B  F  A DISK        17-SEP-15       1       1       NO         TAG20150917T221650
12      B  A  A DISK        17-SEP-15       1       1       NO         ARC_RMAN
13      B  A  A DISK        17-SEP-15       1       1       NO         ARC_RMAN

RMAN> crosscheck backup;

Starting implicit crosscheck backup at 18-SEP-15
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=1146 device type=DISK
allocated channel: ORA_DISK_2
channel ORA_DISK_2: SID=10 device type=DISK
allocated channel: ORA_DISK_3
channel ORA_DISK_3: SID=1147 device type=DISK
allocated channel: ORA_DISK_4
channel ORA_DISK_4: SID=11 device type=DISK
Crosschecked 5 objects
Finished implicit crosscheck backup at 18-SEP-15

Starting implicit crosscheck copy at 18-SEP-15
using channel ORA_DISK_1
using channel ORA_DISK_2
using channel ORA_DISK_3
using channel ORA_DISK_4
Finished implicit crosscheck copy at 18-SEP-15

searching for all files in the recovery area
cataloging files...
no files cataloged

using channel ORA_DISK_1
using channel ORA_DISK_2
using channel ORA_DISK_3
using channel ORA_DISK_4
crosschecked backup piece: found to be 'EXPIRED'
backup piece handle=+DATA/backup/db_0aqhdno6_1_1 RECID=9 STAMP=890691334
crosschecked backup piece: found to be 'EXPIRED'
backup piece handle=+DATA/backup/db_09qhdno6_1_1 RECID=10 STAMP=890691334
crosschecked backup piece: found to be 'EXPIRED'
backup piece handle=+DATA/backup/ctl_atuo_c-2496627597-20150917-00 RECID=11 STAMP=890691411
crosschecked backup piece: found to be 'EXPIRED'
backup piece handle=+DATA/backup/arc_0dqhdnqq_1_1 RECID=12 STAMP=890691418
crosschecked backup piece: found to be 'EXPIRED'
backup piece handle=+DATA/backup/arc_0cqhdnqq_1_1 RECID=13 STAMP=890691418
Crosschecked 5 objects


RMAN> list backup;


List of Backup Sets
===================


BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
9       Incr 0  433.65M    DISK        00:01:10     17-SEP-15
        BP Key: 9   Status: EXPIRED  Compressed: NO  Tag: DB_RMAN
        Piece Name: +DATA/backup/db_0aqhdno6_1_1
  List of Datafiles in backup set 9
  File LV Type Ckp SCN    Ckp Time  Name
  ---- -- ---- ---------- --------- ----
  2    0  Incr 1138043    17-SEP-15 +DATA/min/datafile/sysaux.257.854775097
  3    0  Incr 1138043    17-SEP-15 +DATA/min/datafile/undotbs1.258.854775097

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
10      Incr 0  638.84M    DISK        00:01:10     17-SEP-15
        BP Key: 10   Status: EXPIRED  Compressed: NO  Tag: DB_RMAN
        Piece Name: +DATA/backup/db_09qhdno6_1_1
  List of Datafiles in backup set 10
  File LV Type Ckp SCN    Ckp Time  Name
  ---- -- ---- ---------- --------- ----
  1    0  Incr 1138042    17-SEP-15 +DATA/min/datafile/system.256.854775095
  4    0  Incr 1138042    17-SEP-15 +DATA/min/datafile/users.259.854775097

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
11      Full    9.36M      DISK        00:00:01     17-SEP-15
        BP Key: 11   Status: EXPIRED  Compressed: NO  Tag: TAG20150917T221650
        Piece Name: +DATA/backup/ctl_atuo_c-2496627597-20150917-00
  SPFILE Included: Modification time: 17-SEP-15
  SPFILE db_unique_name: MIN
  Control File Included: Ckp SCN: 1138347      Ckp time: 17-SEP-15

BS Key  Size       Device Type Elapsed Time Completion Time
------- ---------- ----------- ------------ ---------------
12      2.00K      DISK        00:00:00     17-SEP-15
        BP Key: 12   Status: EXPIRED  Compressed: NO  Tag: ARC_RMAN
        Piece Name: +DATA/backup/arc_0dqhdnqq_1_1

  List of Archived Logs in backup set 12
  Thrd Seq     Low SCN    Low Time  Next SCN   Next Time
  ---- ------- ---------- --------- ---------- ---------
  1    21      1138364    17-SEP-15 1138372    17-SEP-15

BS Key  Size       Device Type Elapsed Time Completion Time
------- ---------- ----------- ------------ ---------------
13      30.99M     DISK        00:00:02     17-SEP-15
        BP Key: 13   Status: EXPIRED  Compressed: NO  Tag: ARC_RMAN
        Piece Name: +DATA/backup/arc_0cqhdnqq_1_1

  List of Archived Logs in backup set 13
  Thrd Seq     Low SCN    Low Time  Next SCN   Next Time
  ---- ------- ---------- --------- ---------- ---------
  1    20      1127220    17-SEP-15 1138364    17-SEP-15

RMAN> list archivelog all;

specification does not match any archived log in the repository


RMAN> report obsolete;

RMAN retention policy will be applied to the command
RMAN retention policy is set to recovery window of 7 days
no obsolete backups found

RMAN> list expired backup;


List of Backup Sets
===================


BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
9       Incr 0  433.65M    DISK        00:01:10     17-SEP-15
        BP Key: 9   Status: EXPIRED  Compressed: NO  Tag: DB_RMAN
        Piece Name: +DATA/backup/db_0aqhdno6_1_1
  List of Datafiles in backup set 9
  File LV Type Ckp SCN    Ckp Time  Name
  ---- -- ---- ---------- --------- ----
  2    0  Incr 1138043    17-SEP-15 +DATA/min/datafile/sysaux.257.854775097
  3    0  Incr 1138043    17-SEP-15 +DATA/min/datafile/undotbs1.258.854775097

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
10      Incr 0  638.84M    DISK        00:01:10     17-SEP-15
        BP Key: 10   Status: EXPIRED  Compressed: NO  Tag: DB_RMAN
        Piece Name: +DATA/backup/db_09qhdno6_1_1
  List of Datafiles in backup set 10
  File LV Type Ckp SCN    Ckp Time  Name
  ---- -- ---- ---------- --------- ----
  1    0  Incr 1138042    17-SEP-15 +DATA/min/datafile/system.256.854775095
  4    0  Incr 1138042    17-SEP-15 +DATA/min/datafile/users.259.854775097

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
11      Full    9.36M      DISK        00:00:01     17-SEP-15
        BP Key: 11   Status: EXPIRED  Compressed: NO  Tag: TAG20150917T221650
        Piece Name: +DATA/backup/ctl_atuo_c-2496627597-20150917-00
  SPFILE Included: Modification time: 17-SEP-15
  SPFILE db_unique_name: MIN
  Control File Included: Ckp SCN: 1138347      Ckp time: 17-SEP-15

BS Key  Size       Device Type Elapsed Time Completion Time
------- ---------- ----------- ------------ ---------------
12      2.00K      DISK        00:00:00     17-SEP-15
        BP Key: 12   Status: EXPIRED  Compressed: NO  Tag: ARC_RMAN
        Piece Name: +DATA/backup/arc_0dqhdnqq_1_1

  List of Archived Logs in backup set 12
  Thrd Seq     Low SCN    Low Time  Next SCN   Next Time
  ---- ------- ---------- --------- ---------- ---------
  1    21      1138364    17-SEP-15 1138372    17-SEP-15

BS Key  Size       Device Type Elapsed Time Completion Time
------- ---------- ----------- ------------ ---------------
13      30.99M     DISK        00:00:02     17-SEP-15
        BP Key: 13   Status: EXPIRED  Compressed: NO  Tag: ARC_RMAN
        Piece Name: +DATA/backup/arc_0cqhdnqq_1_1

  List of Archived Logs in backup set 13
  Thrd Seq     Low SCN    Low Time  Next SCN   Next Time
  ---- ------- ---------- --------- ---------- ---------
  1    20      1127220    17-SEP-15 1138364    17-SEP-15

RMAN> delete noprompt expired backup;

using channel ORA_DISK_1
using channel ORA_DISK_2
using channel ORA_DISK_3
using channel ORA_DISK_4

List of Backup Pieces
BP Key  BS Key  Pc# Cp# Status      Device Type Piece Name
------- ------- --- --- ----------- ----------- ----------
9       9       1   1   EXPIRED     DISK        +DATA/backup/db_0aqhdno6_1_1
10      10      1   1   EXPIRED     DISK        +DATA/backup/db_09qhdno6_1_1
11      11      1   1   EXPIRED     DISK        +DATA/backup/ctl_atuo_c-2496627597-20150917-00
12      12      1   1   EXPIRED     DISK        +DATA/backup/arc_0dqhdnqq_1_1
13      13      1   1   EXPIRED     DISK        +DATA/backup/arc_0cqhdnqq_1_1
deleted backup piece
backup piece handle=+DATA/backup/db_0aqhdno6_1_1 RECID=9 STAMP=890691334
deleted backup piece
backup piece handle=+DATA/backup/db_09qhdno6_1_1 RECID=10 STAMP=890691334
deleted backup piece
backup piece handle=+DATA/backup/ctl_atuo_c-2496627597-20150917-00 RECID=11 STAMP=890691411
deleted backup piece
backup piece handle=+DATA/backup/arc_0dqhdnqq_1_1 RECID=12 STAMP=890691418
deleted backup piece
backup piece handle=+DATA/backup/arc_0cqhdnqq_1_1 RECID=13 STAMP=890691418
Deleted 5 EXPIRED objects


RMAN> list backup;

specification does not match any backup in the repository
```
>重新注册备份，并观察控制文件中的备份信息，和数据库结构的信息

```sql
RMAN> catalog start with '/u01/rmanbackup';

searching for all files that match the pattern /u01/rmanbackup

List of Files Unknown to the Database
=====================================
File Name: /u01/rmanbackup/arc1.rman
File Name: /u01/rmanbackup/db1.rman
File Name: /u01/rmanbackup/ctl2.rman
File Name: /u01/rmanbackup/ctl1.rman
File Name: /u01/rmanbackup/db2.rman
File Name: /u01/rmanbackup/arc2.rman

Do you really want to catalog the above files (enter YES or NO)? yes
cataloging files...
cataloging done

List of Cataloged Files
=======================
File Name: /u01/rmanbackup/arc1.rman
File Name: /u01/rmanbackup/db1.rman
File Name: /u01/rmanbackup/ctl2.rman
File Name: /u01/rmanbackup/ctl1.rman
File Name: /u01/rmanbackup/db2.rman
File Name: /u01/rmanbackup/arc2.rman

RMAN> list backup summary;


List of Backups
===============
Key     TY LV S Device Type Completion Time #Pieces #Copies Compressed Tag
------- -- -- - ----------- --------------- ------- ------- ---------- ---
14      B  A  A DISK        17-SEP-15       1       1       NO         ARC_RMAN
15      B  0  A DISK        17-SEP-15       1       1       NO         DB_RMAN
16      B  0  A DISK        17-SEP-15       1       1       NO         DB_RMAN
17      B  A  A DISK        17-SEP-15       1       1       NO         ARC_RMAN

--数据文件、临时文件的记录还是源库的信息，一会儿恢复的时候要setnewname
RMAN> report schema;

RMAN-06139: WARNING: control file is not current for REPORT SCHEMA
Report of database schema for database with db_unique_name MIN

List of Permanent Datafiles
===========================
File Size(MB) Tablespace           RB segs Datafile Name
---- -------- -------------------- ------- ------------------------
1    0        SYSTEM               ***     +DATA/min/datafile/system.256.854775095
2    0        SYSAUX               ***     +DATA/min/datafile/sysaux.257.854775097
3    0        UNDOTBS1             ***     +DATA/min/datafile/undotbs1.258.854775097
4    0        USERS                ***     +DATA/min/datafile/users.259.854775097

List of Temporary Files
=======================
File Size(MB) Tablespace           Maxsize(MB) Tempfile Name
---- -------- -------------------- ----------- --------------------
1    20       TEMP                 32767       +DATA/min/tempfile/temp.268.854775211

--你会发现这是老的控制文件，是我们0级备份里的，其中关于redo的状态记录，当前redo还停留在序列号22
SQL> select GROUP#,SEQUENCE# ,STATUS,ARCHIVED  from v$log;

    GROUP#  SEQUENCE# STATUS           ARC
---------- ---------- ---------------- ---
         1         22 CURRENT          NO
         3         21 ACTIVE           YES
         2         20 ACTIVE           YES

SQL> desc v$logfile;
 Name                                      Null?    Type
 ----------------------------------------- -------- ----------------------------
 GROUP#                                             NUMBER
 STATUS                                             VARCHAR2(7)
 TYPE                                               VARCHAR2(7)
 MEMBER                                             VARCHAR2(513)
 IS_RECOVERY_DEST_FILE                              VARCHAR2(3)

--redo的记录还是源库的信息，一会儿恢复的时候要setnewname
SQL> col member for a80
SQL> set line 200
SQL> select GROUP#,STATUS,MEMBER from v$logfile;

    GROUP# STATUS  MEMBER
---------- ------- --------------------------------------------------------------------------------
         3         +DATA/min/onlinelog/group_3.266.854775193
         3         +DATA/min/onlinelog/group_3.267.854775195
         2         +DATA/min/onlinelog/group_2.264.854775189
         2         +DATA/min/onlinelog/group_2.265.854775191
         1         +DATA/min/onlinelog/group_1.262.854775185
         1         +DATA/min/onlinelog/group_1.263.854775187

6 rows selected.
```
### 还原数据库
```sql
RMAN> RUN
2>  {
3>    ALLOCATE CHANNEL c1 DEVICE TYPE DISK;
4>    ALLOCATE CHANNEL c2 DEVICE TYPE DISK;
5>    set archivelog destination to "/u01/app/oracle/archived_log";
6>       set newname for datafile 1 to "/u01/app/oracle/oradata/min/system01.dbf";
7>       set newname for datafile 2 to "/u01/app/oracle/oradata/min/sysaux01.dbf";
8>       set newname for datafile 3 to "/u01/app/oracle/oradata/min/undotbs01.dbf";
9>       set newname for datafile 4 to "/u01/app/oracle/oradata/min/users01.dbf";
10>      set newname for tempfile 1 to "/u01/app/oracle/oradata/min/temp01.dbf";
11>      SQL "ALTER DATABASE RENAME FILE ''+DATA/min/onlinelog/group_1.262.854775185''  to  ''/u01/app/oracle/oradata/min/redo1a.log'' ";
12>      SQL "ALTER DATABASE RENAME FILE ''+DATA/min/onlinelog/group_1.263.854775187''  to  ''/u01/app/oracle/oradata/min/redo1b.log'' ";
13>      SQL "ALTER DATABASE RENAME FILE ''+DATA/min/onlinelog/group_2.264.854775189''  to  ''/u01/app/oracle/oradata/min/redo2a.log'' ";
14>      SQL "ALTER DATABASE RENAME FILE ''+DATA/min/onlinelog/group_2.265.854775191''  to  ''/u01/app/oracle/oradata/min/redo2b.log'' ";
15>      SQL "ALTER DATABASE RENAME FILE ''+DATA/min/onlinelog/group_3.266.854775193''  to  ''/u01/app/oracle/oradata/min/redo3a.log'' ";
16>      SQL "ALTER DATABASE RENAME FILE ''+DATA/min/onlinelog/group_3.267.854775195''  to  ''/u01/app/oracle/oradata/min/redo3b.log'' ";
17>    RESTORE DATABASE;
18>    SWITCH DATAFILE ALL;
19>    switch tempfile all;
20>  }

released channel: ORA_DISK_1
released channel: ORA_DISK_2
released channel: ORA_DISK_3
released channel: ORA_DISK_4
allocated channel: c1
channel c1: SID=1146 device type=DISK

allocated channel: c2
channel c2: SID=11 device type=DISK

executing command: SET ARCHIVELOG DESTINATION

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

sql statement: ALTER DATABASE RENAME FILE ''+DATA/min/onlinelog/group_1.262.854775185''  to  ''/u01/app/oracle/oradata/min/redo1a.log''

sql statement: ALTER DATABASE RENAME FILE ''+DATA/min/onlinelog/group_1.263.854775187''  to  ''/u01/app/oracle/oradata/min/redo1b.log''

sql statement: ALTER DATABASE RENAME FILE ''+DATA/min/onlinelog/group_2.264.854775189''  to  ''/u01/app/oracle/oradata/min/redo2a.log''

sql statement: ALTER DATABASE RENAME FILE ''+DATA/min/onlinelog/group_2.265.854775191''  to  ''/u01/app/oracle/oradata/min/redo2b.log''

sql statement: ALTER DATABASE RENAME FILE ''+DATA/min/onlinelog/group_3.266.854775193''  to  ''/u01/app/oracle/oradata/min/redo3a.log''

sql statement: ALTER DATABASE RENAME FILE ''+DATA/min/onlinelog/group_3.267.854775195''  to  ''/u01/app/oracle/oradata/min/redo3b.log''

Starting restore at 18-SEP-15

skipping datafile 2; already restored to file /u01/app/oracle/oradata/min/sysaux01.dbf
channel c1: starting datafile backup set restore
channel c1: specifying datafile(s) to restore from backup set
channel c1: restoring datafile 00001 to /u01/app/oracle/oradata/min/system01.dbf
channel c1: restoring datafile 00004 to /u01/app/oracle/oradata/min/users01.dbf
channel c1: reading from backup piece /u01/rmanbackup/db2.rman
channel c2: starting datafile backup set restore
channel c2: specifying datafile(s) to restore from backup set
channel c2: restoring datafile 00003 to /u01/app/oracle/oradata/min/undotbs01.dbf
channel c2: reading from backup piece /u01/rmanbackup/db1.rman
channel c2: piece handle=/u01/rmanbackup/db1.rman tag=DB_RMAN
channel c2: restored backup piece 1
channel c2: restore complete, elapsed time: 00:00:08
channel c1: piece handle=/u01/rmanbackup/db2.rman tag=DB_RMAN
channel c1: restored backup piece 1
channel c1: restore complete, elapsed time: 00:00:36
Finished restore at 18-SEP-15

datafile 1 switched to datafile copy
input datafile copy RECID=5 STAMP=890701322 file name=/u01/app/oracle/oradata/min/system01.dbf
datafile 2 switched to datafile copy
input datafile copy RECID=6 STAMP=890701322 file name=/u01/app/oracle/oradata/min/sysaux01.dbf
datafile 3 switched to datafile copy
input datafile copy RECID=7 STAMP=890701322 file name=/u01/app/oracle/oradata/min/undotbs01.dbf
datafile 4 switched to datafile copy
input datafile copy RECID=8 STAMP=890701322 file name=/u01/app/oracle/oradata/min/users01.dbf

renamed tempfile 1 to /u01/app/oracle/oradata/min/temp01.dbf in control file
released channel: c1
released channel: c2

RMAN>
```
>restore完毕，我们再来看一下控制文件中的信息

```sql
RMAN> report schema;

RMAN-06139: WARNING: control file is not current for REPORT SCHEMA
Report of database schema for database with db_unique_name MIN

List of Permanent Datafiles
===========================
File Size(MB) Tablespace           RB segs Datafile Name
---- -------- -------------------- ------- ------------------------
1    750      SYSTEM               ***     /u01/app/oracle/oradata/min/system01.dbf
2    560      SYSAUX               ***     /u01/app/oracle/oradata/min/sysaux01.dbf
3    100      UNDOTBS1             ***     /u01/app/oracle/oradata/min/undotbs01.dbf
4    5        USERS                ***     /u01/app/oracle/oradata/min/users01.dbf

List of Temporary Files
=======================
File Size(MB) Tablespace           Maxsize(MB) Tempfile Name
---- -------- -------------------- ----------- --------------------
1    20       TEMP                 32767       /u01/app/oracle/oradata/min/temp01.dbf
```
### 完全恢复数据库
```sql
RMAN> recover database;

Starting recover at 18-SEP-15
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=1146 device type=DISK
allocated channel: ORA_DISK_2
channel ORA_DISK_2: SID=11 device type=DISK
allocated channel: ORA_DISK_3
channel ORA_DISK_3: SID=1137 device type=DISK
allocated channel: ORA_DISK_4
channel ORA_DISK_4: SID=10 device type=DISK

starting media recovery

archived log for thread 1 with sequence 38 is already on disk as file /u01/app/oracle/oradata/min/redo2a.log
archived log for thread 1 with sequence 39 is already on disk as file /u01/app/oracle/oradata/min/redo3a.log
archived log for thread 1 with sequence 40 is already on disk as file /u01/app/oracle/oradata/min/redo1a.log
channel ORA_DISK_1: starting archived log restore to default destination
channel ORA_DISK_1: restoring archived log
archived log thread=1 sequence=20
channel ORA_DISK_1: reading from backup piece /u01/rmanbackup/arc2.rman
channel ORA_DISK_2: starting archived log restore to default destination
channel ORA_DISK_2: restoring archived log
archived log thread=1 sequence=21
channel ORA_DISK_2: reading from backup piece /u01/rmanbackup/arc1.rman
channel ORA_DISK_2: piece handle=/u01/rmanbackup/arc1.rman tag=ARC_RMAN
channel ORA_DISK_2: restored backup piece 1
channel ORA_DISK_2: restore complete, elapsed time: 00:00:01
channel ORA_DISK_1: piece handle=/u01/rmanbackup/arc2.rman tag=ARC_RMAN
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:03
archived log file name=/u01/app/oracle/archived_log/1_20_854775184.dbf thread=1 sequence=20
archived log file name=/u01/app/oracle/archived_log/1_21_854775184.dbf thread=1 sequence=21
archived log file name=/u01/app/oracle/archived_log/1_22_854775184.dbf thread=1 sequence=22
archived log file name=/u01/app/oracle/archived_log/1_23_854775184.dbf thread=1 sequence=23
archived log file name=/u01/app/oracle/archived_log/1_24_854775184.dbf thread=1 sequence=24
archived log file name=/u01/app/oracle/archived_log/1_25_854775184.dbf thread=1 sequence=25
archived log file name=/u01/app/oracle/archived_log/1_26_854775184.dbf thread=1 sequence=26
archived log file name=/u01/app/oracle/archived_log/1_27_854775184.dbf thread=1 sequence=27
archived log file name=/u01/app/oracle/archived_log/1_28_854775184.dbf thread=1 sequence=28
archived log file name=/u01/app/oracle/archived_log/1_29_854775184.dbf thread=1 sequence=29
archived log file name=/u01/app/oracle/archived_log/1_30_854775184.dbf thread=1 sequence=30
archived log file name=/u01/app/oracle/archived_log/1_31_854775184.dbf thread=1 sequence=31
archived log file name=/u01/app/oracle/archived_log/1_32_854775184.dbf thread=1 sequence=32
archived log file name=/u01/app/oracle/archived_log/1_33_854775184.dbf thread=1 sequence=33
archived log file name=/u01/app/oracle/archived_log/1_34_854775184.dbf thread=1 sequence=34
archived log file name=/u01/app/oracle/archived_log/1_35_854775184.dbf thread=1 sequence=35
archived log file name=/u01/app/oracle/archived_log/1_36_854775184.dbf thread=1 sequence=36
archived log file name=/u01/app/oracle/archived_log/1_37_854775184.dbf thread=1 sequence=37
archived log file name=/u01/app/oracle/oradata/min/redo2a.log thread=1 sequence=38
archived log file name=/u01/app/oracle/oradata/min/redo3a.log thread=1 sequence=39
archived log file name=/u01/app/oracle/oradata/min/redo1a.log thread=1 sequence=40
media recovery complete, elapsed time: 00:00:43
Finished recover at 18-SEP-15

RMAN>
```
从恢复的日志可以看出，oracle会绕过38、39号的归档直接利用在线日志做恢复，oracle的聪明之处可见一斑！
试着正常方式打开数据库，会发现报错：

```sql
RMAN> alter database open;

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of alter db command at 09/18/2015 01:04:41
ORA-01589: must use RESETLOGS or NORESETLOGS option for database open

--看一下此时控制文件中的日志序列号，会发现没有前推，必须用resetlogs打开数据库
SQL>  select group#,status,sequence# from v$log;

    GROUP# STATUS            SEQUENCE#
---------- ---------------- ----------
         1 CURRENT                  22
         3 ACTIVE                   21
         2 ACTIVE                   20
RMAN> alter database open resetlogs;

database opened
```

>使用resetlog的原因是recover命令只能修复控制文件中数据物理结构信息，而无法修改控制文件中的当前重做日志的序列号的信息，recover命令结束后，控制文件中的当前日志序列号还是陈旧的，若按常规方式打开数据库，将报错，为了抹去控制文件这个固执的念头，oracle采用重设日志的功能，日志序列号从1开始。此处虽然使用了resetlogs，但是因为“recover database”命令执行成功，所有提交的事务不会丢失，resetlogs仅仅是为了照顾还原的控制文件，与不完全恢复的resetlogs是不同的，至此恢复结束。

```sql
SQL> conn test/test
ERROR:
ORA-28002: the password will expire within 6 days


Connected.
SQL> select count(1) from t;

  COUNT(1)
----------
   3000000
```
