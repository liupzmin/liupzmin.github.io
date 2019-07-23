---
date: 2016-05-27 18:22:59
categories: oracle
tags: DataGuard
title: DG Broker管理dataguard注意事项
---


`Once you start using the Broker you must always use the Broker to make any changes to your Data Guard configuration. This means that you must use Grid Control or the Broker CLI DGMGRL to change any Data Guard settings. If you use SQL*Plus to make configuration changes, the Broker will put things back the way it sees the world or this will lead to inconsistencies between the Broker configuration parameters and the database.`

一旦开始使用Broker，就必须一直使用它修改DG的配置，这意味着必须使用grid control或者dgmgrl修改DG的配置，如果使用sqlplus修改配置，broker会将配置改回原样，或者导致broker配置参数与数据库之间不一致

>此例记录修改保护模式，由最高可用降级为最大性能

```sql
DGMGRL> show configuration;
Configuration - minconf
Protection Mode: MaxAvailability
Databases:
minstd - Primary database
min - Physical standby database
Fast-Start Failover: DISABLED
Configuration Status:
SUCCESS
DGMGRL> EDIT DATABASE 'minstd' SET PROPERTY 'LogXptMode'='ASYNC';
Property "LogXptMode" updated
DGMGRL> EDIT DATABASE 'min' SET PROPERTY 'LogXptMode'='ASYNC';
Error: ORA-16805: change of LogXptMode property violates overall protection mode
Failed.
DGMGRL> EDIT CONFIGURATION SET PROTECTION MODE AS MaxPerformance;
Succeeded.
DGMGRL> show configuration;
Configuration - minconf
Protection Mode: MaxPerformance
Databases:
minstd - Primary database
min - Physical standby database
Fast-Start Failover: DISABLED
Configuration Status:
SUCCESS
````

使用broker修改会使LOG_ARCHIVE_DEST_2参数变得较为复杂，例如

```
service="minstd", LGWR SYNC AFFIRM delay=0 optional compression=disable max_failure=0 max_connections=1 reopen=300 db_unique_name="minstd" net_timeout=30, valid_for=(all_logfiles,primary_role)
```

而且修改后在sqlplus中show parameter LOG_ARCHIVE_DEST_2仅有主库显示更改之后的状态，而备库还保留原样，但是在show database verbose中是修改好的，仅当在发生切换时，备库变为主库时才会更正参数，可见最好使用broker来管理配置
