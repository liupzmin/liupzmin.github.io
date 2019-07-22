---
date: 2017-07-01 11:04
tags: oracle backup
title: 利用DBMS_SCHEDULER在Oracle 11gR2 RAC上执行rman备份
categories: oracle backup
---

# 利用DBMS_SCHEDULER在Oracle 11gR2 RAC上执行rman备份

RMAN backup in Oracle 11gR2 RAC is exactly same like RMAN backup in Oracle 11gR2 single node.
The only difference is: Typically, in case of Oracle single node database, we will schedule RMAN scripts with the help of CRON job and it will run according to our convenience, but in case of Oracle RAC if we schedule RMAN script and if unfortunately that RAC node goes down ( where we configured RMAN scripts ), then RMAN backup won’t run obviously.

So, Same strategy will not be work in Oracle RAC node. For RMAN consistent backups use dbms_scheduler & we need to place RMAN scripts in shared directory. ( Or in my case, I have created identical scripts on both cluster node’s )

**注意：**  *需要将脚本放在共享位置或者每个节点的相同位置*

>看一下rman的备份脚本，此脚本将备份放在ASM中，将日志放在节点本地

```shell
################################################################
#    rman_backup_rac.sh FOR RAC                                #
#    created by lp                                             #
#    2017/03/22                                                #
#    usage: rman_backup_rac.sh <$BACKUP_LEVEL>                 #
#           BACKUP_LEVEL:                                      #
#              F: full backup                                  #
#              0: level 0                                      #
#              1: level 1                                      #
################################################################


#!/bin/bash
# User specific environment and startup programs
export ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1
export RMAN_BAK_LOG_BASE=/home/oracle/DbBackup
export RMAN_BAK_DATA_BASE=+data/NXRAC/backup
export ORACLE_SID=`ps -ef|grep pmon|grep -v ASM|grep -v grep|awk -F'_' '{print $NF}'`
export TIMESTAMP=`date +%Y%m%d%H%M`;

#the destination of rman backuppiece
export RMAN_DATA=${RMAN_BAK_DATA_BASE}/rman

#the destination of rman backup logs
export RMAN_LOG=${RMAN_BAK_LOG_BASE}/logs


if [[ ! -z $1 ]] && echo $1 |grep -Ew "[01F]" >/dev/null 2>&1
then
                                export RMAN_LEVEL=${1}

                                # Check rman level
                                if [ "$RMAN_LEVEL" == "F" ];
                                        then  unset INCR_LVL
                                        BACKUP_TYPE=full
                                        else
                                        INCR_LVL="INCREMENTAL LEVEL ${RMAN_LEVEL}"
                                        BACKUP_TYPE=lev${RMAN_LEVEL}
                                fi
else
                                echo "${1} wrong argument!" >${RMAN_LOG}/wrong_argument_${TIMESTAMP}.log
                                exit 1
fi





#the prefix of rman backuppiece                                 
export RMAN_FILE=${RMAN_DATA}/${BACKUP_TYPE}_${TIMESTAMP}

#the logfile of shell script including the rman logs contents
export SSH_LOG=${RMAN_LOG}/${BACKUP_TYPE}_${TIMESTAMP}.log

#the size of backuppiece
export MAXPIECESIZE=4G

####################################################################
#                                                                  #
#   the name of rman logs excluding the file expanded-name,        #
#   when the shell is complete,the content of this file will be    #
#   appended to the $SSH_LOG and the rman logfile will be deleted. #
#                                                                  #
####################################################################
export RMAN_LOG_FILE=${RMAN_LOG}/${BACKUP_TYPE}_${TIMESTAMP}_1


#Check RMAN Backup Path

if ! test -d ${RMAN_LOG}
then
mkdir -p ${RMAN_LOG}
fi

echo "---------------------------------" >>${SSH_LOG}
echo "   " >>${SSH_LOG}
echo "Rman Begin  to Working ........." >>${SSH_LOG}
echo "Begin time at:" `date` --`date +%Y%m%d%H%M` >>${SSH_LOG}

#Startup rman to backup

$ORACLE_HOME/bin/rman log=${RMAN_LOG_FILE}.log <<EOF
connect target /
run {
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
CONFIGURE BACKUP OPTIMIZATION ON;
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE DEVICE TYPE DISK PARALLELISM 4 BACKUP TYPE TO BACKUPSET;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '${RMAN_DATA}/control_auto_%F';
ALLOCATE CHANNEL 'ch1' TYPE DISK maxpiecesize=${MAXPIECESIZE} CONNECT 'SYS/passwd@node1';
ALLOCATE CHANNEL 'ch2' TYPE DISK maxpiecesize=${MAXPIECESIZE} CONNECT 'SYS/passwd@node1';
ALLOCATE CHANNEL 'ch3' TYPE DISK maxpiecesize=${MAXPIECESIZE} CONNECT 'SYS/passwd@mode2';
ALLOCATE CHANNEL 'ch4' TYPE DISK maxpiecesize=${MAXPIECESIZE} CONNECT 'SYS/passwd@node2';
CROSSCHECK ARCHIVELOG ALL;
DELETE NOPROMPT OBSOLETE;
DELETE NOPROMPT EXPIRED BACKUP;
BACKUP AS COMPRESSED BACKUPSET
${INCR_LVL}
DATABASE FORMAT '${RMAN_FILE}_db_%U' TAG '${BACKUP_TYPE}_${TIMESTAMP}';
SQL 'ALTER SYSTEM ARCHIVE LOG CURRENT';
BACKUP FILESPERSET 20 ARCHIVELOG ALL FORMAT '${RMAN_FILE}_arc_%U' TAG '${ORACLE_SID}_arc_${TIMESTAMP}'
DELETE  INPUT;  
RELEASE CHANNEL ch1;
RELEASE CHANNEL ch2;
RELEASE CHANNEL ch3;
RELEASE CHANNEL ch4;
ALLOCATE CHANNEL ch00 TYPE DISK;
BACKUP
    FORMAT '${RMAN_DATA}/cntrl_%U'
    CURRENT CONTROLFILE;
RELEASE CHANNEL ch00;
}
exit;
EOF

RC=$?

cat ${RMAN_LOG_FILE}.log >>${SSH_LOG}
echo "Rman Stop working @ time:"`date` `date +%Y%m%d%H%M` >>${SSH_LOG}

if [ $RC -ne "0" ]; then
    echo "------ error ------" >>${SSH_LOG}
else
    echo "------ Success during RMAN backup peroid------" >>${SSH_LOG}
    rm -rf ${RMAN_LOG_FILE}.log
fi

exit
```

**DBMS_SCHEDULER:**

Here we are using DBMS_SCHEDULER instead of DBMS_JOB, because DBMS_SCHEDULER is RAC aware.

**Before jump into real DBMS_SCHEDULER configuration, we need to focus on an important thing, That:**

Both RAC nodes local time zone must be identical with DBMS_SCHEDULER default time.

On all RAC node, Ensure local time zone and set it accordingly.

```shell
[oracle@node2 ]$ cat /etc/sysconfig/clock
ZONE="Asia/Shanghai"
```
**configure default time zone for DBMS_SCHEDULER**

```SQL
SQL> select value from dba_scheduler_global_attribute where attribute_name = 'DEFAULT_TIMEZONE';

VALUE
--------------------------------------------------------------------------------
PRC

SQL> exec dbms_scheduler.set_scheduler_attribute ('DEFAULT_TIMEZONE', 'Asia/Shanghai');

PL/SQL procedure successfully completed.

SQL> select value from dba_scheduler_global_attribute where attribute_name = 'DEFAULT_TIMEZONE';

VALUE
--------------------------------------------------------------------------------
Asia/Shanghai

```

Now we need to create credential so that are assigned to DBMS_SCHEDULER jobs so that they can authenticate with a local/remote host operating system or a remote Oracle database.

```SQL
SQL> exec dbms_scheduler.create_credential(credential_name => 'oracle', username => 'oracle', password => 'oracle');

PL/SQL procedure successfully completed.
```

Now its time to create DBMS_SCHEDULER job for RMAN incremental level 0 backup, Here in this procedure I am going to create *RMAN_INC0_BACKUP* job with required attributes.

```SQL
begin
dbms_scheduler.create_job(
job_name            => 'RMAN_INC0_BACKUP',
job_type            => 'EXECUTABLE',
job_action          => '/bin/sh',
number_of_arguments => 2,
start_date          => SYSTIMESTAMP,
credential_name     => 'oracle',
auto_drop           => FALSE,
enabled             => FALSE);
end;
/
```
Set argument_position & argument_value ( i.e. Path of the RMAN script ) for the same job:

```SQL
begin
dbms_scheduler.set_job_argument_value(
job_name            => 'RMAN_INC0_BACKUP',
argument_position   =>  1,
argument_value      => '/home/oracle/rman.sh');
end;
/

begin
dbms_scheduler.set_job_argument_value(
job_name            => 'RMAN_INC0_BACKUP',
argument_position   =>  2,
argument_value      => 0);
end;
/

```


Set start_date for the same job, In my case *RMAN_INC0_BACKUP* job will execute every week on sunday @03am, so job start date and its first run timing would  according to my convenience.
```SQL
begin
dbms_scheduler.set_attribute(
name      => 'RMAN_INC0_BACKUP',
attribute => 'start_date',
value     => trunc(sysdate)+3/24);
end;
/
```

Test your backup job manually in SQL prompt by instantiating *RMAN_INC0_BACKUP* job.

```SQL
SQL> exec dbms_scheduler.run_job('RMAN_INC0_BACKUP');
PL/SQL procedure successfully completed.
```

Verify running RMAN backup status by issuing following SQL query, It will show you RMAN backup details with start time & end time.

```SQL
select SESSION_KEY, INPUT_TYPE, STATUS,
to_char(START_TIME,'mm/dd/yy hh24:mi') start_time,
to_char(END_TIME,'mm/dd/yy hh24:mi') end_time,
elapsed_seconds/3600 hrs
from V$RMAN_BACKUP_JOB_DETAILS
order by session_key;
```

In case of any error while test run, you can make sure details of error by issuing the following query, OR You can also query to *dba_scheduler_job_run_details* dictionary view for more details.

```SQL
select JOB_NAME,STATUS,STATE,ERROR#,CREDENTIAL_NAME from dba_scheduler_job_run_details where CREDENTIAL_NAME like 'RMAN%';
```

After successfully completion of test run, Enable & schedule it by following procedure by setting value to *repeat_interval* parameter, In my case *RMAN_INC0_BACKUP* job will execute every week on Sunday @03pm.

```SQL
begin
dbms_scheduler.set_attribute(
name      => 'RMAN_INC0_BACKUP',
attribute => 'repeat_interval',
value     => 'freq=daily;byday=sun;byhour=03');
dbms_scheduler.enable( 'RMAN_INC0_BACKUP' );
end;
/
```

Ensure dbms_scheduler job details by issuing the following query OR you can also query to *dba_scheduler_jobs* and *dba_scheduler_job_args*.

```SQL
SQL> select job_name,enabled,owner, state from dba_scheduler_jobs where job_name in ('RMAN_INC0_BACKUP');
```

Keep your eye on behavior of dbms_scheduler job by issuing the following query:

```SQL
SQL> select job_name,RUN_COUNT,LAST_START_DATE,NEXT_RUN_DATE from dba_scheduler_jobs where job_name in ('RMAN_INC0_BACKUP');
SQL> select *  from dba_scheduler_job_args where job_name like 'RMAN%';
```
In accordance with the above method to create a level 1 backup job *RMAN_INC1_BACKUP*，The only difference is the *repeat_interval*：

```SQL
begin
dbms_scheduler.set_attribute(
name      => 'RMAN_INC1_BACKUP',
attribute => 'repeat_interval',
value     => 'freq=daily;byday=mon,tue,wed,thu,fri,sat;byhour=03');
dbms_scheduler.enable( 'RMAN_INC1_BACKUP' );
end;
/
```

**Important Note:**
DBMS_SCHEDULER is smart enough to start backup on the node where the last backup was successfully executed.

**参考：**

<https://dbatricksworld.com/how-to-backup-oracle-rac-11gr2-database-with-rman-backup-utility-with-the-help-of-dbms_scheduler-part-i-rman-full-database-backup/>

<http://dbatricksworld.com/how-to-backup-oracle-rac-11gr2-database-with-rman-backup-utility-with-the-help-of-dbms_scheduler-part-ii-rman-incremental-database-backup/>
