---
date: 2016-02-16 16:54
categories: oracle
tags: DataGuard
title: Oracle DataGuard switchover切换一例
---

*做DG的切换测试，发现了一些有趣的小问题，寥作记录*
*v$archived_log中的fal字段意思如下：*
*Indicates whether the archive log was generated as the result of a FAL request (YES) or not (NO)*
1. 主库当时归档日志状态

```sql
SQL> select DEST_ID,NAME ,SEQUENCE#,STANDBY_DEST,ARCHIVED,APPLIED,DELETED,STATUS,FAL,REGISTRAR,CREATOR from v$archived_log  order by 3;

   DEST_ID NAME                                                                              SEQUENCE# STA ARC APPLIED   DEL S FAL REGISTR CREATOR
---------- -------------------------------------------------------------------------------- ---------- --- --- --------- --- - --- ------- -------
         2 /u01/app/archive/1_74_903912515.arc                                                      74 NO  YES YES       NO  A YES RFS     ARCH
         2 /u01/app/archive/1_75_903912515.arc                                                      75 NO  YES YES       NO  A YES RFS     ARCH
         1 /u01/app/archive/1_76_903912515.arc                                                      76 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_77_903912515.arc                                                      77 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_78_903912515.arc                                                      78 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_79_903912515.arc                                                      79 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_80_903912515.arc                                                      80 NO  YES NO        NO  A NO  ARCH    ARCH
         2 min                                                                                      80 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_81_903912515.arc                                                      81 NO  YES NO        NO  A NO  ARCH    ARCH
         2 min                                                                                      81 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_82_903912515.arc                                                      82 NO  YES NO        NO  A NO  ARCH    ARCH
         2 min                                                                                      82 YES YES YES       NO  A NO  LGWR    LGWR
         1 /u01/app/archive/1_83_903912515.arc                                                      83 NO  YES NO        NO  A NO  ARCH    ARCH
         2 min                                                                                      83 YES YES YES       NO  A NO  LGWR    LGWR
         1 /u01/app/archive/1_84_903912515.arc                                                      84 NO  YES NO        NO  A NO  ARCH    ARCH
         2 min                                                                                      84 YES YES NO        NO  A NO  LGWR    LGWR
```

2. 备库当时归档日志状态


```sql
SQL> select DEST_ID,NAME ,SEQUENCE#,STANDBY_DEST,ARCHIVED,APPLIED,DELETED,STATUS,FAL,REGISTRAR,CREATOR from v$archived_log  order by 3;

   DEST_ID NAME                                                                                                  SEQUENCE# STA ARC APPLIED   DEL S FAL REGISTR CREATOR
---------- ---------------------------------------------------------------------------------------------------- ---------- --- --- --------- --- - --- ------- -------
         1 /u01/app/archive/1_72_903912515.arc                                                                          72 NO  YES YES       NO  A NO  ARCH    ARCH
         1 /u01/app/archive/1_73_903912515.arc                                                                          73 NO  YES YES       NO  A NO  ARCH    ARCH
         1 /u01/app/archive/1_74_903912515.arc                                                                          74 NO  YES YES       NO  A NO  ARCH    ARCH
         2 minstd                                                                                                       74 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_75_903912515.arc                                                                          75 NO  YES YES       NO  A NO  FGRD    FGRD
         2 minstd                                                                                                       75 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_76_903912515.arc                                                                          76 NO  YES YES       NO  A NO  ARCH    ARCH
         2 minstd                                                                                                       76 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_77_903912515.arc                                                                          77 NO  YES YES       NO  A NO  ARCH    ARCH
         2 minstd                                                                                                       77 YES YES YES       NO  A NO  LGWR    LGWR
         2 minstd                                                                                                       78 YES YES NO        NO  A NO  LGWR    LGWR
         1 /u01/app/archive/1_78_903912515.arc                                                                          78 NO  YES YES       NO  A NO  ARCH    ARCH
         1 /u01/app/archive/1_79_903912515.arc                                                                          79 NO  YES YES       NO  A NO  RFS     FGRD
         2 minstd                                                                                                       79 YES YES NO        NO  A NO  FGRD    FGRD
         1 /u01/app/archive/1_80_903912515.arc                                                                          80 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_81_903912515.arc                                                                          81 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_82_903912515.arc                                                                          82 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_83_903912515.arc                                                                          83 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_84_903912515.arc                                                                          84 NO  YES IN-MEMORY NO  A NO  RFS     ARCH
```

3. 此时进行切换

```sql
SQL> SELECT SWITCHOVER_STATUS FROM V$DATABASE;

SWITCHOVER_STATUS
--------------------
TO STANDBY

SQL> ALTER DATABASE COMMIT TO SWITCHOVER TO PHYSICAL STANDBY;

Database altered.
```

>主库的alert日志


```
Tue Feb 16 14:55:15 2016
ALTER DATABASE COMMIT TO SWITCHOVER TO PHYSICAL STANDBY
ALTER DATABASE COMMIT TO SWITCHOVER TO PHYSICAL STANDBY [Process Id: 5441] (minstd)
Waiting for all non-current ORLs to be archived...
All non-current ORLs have been archived.
Waiting for all FAL entries to be archived...
All FAL entries have been archived.
Waiting for potential Physical Standby switchover target to become synchronized...
Active, synchronized Physical Standby switchover target has been identified
Switchover End-Of-Redo Log thread 1 sequence 85 has been fixed
Switchover: Primary highest seen SCN set to 0x0.0x101743
ARCH: Noswitch archival of thread 1, sequence 85
ARCH: End-Of-Redo Branch archival of thread 1 sequence 85
ARCH: LGWR is actively archiving destination LOG_ARCHIVE_DEST_2
ARCH: Standby redo logfile selected for thread 1 sequence 85 for destination LOG_ARCHIVE_DEST_2
Archived Log entry 17 added for thread 1 sequence 85 ID 0x97bd9e7c dest 1:
ARCH: Archiving is disabled due to current logfile archival
Primary will check for some target standby to have received alls redo
Final check for a synchronized target standby. Check will be made once.
LOG_ARCHIVE_DEST_2 is a potential Physical Standby switchover target
Active, synchronized target has been identified
Target has also received all redo
```
>这时主库做了一次日志切换，加入EOR标记，并传输日志到所有备库，然后检查确认所有备库全部接受到所有的redo
>这时查询备库

```sql
SQL> select DEST_ID,NAME ,SEQUENCE#,STANDBY_DEST,ARCHIVED,APPLIED,DELETED,STATUS,FAL,REGISTRAR,CREATOR from v$archived_log  order by 3;

   DEST_ID NAME                                                                                                  SEQUENCE# STA ARC APPLIED   DEL S FAL REGISTR CREATOR
---------- ---------------------------------------------------------------------------------------------------- ---------- --- --- --------- --- - --- ------- -------
         1 /u01/app/archive/1_72_903912515.arc                                                                          72 NO  YES YES       NO  A NO  ARCH    ARCH
         1 /u01/app/archive/1_73_903912515.arc                                                                          73 NO  YES YES       NO  A NO  ARCH    ARCH
         1 /u01/app/archive/1_74_903912515.arc                                                                          74 NO  YES YES       NO  A NO  ARCH    ARCH
         2 minstd                                                                                                       74 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_75_903912515.arc                                                                          75 NO  YES YES       NO  A NO  FGRD    FGRD
         2 minstd                                                                                                       75 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_76_903912515.arc                                                                          76 NO  YES YES       NO  A NO  ARCH    ARCH
         2 minstd                                                                                                       76 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_77_903912515.arc                                                                          77 NO  YES YES       NO  A NO  ARCH    ARCH
         2 minstd                                                                                                       77 YES YES YES       NO  A NO  LGWR    LGWR
         2 minstd                                                                                                       78 YES YES NO        NO  A NO  LGWR    LGWR
         1 /u01/app/archive/1_78_903912515.arc                                                                          78 NO  YES YES       NO  A NO  ARCH    ARCH
         1 /u01/app/archive/1_79_903912515.arc                                                                          79 NO  YES YES       NO  A NO  RFS     FGRD
         2 minstd                                                                                                       79 YES YES NO        NO  A NO  FGRD    FGRD
         1 /u01/app/archive/1_80_903912515.arc                                                                          80 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_81_903912515.arc                                                                          81 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_82_903912515.arc                                                                          82 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_83_903912515.arc                                                                          83 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_84_903912515.arc                                                                          84 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_85_903912515.arc                                                                          85 NO  YES YES       NO  A NO  RFS     ARCH
```

>85号日志确实已经收到并且应用，备库alert如下

```
Tue Feb 16 14:52:22 2016
Archived Log entry 19 added for thread 1 sequence 84 ID 0x97bd9e7c dest 1:
Media Recovery Waiting for thread 1 sequence 85 (in transit)
Recovery of Online Redo Log: Thread 1 Group 4 Seq 85 Reading mem 0
Mem# 0: /u01/app/oradata/min/stdb_redo01.log
Tue Feb 16 14:55:17 2016
RFS[5]: Assigned to RFS process 7837
RFS[5]: Selected log 4 for thread 1 sequence 85 dbid -1749202365 branch 903912515
Tue Feb 16 14:55:17 2016
Archived Log entry 20 added for thread 1 sequence 85 ID 0x97bd9e7c dest 1:
Tue Feb 16 14:55:17 2016
RFS[2]: Possible network disconnect with primary database
Tue Feb 16 14:55:17 2016
RFS[6]: Assigned to RFS process 7782
RFS[6]: Possible network disconnect with primary database
Tue Feb 16 14:55:17 2016
RFS[1]: Possible network disconnect with primary database
Tue Feb 16 14:55:17 2016
RFS[7]: Assigned to RFS process 7835
RFS[7]: Possible network disconnect with primary database
Tue Feb 16 14:55:17 2016
Resetting standby activation ID 2545786492 (0x97bd9e7c)
Media Recovery End-Of-Redo indicator encountered
Media Recovery Continuing
Media Recovery Waiting for thread 1 sequence 86
```

>收到85号日志之后，与主库失去联系，在应用85号日志的时候遇到EOR标记
>备库切换为主库

```sql
SQL> SELECT SWITCHOVER_STATUS FROM V$DATABASE;

SWITCHOVER_STATUS
--------------------
TO PRIMARY

SQL> alter database recover managed standby database cancel;

Database altered.

SQL> ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN;

Database altered.

SQL> alter database open;

Database altered.
```

>新主库alert

```
ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN
ALTER DATABASE SWITCHOVER TO PRIMARY (min)
Maximum wait for role transition is 15 minutes.
All dispatchers and shared servers shutdown
CLOSE: killing server sessions.
CLOSE: all sessions shutdown successfully.
Tue Feb 16 15:00:29 2016
SMON: disabling cache recovery
Backup controlfile written to trace file /home/oracle/app/oracle/diag/rdbms/min/min/trace/min_ora_7726.trc
SwitchOver after complete recovery through change 1054531
Online log /u01/app/oradata/min/redo01.log: Thread 1 Group 1 was previously cleared
Online log /u01/app/oradata/min/redo02.log: Thread 1 Group 2 was previously cleared
Online log /u01/app/oradata/min/redo03.log: Thread 1 Group 3 was previously cleared
Standby became primary SCN: 1054529
AUDIT_TRAIL initialization parameter is changed back to its original value as specified in the parameter file.
Switchover: Complete - Database mounted as primary
Completed: ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN
Tue Feb 16 15:00:42 2016
alter database open
Tue Feb 16 15:00:42 2016
Assigning activation ID 2545831887 (0x97be4fcf)
Thread 1 advanced to log sequence 87 (thread open)
Tue Feb 16 15:00:42 2016
ARC8: Becoming the 'no SRL' ARCH
Thread 1 opened at log sequence 87
Current log# 2 seq# 87 mem# 0: /u01/app/oradata/min/redo02.log
Successful open of redo thread 1
MTTR advisory is disabled because FAST_START_MTTR_TARGET is not set
Tue Feb 16 15:00:42 2016
ARC9: Becoming the 'no SRL' ARCH
Tue Feb 16 15:00:42 2016
ARCa: Becoming the 'no SRL' ARCH
Tue Feb 16 15:00:42 2016
SMON: enabling cache recovery
Archived Log entry 21 added for thread 1 sequence 86 ID 0x97be4fcf dest 1:
Tue Feb 16 15:00:42 2016
ARCt: Becoming the 'no SRL' ARCH
Tue Feb 16 15:00:42 2016
NSA2 started with pid=18, OS id=7850
[7726] Successfully onlined Undo Tablespace 2.
Undo initialization finished serial:0 start:86759444 end:86759474 diff:30 (0 seconds)
Dictionary check beginning
Dictionary check complete
Verifying file header compatibility for 11g tablespace encryption..
Verifying 11g file header compatibility for tablespace encryption completed
SMON: enabling tx recovery
Database Characterset is ZHS16GBK
Starting background process SMCO
Tue Feb 16 15:00:42 2016
idle dispatcher 'D000' terminated, pid = (17, 1)
ARCt: Standby redo logfile selected for thread 1 sequence 86 for destination LOG_ARCHIVE_DEST_2
Tue Feb 16 15:00:42 2016
SMCO started with pid=50, OS id=7852
No Resource Manager plan active
Starting background process QMNC
Tue Feb 16 15:00:42 2016
QMNC started with pid=52, OS id=7856
LOGSTDBY: Validating controlfile with logical metadata
LOGSTDBY: Validation complete
Completed: alter database open
Tue Feb 16 15:00:43 2016
ARC0: Becoming the 'no SRL' ARCH
Tue Feb 16 15:00:43 2016
ARC1: Becoming the 'no SRL' ARCH
ARC0: Archive log rejected (thread 1 sequence 86) at host 'minstd'
FAL[server, ARC0]: FAL archive failed, see trace file.
ARCH: FAL archive failed. Archiver continuing
ORACLE Instance min - Archival Error. Archiver continuing.
Thread 1 advanced to log sequence 88 (LGWR switch)
Current log# 3 seq# 88 mem# 0: /u01/app/oradata/min/redo03.log
Tue Feb 16 15:00:45 2016
ARC2: Becoming the 'no SRL' ARCH
Archived Log entry 23 added for thread 1 sequence 87 ID 0x97be4fcf dest 1:
Tue Feb 16 15:00:45 2016
ARC3: Becoming the 'no SRL' ARCH
ARC3: Standby redo logfile selected for thread 1 sequence 87 for destination LOG_ARCHIVE_DEST_2
******************************************************************
LGWR: Setting 'active' archival for destination LOG_ARCHIVE_DEST_2
******************************************************************
LNS: Standby redo logfile selected for thread 1 sequence 88 for destination LOG_ARCHIVE_DEST_2
Tue Feb 16 15:01:05 2016
ARCr: Becoming the 'no SRL' ARCH
```
>新主库打开的时候向备库传输86号日志被拒绝，这里我猜测是一个gap检测的问题，至于为何报错，暂时没搞清楚，我觉得出现了FAL应该是备库检测到gap来请求日志的，但为何拒绝不甚清楚，而在新备库的日志却发现是接受成功的

```
Tue Feb 16 15:00:42 2016
Using STANDBY_ARCHIVE_DEST parameter default value as /u01/app/archive
RFS[1]: Assigned to RFS process 5901
RFS[1]: Selected log 4 for thread 1 sequence 86 dbid -1749202365 branch 903912515
Tue Feb 16 15:00:42 2016
Archived Log entry 19 added for thread 1 sequence 86 ID 0x97be4fcf dest 1:
Tue Feb 16 15:00:45 2016
RFS[2]: Assigned to RFS process 5905
RFS[2]: Selected log 4 for thread 1 sequence 87 dbid -1749202365 branch 903912515
Tue Feb 16 15:00:45 2016
Archived Log entry 20 added for thread 1 sequence 87 ID 0x97be4fcf dest 1:
Tue Feb 16 15:00:45 2016
Primary database is in MAXIMUM PERFORMANCE mode
RFS[3]: Assigned to RFS process 5907
RFS[3]: Selected log 4 for thread 1 sequence 88 dbid -1749202365 branch 903912515
Tue Feb 16 15:01:36 2016
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE disconnect
Attempt to start background Managed Standby Recovery process (minstd)
Tue Feb 16 15:01:36 2016
MRP0 started with pid=55, OS id=5923
MRP0: Background Managed Standby Recovery process started (minstd)
started logmerger process
```
>可见86，87已经接收到了
>下一刻便开始正常传输了，主库日志信息如下：

```
Tue Feb 16 15:00:45 2016
ARC3: Becoming the 'no SRL' ARCH
ARC3: Standby redo logfile selected for thread 1 sequence 87 for destination LOG_ARCHIVE_DEST_2
******************************************************************
LGWR: Setting 'active' archival for destination LOG_ARCHIVE_DEST_2
******************************************************************
LNS: Standby redo logfile selected for thread 1 sequence 88 for destination LOG_ARCHIVE_DEST_2
Tue Feb 16 15:01:05 2016
ARCr: Becoming the 'no SRL' ARCH
```
>有一个归档目的地2激活的提示，难道是一开始未激活？
>此时再来看一下归档日志信息

**新主库：**

```sql
SQL> select DEST_ID,NAME ,SEQUENCE#,STANDBY_DEST,ARCHIVED,APPLIED,DELETED,STATUS,FAL,REGISTRAR,CREATOR from v$archived_log  order by 3;

   DEST_ID NAME                                                                                                  SEQUENCE# STA ARC APPLIED   DEL S FAL REGISTR CREATOR
---------- ---------------------------------------------------------------------------------------------------- ---------- --- --- --------- --- - --- ------- -------
         1 /u01/app/archive/1_72_903912515.arc                                                                          72 NO  YES YES       NO  A NO  ARCH    ARCH
         1 /u01/app/archive/1_73_903912515.arc                                                                          73 NO  YES YES       NO  A NO  ARCH    ARCH
         2 minstd                                                                                                       74 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_74_903912515.arc                                                                          74 NO  YES YES       NO  A NO  ARCH    ARCH
         1 /u01/app/archive/1_75_903912515.arc                                                                          75 NO  YES YES       NO  A NO  FGRD    FGRD
         2 minstd                                                                                                       75 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_76_903912515.arc                                                                          76 NO  YES YES       NO  A NO  ARCH    ARCH
         2 minstd                                                                                                       76 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_77_903912515.arc                                                                          77 NO  YES YES       NO  A NO  ARCH    ARCH
         2 minstd                                                                                                       77 YES YES YES       NO  A NO  LGWR    LGWR
         2 minstd                                                                                                       78 YES YES YES       NO  A NO  LGWR    LGWR
         1 /u01/app/archive/1_78_903912515.arc                                                                          78 NO  YES YES       NO  A NO  ARCH    ARCH
         1 /u01/app/archive/1_79_903912515.arc                                                                          79 NO  YES YES       NO  A NO  RFS     FGRD
         2 minstd                                                                                                       79 YES YES YES       NO  A NO  FGRD    FGRD
         1 /u01/app/archive/1_80_903912515.arc                                                                          80 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_81_903912515.arc                                                                          81 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_82_903912515.arc                                                                          82 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_83_903912515.arc                                                                          83 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_84_903912515.arc                                                                          84 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_85_903912515.arc                                                                          85 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_86_903912515.arc                                                                          86 NO  YES NO        NO  A NO  ARCH    ARCH
         2 minstd                                                                                                       86 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_87_903912515.arc                                                                          87 NO  YES NO        NO  A NO  ARCH    ARCH
         2 minstd                                                                                                       87 YES YES YES       NO  A YES ARCH    ARCH

24 rows selected.

```
>备库归档目的地86，97两个日志的FAL都是yes，而且在新主库查看切换期间产生的84，85两个日志的applied状态为yes，而在新备库（原主库）查询84，85在新主库（原备库）的applied状态却为NO，很有意思的一个现象

**新备库:**

```sql
SQL> select DEST_ID,NAME ,SEQUENCE#,STANDBY_DEST,ARCHIVED,APPLIED,DELETED,STATUS,FAL,REGISTRAR,CREATOR from v$archived_log  order by 3;

   DEST_ID NAME                                                          SEQUENCE# STA ARC APPLIED   DEL S FAL REGISTR CREATOR
---------- ------------------------------------------------------------ ---------- --- --- --------- --- - --- ------- -------
         2 /u01/app/archive/1_74_903912515.arc                                  74 NO  YES YES       NO  A YES RFS     ARCH
         2 /u01/app/archive/1_75_903912515.arc                                  75 NO  YES YES       NO  A YES RFS     ARCH
         1 /u01/app/archive/1_76_903912515.arc                                  76 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_77_903912515.arc                                  77 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_78_903912515.arc                                  78 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_79_903912515.arc                                  79 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_80_903912515.arc                                  80 NO  YES YES       NO  A NO  ARCH    ARCH
         2 min                                                                  80 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_81_903912515.arc                                  81 NO  YES YES       NO  A NO  ARCH    ARCH
         2 min                                                                  81 YES YES YES       NO  A YES ARCH    ARCH
         1 /u01/app/archive/1_82_903912515.arc                                  82 NO  YES YES       NO  A NO  ARCH    ARCH
         2 min                                                                  82 YES YES YES       NO  A NO  LGWR    LGWR
         1 /u01/app/archive/1_83_903912515.arc                                  83 NO  YES YES       NO  A NO  ARCH    ARCH
         2 min                                                                  83 YES YES YES       NO  A NO  LGWR    LGWR
         1 /u01/app/archive/1_84_903912515.arc                                  84 NO  YES YES       NO  A NO  ARCH    ARCH
         2 min                                                                  84 YES YES NO        NO  A NO  LGWR    LGWR
         1 /u01/app/archive/1_85_903912515.arc                                  85 NO  YES YES       NO  A NO  RFS     FGRD
         2 min                                                                  85 YES YES NO        NO  A NO  FGRD    FGRD
         1 /u01/app/archive/1_86_903912515.arc                                  86 NO  YES YES       NO  A NO  RFS     ARCH
         1 /u01/app/archive/1_87_903912515.arc                                  87 NO  YES YES       NO  A NO  RFS     ARCH

20 rows selected.

```

**总结：**
1. switchover前后每个主库角色都会切换1次日志（本次实验为准，并不绝对）
2. 新主库产生的前2个日志是以FAL方式传输到备库
3. 在原主库查询switcover之前产生的两个日志的applied状态时为NO，而在新的主库（原备库）查询日志的应用状态是为YES的

>关于本文中提到的FAL报错的问题，希望广大DBA朋友帮助解惑，如文中还有其他错误之处，还望批评指正

**********
>后记：在一套新的DG环境下测试切换，并未出现FAL报错的情况，怀疑是和当时备库的状态有关