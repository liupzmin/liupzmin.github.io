---
title: DataGuard配置Broker解决ORA-16664、ORA-16792
tags: DataGuard
categories: oracle
date: 2016-03-22 16:21
---



给某一个客户安装ADG完毕，配置Broker方便日后的管理与切换，碰到些许问题，以作记录

创建完configuration之后，enable的过程很慢，而且状态出现error

主库查询：

```sql
DGMGRL> SHOW CONFIGURATION VERBOSE;
Configuration - anxinconf
Protection Mode: MaxAvailability
Databases:
anxin - Primary database
anxinstd - Physical standby database
Error: ORA-16664: unable to receive the result from a database
Properties:
FastStartFailoverThreshold = '30'
OperationTimeout = '30'
FastStartFailoverLagLimit = '30'
CommunicationTimeout = '180'
ObserverReconnect = '0'
FastStartFailoverAutoReinstate = 'TRUE'
FastStartFailoverPmyShutdown = 'TRUE'
BystandersFollowRoleChange = 'ALL'
ObserverOverride = 'FALSE'
ExternalDestination1 = ''
ExternalDestination2 = ''
PrimaryLostWriteAction = 'CONTINUE'
Fast-Start Failover: DISABLED
Configuration Status:
ERROR
```

备库查询：

```sql
DGMGRL> SHOW CONFIGURATION VERBOSE;
Configuration - anxinconf
Protection Mode: MaxAvailability
Databases:
anxin - Primary database
anxinstd - Physical standby database
Warning: ORA-16792: configurable property value is inconsistent with database setting
Properties:
FastStartFailoverThreshold = '30'
OperationTimeout = '30'
FastStartFailoverLagLimit = '30'
CommunicationTimeout = '180'
ObserverReconnect = '0'
FastStartFailoverAutoReinstate = 'TRUE'
FastStartFailoverPmyShutdown = 'TRUE'
BystandersFollowRoleChange = 'ALL'
ObserverOverride = 'FALSE'
ExternalDestination1 = ''
ExternalDestination2 = ''
PrimaryLostWriteAction = 'CONTINUE'
Fast-Start Failover: DISABLED
Configuration Status:
WARNING
```

查看备数据库的状态


```sql
DGMGRL> show database verbose anxinstd;
Database - anxinstd
Role: PHYSICAL STANDBY
Intended State: APPLY-ON
Transport Lag: 0 seconds (computed 1 second ago)
Apply Lag: 0 seconds (computed 1 second ago)
Apply Rate: 54.00 KByte/s
Real Time Query: ON
Instance(s):
anxinstd
Warning: ORA-16714: the value of property ArchiveLagTarget is inconsistent with the database setting
Warning: ORA-16714: the value of property LogArchiveMinSucceedDest is inconsistent with the database setting
Warning: ORA-16714: the value of property LogArchiveTrace is inconsistent with the database setting
Warning: ORA-16714: the value of property LogArchiveFormat is inconsistent with the database setting

......
```

此时主备alert日志均有报错 `Fatal NI connect error 12514, connecting to:`

主库：
```
(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=192.168.1.101)(PORT=1521)))(CONNECT_DATA=(SERVICE_NAME=anxinstd_DGB)(CID=(PROGRAM=oracle)(HOST=db01)(USER=oracle))))
```

备库：

```
(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=192.168.1.100)(PORT=1521)))(CONNECT_DATA=(SERVICE_NAME=anxin_DGB)(CID=(PROGRAM=oracle)(HOST=db02)(USER=oracle))))
```

备库状态报告4个属性值与数据库设置不一致，重新设置

```sql
DGMGRL> edit database anxinstd set property LogArchiveFormat='%t_%s_%r.arc';
Property "logarchiveformat" updated
DGMGRL> edit database anxinstd set property ArchiveLagTarget=0;
Property "archivelagtarget" updated
DGMGRL> edit database anxinstd set property LogArchiveTrace=0;
Property "logarchivetrace" updated
DGMGRL> edit database anxinstd set property LogArchiveMinSucceedDest=1;
Property "logarchiveminsucceeddest" updated
DGMGRL> show database verbose anxinstd;
```

再次查看broker状态

```sql
DGMGRL> SHOW CONFIGURATION VERBOSE;
Configuration - anxinconf
Protection Mode: MaxAvailability
Databases:
anxin - Primary database
anxinstd - Physical standby database
Properties:
FastStartFailoverThreshold = '30'
OperationTimeout = '30'
FastStartFailoverLagLimit = '30'
CommunicationTimeout = '180'
ObserverReconnect = '0'
FastStartFailoverAutoReinstate = 'TRUE'
FastStartFailoverPmyShutdown = 'TRUE'
BystandersFollowRoleChange = 'ALL'
ObserverOverride = 'FALSE'
ExternalDestination1 = ''
ExternalDestination2 = ''
PrimaryLostWriteAction = 'CONTINUE'
Fast-Start Failover: DISABLED
Configuration Status:
SUCCESS
```

主备状态都正常了

接下来测试切换的时候又出现了问题

```sql
DGMGRL> SWITCHOVER TO anxinstd;
Performing switchover NOW, please wait...
Operation requires a connection to instance "anxinstd" on database "anxinstd"
Connecting to instance "anxinstd"...
Unable to connect to database
ORA-12514: TNS:listener does not currently know of service requested in connect descriptor
Failed.
Warning: You are no longer connected to ORACLE.
connect to instance "anxinstd" of database "anxinstd"
```

**官方文档中有如下描述：**

```
After performing a switchover using DGMGRL, Data Guard requires a shutdown and startup of both the primary and standby databases. This issue can occur if any necessary entry is missing in the listener.ora file.
DGMGRL is unable to connect to the database after it has been stopped while performing the switchover

To enable DGMGRL to restart instances during the course of broker operations, a service with a specific name must be statically registered with the listener of each instance. 
The value for the GLOBAL_DBNAME attribute must be set to a concatenation of db_unique_name_DGMGRL.db_domain in the LISTENER.ORA file.
```

看一下StaticConnectIdentifier的值

```sql
DGMGRL> show database anxin StaticConnectIdentifier;
StaticConnectIdentifier = '(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=db01)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=anxin_DGMGRL)(INSTANCE_NAME=anxin)(SERVER=DEDICATED)))'
DGMGRL> show database anxinstd StaticConnectIdentifier;
StaticConnectIdentifier = '(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=db02)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=anxinstd_DGMGRL)(INSTANCE_NAME=anxinstd)(SERVER=DEDICATED)))'
```

均去连接一个db_unique_name_DGMGRL.db_domain格式的服务，那么需要在监听里静态注册一个db_unique_name_DGMGRL.db_domain的服务

主：

```
xin =
(DESCRIPTION_LIST =
(DESCRIPTION =
(ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.1.100)(PORT = 1521))
)
)
SID_LIST_xin=
(SID_LIST=
(SID_DESC=
(GLOBAL_DBNAME=anxin)
(ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1)
(SID_NAME=anxin)
)
(SID_DESC=
(GLOBAL_DBNAME=anxin_DGMGRL)
(ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1)
(SID_NAME=anxin)
)
)
)
ADR_BASE_LISTENER = /u01/app/oracle
```

备：
```
xin =
(DESCRIPTION_LIST =
(DESCRIPTION =
(ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.1.101)(PORT = 1521))
)
)
SID_LIST_xin=
(SID_LIST=
(SID_DESC=
(GLOBAL_DBNAME=anxinstd)
(ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1)
(SID_NAME=anxinstd)
)
(SID_DESC=
(GLOBAL_DBNAME=anxinstd_DGMGRL)
(ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1)
(SID_NAME=anxinstd)
)
)
)
ADR_BASE_LISTENER = /u01/app/oracle
```

再次测试，问题依旧，莫非和之前alert里的 Fatal NI connect error 12514报错有关？错误信息里的链接描述符均去请求一个db_unique_name_DGB.db_domain的服务，那么再依样添加到静态注册里试试

```sql
DGMGRL> switchover to anxinstd;
Performing switchover NOW, please wait...
Operation requires a connection to instance "anxinstd" on database "anxinstd"
Connecting to instance "anxinstd"...
Connected.
New primary database "anxinstd" is opening...
Operation requires startup of instance "anxin" on database "anxin"
Starting instance "anxin"...
ORACLE instance started.
Database mounted.
Database opened.
Switchover succeeded, new primary is "anxinstd"
```

成功了，那么这个db_unique_name_DGB.db_domain的服务究竟是做什么用的呢？

{db_unique_name}_DGB.{db_domain}: This Service is used by the DMON-Processes to communicate between each other
DMON是一个用来管理broker的后台进程，这个进程负责与本地数据库以及远程数据库的DMON进程进行通讯（与远端数据库的DMON进程进行通讯的时候使用的是一个 动态注册 的service name “db_unique_name_DGB.db_domain”）

既然是动态注册，那缘何注册失败呢？
文档 ID 365314.1给出了答案：**Database Will Not Register With Listener configured on IP instead of Hostname**

将主备的{db_unique_name}_DGB.{db_domain}静态entry删掉，host采用hostname，重启监听测试，switchover成功。

由此可见监听配置里还是采用hostname为好，通过本次事件也解惑了萦绕我心头很久的问题，很多时候建库完毕，使用工具创建动态注册的监听，监听状态里会有很多XDB之类的服务，而我改成静态监听之后（每次都用IP）却没有了之前的自动注册的服务，可见这就是根本原因：`Database Will Not Register With Listener configured on IP instead of Hostname`