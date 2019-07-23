---
title: DataGuard ORA-16825、ORA-16661解决一例
date: 2015-12-31 16:02:08
tags: DataGuard
categories: oracle
---

某天凌晨5点左右，被客户方运维人员电话吵醒，客户的交换机因故重启，导致做了RHCS的两台Linux服务器（非oracle服务器）因心跳问题也导致重启

第一时间登上去查看oracle运行情况,发现主库已是mount状态，备库已变为主库，可以肯定是因为心跳问题导致了自动切换（生产上的DG环境配置了Fast-Start Failover）

BROKER连上去查看状态

```sql
DGMGRL> SHOW CONFIGURATION VERBOSE;
Configuration - jsconf
Protection Mode: MaxAvailability
Databases:
jsread - Primary database
Error: ORA-16825: multiple errors or warnings, including fast-start failover-related errors or warnings, detected for the database
  
  jsrw   - (*) Physical standby database (disabled)
      ORA-16661: the standby database needs to be reinstated


  (*) Fast-Start Failover target


  Properties:
    FastStartFailoverThreshold      = '30'
    OperationTimeout                = '30'
    FastStartFailoverLagLimit       = '30'
    CommunicationTimeout            = '180'
    ObserverReconnect               = '0'
    FastStartFailoverAutoReinstate  = 'TRUE'
    FastStartFailoverPmyShutdown    = 'TRUE'
    BystandersFollowRoleChange      = 'ALL'
    ObserverOverride                = 'FALSE'
    ExternalDestination1            = ''
    ExternalDestination2            = ''
    PrimaryLostWriteAction          = 'CONTINUE'


Fast-Start Failover: ENABLED


  Threshold:          30 seconds
  Target:             jsrw
  Observer:           CAHW50
  Lag Limit:          30 seconds (not in use)
  Shutdown Primary:   TRUE
  Auto-reinstate:     TRUE
  Observer Reconnect: (none)
  Observer Override:  FALSE


Configuration Status:
ERROR
```

查看主备库的状态

```sql
DGMGRL> show database jsrw


Database - jsrw


  Role:            PHYSICAL STANDBY
  Intended State:  APPLY-ON
  Transport Lag:   (unknown)
  Apply Lag:       (unknown)
  Apply Rate:      (unknown)
  Real Time Query: OFF
  Instance(s):
    jsrw


Database Status:
ORA-16661: the standby database needs to be reinstated


DGMGRL> show database jsread


Database - jsread


  Role:            PRIMARY
  Intended State:  TRANSPORT-ON
  Instance(s):
    jsread


  Database Error(s):
    ORA-16820: fast-start failover observer is no longer observing this database


  Database Warning(s):
    ORA-16817: unsynchronized fast-start failover configuration


Database Status:
ERROR
```

看样子原主库已被隔离，查看alert日志

原主库日志：

```
Tue Dec 15 04:14:45 2015
ORA-16198: LGWR received timedout error from KSR
LGWR: Attempting destination LOG_ARCHIVE_DEST_2 network reconnect (16198)
LGWR: Destination LOG_ARCHIVE_DEST_2 network reconnect abandoned
Error 16198 for archive log file 6 to 'jsread'
ORA-16198: LGWR received timedout error from KSR
LGWR: Error 16198 disconnecting from destination LOG_ARCHIVE_DEST_2 standby host 'jsread'
Destination LOG_ARCHIVE_DEST_2 is UNSYNCHRONIZED
Primary has heard from neither observer nor target standby within FastStartFailoverThreshold seconds.
It is likely an automatic failover has already occurred. Primary is shutting down.
Errors in file /orasystem_readwrite/oracle/oraWR/diag/rdbms/jsrw/jsrw/trace/jsrw_lgwr_73912.trc:
ORA-16830: primary isolated from fast-start failover partners longer than FastStartFailoverThreshold seconds: shutting down
```

日志中可以看到，主库失去了备库和observer的链接，已超时30秒，因此主库猜测此时极有可能已经发生了自动故障转移，然后将自己关掉

原备库日志：

```
Tue Dec 15 04:14:47 2015
RFS[44]: Assigned to RFS process 41247
RFS[44]: Possible network disconnect with primary database
Tue Dec 15 04:14:47 2015
RFS[40]: Possible network disconnect with primary database
Tue Dec 15 04:14:47 2015
RFS[42]: Possible network disconnect with primary database
Tue Dec 15 04:14:50 2015
Attempting Fast-Start Failover because the threshold of 30 seconds has elapsed.
Tue Dec 15 04:14:50 2015
Data Guard Broker: Beginning failover
Tue Dec 15 04:14:50 2015
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL
Tue Dec 15 04:14:50 2015
MRP0: Background Media Recovery cancelled with status 16037
Errors in file /orasystem_readonly/oracle/oraread/diag/rdbms/jsread/jsread/trace/jsread_pr00_59244.trc:
ORA-16037: user requested cancel of managed recovery operation
Managed Standby Recovery not using Real Time Apply
Tue Dec 15 04:14:50 2015
ALTER SYSTEM SET service_names='jsread' SCOPE=MEMORY SID='jsread';
Recovery interrupted!
Recovered data files to a consistent state at change 229593026
Tue Dec 15 04:14:51 2015
MRP0: Background Media Recovery process shutdown (jsread)
Managed Standby Recovery Canceled (jsread)
Completed: ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE FINISH FORCE
Attempt to do a Terminal Recovery (jsread)
Media Recovery Start: Managed Standby Recovery (jsread)
started logmerger process
Tue Dec 15 04:14:51 2015
Managed Standby Recovery not using Real Time Apply
Parallel Media Recovery started with 64 slaves
Media Recovery Waiting for thread 1 sequence 3176 (in transit)
Killing 1 processes with pids 41252 (all RFS, wait for I/O) in order to disallow current and future RFS connections. Requested by OS process 130750
Begin: Standby Redo Logfile archival
End: Standby Redo Logfile archival
Terminal Recovery timestamp is '12/15/2015 04:14:57'
Terminal Recovery: applying standby redo logs.
Terminal Recovery: thread 1 seq# 3176 redo required
Terminal Recovery:
Recovery of Online Redo Log: Thread 1 Group 9 Seq 3176 Reading mem 0
Mem# 0: /oradata_readonly/jsread/stb_redo02.log
Identified End-Of-Redo (failover) for thread 1 sequence 3176 at SCN 0xffff.ffffffff
Incomplete Recovery applied until change 229593027 time 12/15/2015 04:14:14
Media Recovery Complete (jsread)
Terminal Recovery: successful completion
Forcing ARSCN to IRSCN for TR 0:229593027
Tue Dec 15 04:14:57 2015
ARCH: Archival stopped, error occurred. Will continue retrying
Attempt to set limbo arscn 0:229593027 irscn 0:229593027
ORACLE Instance jsread - Archival ErrorResetting standby activation ID 486853183 (0x1d04ca3f)
ORA-16014: log 9 sequence# 3176 not archived, no available destinations
ORA-00312: online log 9 thread 1: '/oradata_readonly/jsread/stb_redo02.log'
Completed: ALTER DATABASE RECOVER MANAGED STANDBY DATABASE FINISH FORCE
ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WAIT WITH SESSION SHUTDOWN
ALTER DATABASE SWITCHOVER TO PRIMARY (jsread)
Maximum wait for role transition is 15 minutes.
All dispatchers and shared servers shutdown
CLOSE: killing server sessions.
Active process 117884 user 'grid' program 'oracle@jsc_db_r (TNS V1-V3)'
Tue Dec 15 04:15:00 2015
Active process 117884 user 'grid' program 'oracle@jsc_db_r (TNS V1-V3)'
CLOSE: all sessions shutdown successfully.
Tue Dec 15 04:15:01 2015
SMON: disabling cache recovery
Backup controlfile written to trace file /orasystem_readonly/oracle/oraread/diag/rdbms/jsread/jsread/trace/jsread_rsm0_79936.trc
Standby terminal recovery start SCN: 229593026
RESETLOGS after incomplete recovery UNTIL CHANGE 229593027
Online log /oradata_readonly/jsread/redo01.log: Thread 1 Group 1 was previously cleared
Online log /oradata_readonly/jsread/redo02.log: Thread 1 Group 2 was previously cleared
Online log /oradata_readonly/jsread/redo03.log: Thread 1 Group 3 was previously cleared
Online log /oradata_readonly/jsread/redo04.log: Thread 1 Group 4 was previously cleared
Online log /oradata_readonly/jsread/redo05.log: Thread 1 Group 5 was previously cleared
Online log /oradata_readonly/jsread/redo06.log: Thread 1 Group 6 was previously cleared
Online log /oradata_readonly/jsread/redo07.log: Thread 1 Group 7 was previously cleared
Online log /oradata_readonly/jsread/redo08.log: Thread 1 Group 16 was previously cleared
Online log /oradata_readonly/jsread/redo09.log: Thread 1 Group 17 was previously cleared
Online log /oradata_readonly/jsread/redo10.log: Thread 1 Group 18 was previously cleared
Standby became primary SCN: 229593025
Tue Dec 15 04:15:09 2015
Setting recovery target incarnation to 5
AUDIT_TRAIL initialization parameter is changed back to its original value as specified in the parameter file.
Switchover: Complete - Database mounted as primary
Completed: ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WAIT WITH SESSION SHUTDOWN
ALTER DATABASE SET STANDBY DATABASE TO MAXIMIZE AVAILABILITY
Completed: ALTER DATABASE SET STANDBY DATABASE TO MAXIMIZE AVAILABILITY
ALTER DATABASE OPEN
Data Guard Broker initializing...
Tue Dec 15 04:15:09 2015
Assigning activation ID 488981393 (0x1d254391)
LGWR: Primary database is in MAXIMUM AVAILABILITY mode
Destination LOG_ARCHIVE_DEST_2 is UNSYNCHRONIZED
LGWR: Destination LOG_ARCHIVE_DEST_1 is not serviced by LGWR
Thread 1 advanced to log sequence 2 (thread open)
Tue Dec 15 04:15:09 2015
ARC2: Becoming the 'no SRL' ARCH
Tue Dec 15 04:15:09 2015
ARC3: Becoming the 'no SRL' ARCH
Thread 1 opened at log sequence 2
Current log# 2 seq# 2 mem# 0: /oradata_readonly/jsread/redo02.log
Successful open of redo thread 1
MTTR advisory is disabled because FAST_START_MTTR_TARGET is not set
ARC2: Becoming the 'no SRL' ARCH
SMON: enabling cache recovery
Archived Log entry 3278 added for thread 1 sequence 1 ID 0x1d254391 dest 1:
Archiver process freed from errors. No longer stopped
Tue Dec 15 04:15:10 2015
PING[ARC0]: Heartbeat failed to connect to standby 'jsrw'. Error is 16058.
[79936] Successfully onlined Undo Tablespace 2.
Undo initialization finished serial:0 start:1846831906 end:1846832126 diff:220 (2 seconds)
Dictionary check beginning
PING[ARC0]: Heartbeat failed to connect to standby 'jsrw'. Error is 16058.
Dictionary check complete
Verifying file header compatibility for 11g tablespace encryption..
Verifying 11g file header compatibility for tablespace encryption completed
SMON: enabling tx recovery
Starting background process SMCO
Database Characterset is ZHS16GBK
Tue Dec 15 04:15:10 2015
idle dispatcher 'D000' terminated, pid = (24, 1)
Tue Dec 15 04:15:10 2015
SMCO started with pid=24, OS id=131236
No Resource Manager plan active
Tue Dec 15 04:15:11 2015
Starting background process QMNC
Tue Dec 15 04:15:11 2015
QMNC started with pid=35, OS id=131244
LOGSTDBY: Validating controlfile with logical metadata
LOGSTDBY: Validation complete
Completed: ALTER DATABASE OPEN
```

备库确实在失去主库之后开始自动故障转移，并最终转移成功。

这里要说一下，按理交换机恢复之后DG检测到原主库会自动将其转换为备库，但这次显然没有，还记得有两个配置了RHCS的服务器因网络问题重启了，而客户的observer刚好部署在上面，登上去查看甚至没有找到observer的日志，为什么observer没有打日志出来的？

尝试开启observer，重启原主库，寄希望于observer自动将失败的主库转为备库，但是在start observer的时候就遇到了问题

```sql
DGMGRL> start observer
Error: ORA-16647: could not start more than one observer
Failed.
DGMGRL> stop observer
Error:
ORA-01034: ORACLE not available
Process ID: 0
Session ID: 0 Serial number: 0
```

查看observer日志：

```
Observer started
[W000 12/15 05:30:13.80] Observer started.
Observer stopped
Error:
ORA-01034: ORACLE not available
Process ID: 0
Session ID: 0 Serial number: 0
[W000 12/15 05:32:05.86] Failed to start the Observer.
Error: ORA-16647: could not start more than one observer
[W000 12/15 05:33:40.06] Failed to start the Observer.
```

既无法停止，也无法启动，猜测应该是服务器重启导致的状态错乱，此时情况变 的复杂，因交换机重启导致网络不通，导致快速故障转移，推测转移工作未完全结束，observer的服务器又被重启，使得DG处在一个错误的状态下，
现在已无法通过observer自动恢复失败的主库，只能通过手动解决，而日志里确认新主库已转移成功，那么恢复失败的主库即可，那么如何恢复呢？有很多选择重建？利用flashback databse？或者利用broker的reinstate？
其实reinstate也是使用的flashback，那么干脆使用简单的reinstate吧

```sql
DGMGRL> reinstate database jsrw;
Reinstating database "jsrw", please wait...
Operation requires shutdown of instance "jsrw" on database "jsrw"
Shutting down instance "jsrw"...
ORA-01109: database not open
Database dismounted.
ORACLE instance shut down.
Operation requires startup of instance "jsrw" on database "jsrw"
Starting instance "jsrw"...
ORACLE instance started.
Database mounted.
Continuing to reinstate database "jsrw" ...
Reinstatement of database "jsrw" succeeded
```

查看状态

```sql
DGMGRL> show configuration verbose;


Configuration - jsconf


  Protection Mode: MaxAvailability
  Databases:
    jsread - Primary database
      Error: ORA-16820: fast-start failover observer is no longer observing this database


    jsrw   - (*) Physical standby database
      Error: ORA-16820: fast-start failover observer is no longer observing this database


  (*) Fast-Start Failover target


  Properties:
    FastStartFailoverThreshold      = '30'
    OperationTimeout                = '30'
    FastStartFailoverLagLimit       = '30'
    CommunicationTimeout            = '180'
    ObserverReconnect               = '0'
    FastStartFailoverAutoReinstate  = 'TRUE'
    FastStartFailoverPmyShutdown    = 'TRUE'
    BystandersFollowRoleChange      = 'ALL'
    ObserverOverride                = 'FALSE'
    ExternalDestination1            = ''
    ExternalDestination2            = ''
    PrimaryLostWriteAction          = 'CONTINUE'


Fast-Start Failover: ENABLED


  Threshold:          30 seconds
  Target:             jsrw
  Observer:           CAHW50
  Lag Limit:          30 seconds (not in use)
  Shutdown Primary:   TRUE
  Auto-reinstate:     TRUE
  Observer Reconnect: (none)
  Observer Override:  FALSE


Configuration Status:
ERROR
```

可见已经恢复成功，此时stop observer，start observer就没有问题了，但为了保险起见，停止了observer

```sql
DGMGRL> stop observer;
Done.
DGMGRL> start observer;
Observer started
```

这次显然是交换机导致的乌龙failover，根本不需要failover的，本来的高可用措施反而带来了些许麻烦，后来找机会又切换回原来的主库了，自动故障转移也一直没有再启动，届时手动faiover更为灵活，因为客户的应用并不够灵活.