---
date: 2017-05-04 21:29
categories: oracle
tags: SQL Plan Management
title: SQL Plan Management（SPM）in oracle 11g & 12c
---

>本文主要介绍SPM的原理与运行机制，针对12c做一些补充记录，并添加一些11g和12c里面的知识点，以做备忘。

### 干预SQL执行计划的历史

  SQL的执行效率，取决于它的执行计划是否高效。 优化器的算法是一个平衡，需要收集尽量少的信息，用尽量快的速度试图去得到一个最优的执行计划，这也决定了它不是万能的。 所以Oracle提供了一些辅助手段来“修复”优化器可能产生的错误，并不断改进这些方法。 

- Oracle 8： hint
- Oracle 8i&9i: stored outline
- Oracle 10g: sql profile
- Oracle 11g: sql plan manangement 、adaptive cursor sharing
- Oracle 12c: sql plan manangement 、adaptive cursor sharing、Adaptive Execution Plans

### SQL Plan Management

#### SPM组成

  SQL Plan Management（以下简称SPM）是一种优化器自动管理执行计划的预防机制，它确保数据库仅使用已知的、经过验证的执行计划。在Oracle 11g之前，执行计划一直是作为“运行时”生成的对象存在。虽然oracle提供了一些方法去指导它的生成，但Oracle一直没有试图去保存完整的执行计划。 从11g开始，执行计划就可以作为一类资源被保存下来，允许特定SQL语句只能选择“已知”的执行计划。

  同其他方法相比，SPM更加的灵活。如我们所熟知的，一条带有绑定变量的SQL语句，最好的执行计划会根据绑定变量的值而不同，11g以前的方法都无法解决这个问题。在11g中，与adaptive cursor sharing配合，SPM允许你同时接受多个执行计划。执行时，根据不同的变量值，SPM会花费很少的运算从中选择一条最合适的。

  SPM有以下三个主要组件：

  - Plan capture
  
    捕捉并存储与SQL执行计划相关的信息
    
  - Plan selection
  
    使用SQL plan baselines选择合适的执行计划以避免性能退化
    
  - Plan evolution
  
    向SQL plan baselines增加新的执行计划
    
#### SQL Management Base

   SMB是数据字典的一部分，位于SYSAUX表空间，其中存储着statement logs, plan histories, SQL plan baselines, and SQL profiles等内容。

   Plan History是优化器生成的所有执行计划的总称；SQL Plan Baseline是Plan History里那些被标记为“ACCEPTED”的执行计划的总称，称为SQL执行计划基线；SQL Statement Log是一系列的查询签名（signatures），用于在自动捕获执行计划时辨别重复执行的sql；关系如下图所示：

   ![SQL Management Base](http://qiniu.liupzmin.com/SMB.png)

### Plan capture（执行计划的捕获）

 #### Automatic Initial Plan Capture（自动捕获）

   开启自动捕获只需在初始化参数中将**OPTIMIZER_CAPTURE_SQL_PLAN_BASELINES**设置为**TRUE**(默认为FALSE)，开启之后数据库就会为每个重复执行的sql创建SQL Plan Baseline，甄别是否是重复执行的SQL就是利用的上文提到的SQL Statement Log，基线包括供优化器重新生成该执行计划的所有信息，例如SQL text, outline, bind variable values, and compilation environment，这个初始的执行计划会被自动标记为“accepted”，如果之后又有新的执行计划生成，那么该执行计划会被加入到Plan History但是会被标记为“unaccepted”。
     
   Oracle Database 12c Release 2 增加了自动捕获时sql过滤的功能，利用**DBMS_SPM.CONFIGURE**存储过程可以创建automatic capture filter。因此，你可以设置仅仅捕获你想要捕获的SQL，可以从**DBA_SQL_MANAGEMENT_CONFIG**视图中查询当前的设置：
     
   ![SPM CONFIG](http://qiniu.liupzmin.com/SPM-CONFIG.png)
        
   自动捕获状态下执行计划的匹配算法如下：
     

   - 如果SQL plan baseline不存在，那么优化器会为该SQL创建SQL plan baseline和Plan History，并将捕获的plan标记为accepted，加入SQL plan baseline
   - 如果SQL plan baseline存在，那么优化器的行为就依赖于解析时生产的基于成本的执行计划
       - 如果该plan与SQL plan baseline中的plan都不匹配，那么优化器将其标记为unaccepted加入Plan History
       - 如果该plan匹配到SQL plan baseline中的plan，那么执行该plan，不对现有的SQL plan baseline与Plan History做任何改动
         
 #### Manual Plan Capture（手动捕获）

   手动加载已存在的执行计划到SPM是最常用的方式，需要注意的是，默认情况下手动load的plan，会被自动标记为”accepted“，创建新的SQL plan baseline或加入到已有的SQL plan baseline中，oracle提供以下5种方式手动加载Execution Plan：
     
   - From a SQL Tuning Set（DBMS_SPM.LOAD_PLANS_FROM_SQLSET）
   - From the cursor cache （DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE）
   - From the AWR repository (new to Oracle Database 12c Release 2)（DBMS_SPM.LOAD_PLANS_FROM_AWR）
   - Unpacked from a staging table（DBMS_SPM.UNPACK_STGTAB_BASELINE）
   - From existing stored outlines（DBMS_SPM.MIGRATE_STORED_OUTLINE）
     

   ![Loading Plans into a SQL Plan Baseline](http://qiniu.liupzmin.com/manual%20load%20plan%20into%20plan%20baseline.png)
     
  手动load的结果有2种情况，由baseline是否存在而决定：
    

  - 如果SQL Plan Baseline不存在，则数据库做如下事情
      1. Creates a plan history and plan baseline for the statement
      2. Marks the initial plan for the statement as accepted
      3. Adds the plan to the new baseline
  - 如果SQL Plan Baseline存在，则数据库做如下事情
      1. Marks the loaded plan as accepted
      2. Adds the plan to the plan baseline for the statement without verifying the plan's performance

   手动load的plan会被自动标为accepted，因为优化器会假定任何由管理员手动加载的plan都是性能上可接受的；当然，你也可以在load的时候在*DBMS_SPM.LOAD_PLANS_FROM_%* 函数中将**enable**设为**NO**，从而达到禁用的目的。

  具体的load过程，本文暂不做讨论，详情请参阅官方文档 [Managing SQL Plan Baselines](http://docs.oracle.com/database/122/TGSQL/managing-sql-plan-baselines.htm#TGSQL94621)，这里需要说明的一点是从AWR导入执行计划是oracle 12.2才开始加入的，12.2以前要想实现从AWR中导入执行计划，需要先创建SQL Tuning set，并从AWR中将plan load进STS，再使用DBMS_SPM.LOAD_PLANS_FROM_SQLSET将计划加载到SPM中。
    
  下面再谈一下，对于已存在SQL Plan Baseline的SQL，其后续的新的执行计划的捕获情况；

  不论使用哪种方式创建了SQL Plan Baseline，后续新的plan都会被加入到Plan History并被标记为”unaccepted“，此行为不依赖于**OPTIMIZER_CAPTURE_SQL_PLAN_ BASELINES**的设置，新加入的plan不会被使用，直到其经过验证比已accepted的plan性能更好，并演化为accepted加入SQL Plan Baseline。

### Plan Selection（执行计划的选择）

  SPM通过几个标记来实现对执行计划的控制，这些标记可以在视图**DBA_SQL_PLAN_BASELINES**中查到：

- Enabled（控制活动）
    - YES（活动的，但不一定会被使用）
    - NO（禁用的，不活动的，肯定不被使用）
- Accepted（控制使用）
    - YES（只有 “Enabled” 并且 “Accepted” 的计划才会被选择使用）
    - NO（如果是“Enabled” 那么只有被evolve成“Accepted”才有可能被执行）
- Fixed（控制优先级）
    - YES（如果是“Enabled”并且“Accepted”，会优先选择这个计划，这个计划会被视为不需要改变的、固定的）
    - NO（普通的计划，无需优先）
- Reproduced（有效性）
    - YES（优化器可以使用这个计划）
    - NO（计划无效，比如索引被删除）
- ADAPTIVE(12c引入，标记是否是Adaptive Plans)
    - YES（是一个adaptive plan，在evolve的时候会被考虑，evolve时会测试执行，验证后final plan会变为accepted）
    - NO（adaptive plan被evolve后会被标记为NO）

  先来看一下SQL Plan Selection的决策树：

![Decision Tree for SQL Plan Selection](http://qiniu.liupzmin.com/SQL%20Plan%20Selection%20decision%20tree.png)

  当数据库为一条sql执行硬解析的时候，优化器会生成一个best-cost plan，如果初始化参数**OPTIMIZER_USE_SQL_PLAN_BASELINES**被设置成为**TRUE**（默认为TRUE），那么在执行best-cost plan之前会先检查是否存在对应的SQL Plan Baseline，匹配的时候使用SQL语句的signature，signature是一个由SQL文本规范化后的SQL标识符（case insensitivity and with whitespaces removed）（由此可见只要一条sql生成了baseline之后，那么无论大小写改变、空格多少都会被认为是一条sql），这个比较是内存操作，因此开销很小。

  如果baseline不存在，就按生成的计划执行。如果baseline存在，那么要查看history里是否有这个计划，如果没有，就将这个计划插入，并标记为ENABLED,NON-ACCEPTED。

  在baseline中查看是否有FIXED的计划存在，如果存在，执行FIXED的计划，如果存在多个FIXED的计划，根据统计信息重新计算cost，选择cost小的那个。

  如果FIXED的计划不存在，就选择ACCEPTED的计划执行。 如果存在多个ACCEPTED的计划，根据统计信息重新计算cost，选择cost小的那个。

  如果因为某些系统的改变（例如索引删除）导致已accepted的计划无法reproducible，那么优化器将使用新生成的best-cost plan，并将其加入plan history标记为unaccepted

  _* 注意，这里每次重新计算cost的代价不大，因为执行计划是已知的，优化器不必遍历所有的可能，只需根据算法计算出已知计划的cost便可。_

  _* 注意，当sql plan baseline中有Fixed的时候，新生成的执行计划是不会被加入到plan history中的。_

### Plan Evolution（执行计划演化）

  当优化器生成一个新的执行计划后，将其加入到plan history中作为一个unaccepted的plan，它需要被verified之后才可以使用，Verification是一个比较unaccepted和accepted（所有accepted中最小cost的）plan的执行性能的过程，verify的过程是实际执行该plan的过程，通过与accepted的plan比较elapse time, CPU time and buffer gets等性能统计信息来判断新plan的性能，如果新的plan的性能优于原plan或者相差无几，那么该plan会被标记为accepted加入SQL Plan Baseline，否则它仍然是一个unaccepted的plan，但是*LAST_VERIFIED*属性会被更新为当前时刻，12c之后的Automatic plan evolution过程还会考虑自上次被verify之后超过30天的plan，oracle认为如果系统有所改变，也许之前的plan会有更好的性能，当然这个功能可以通过*dbms_spm.alter_sql_plan_baseline*来禁止。

  12c之前没有自动Evolve的机制，从12.1开始automatic plan evolution由SPM Evolve Advisor来完成，11g时代的*DBMS_SPM.EVOLVE_SQL_PLAN_BASELINE*被废弃，下面将分别针对11g和12c来介绍Plan Evolution的过程：

#### 11g DBMS_SPM.EVOLVE_SQL_PLAN_BASELINE

  11g可以使用Oracle Enterprise Manager或者DBMS_SPM.EVOLVE_SQL_PLAN_BASELINE函数来演化SQL Plan，语法如下：

  ```SQL
  DBMS_SPM.EVOLVE_SQL_PLAN_BASELINE (
   	sql_handle   IN VARCHAR2 := NULL,
   	plan_name    IN VARCHAR2 := NULL,
   	time_limit   IN INTEGER  := DBMS_SPM.AUTO_LIMIT,
   	verify       IN VARCHAR2 := 'YES',
   	commit       IN VARCHAR2 := 'YES')
  RETURN CLOB;
  ```

  这里由两个标记控制：
    
    - Verify 
        - YES (验证比较性能)
        - NO (不验证性能)
    - Commit
        - YES (演化)
        - NO (只生成报告，不演化)

  这里可以通过不同的排列组合，达到不同的效果：

  - 自动接收所有性能更好的执行计划，并生成report (Verify->YES, Commit->YES)
  - 自动接收所有新的执行计划，不验证性能，生成report (Verify->NO, Commit->YES)
  - 比较性能，生成report，人工确认是否演化 (Verify->YES, Commit->NO)

#### 12c SPM Evolve Advisor Task & Manually Evolve Task

##### SPM Evolve Advisor Task

  从12.1开始，自动演化任务由SPM Evolve Advisor每天在维护窗口期内自动执行，SPM Evolve Advisor是一个自动任务（SYS_AUTO_SPM_EVOLVE_TASK），它每天做如下操作：

  1. Locates unaccepted plans
  2. Ranks all unaccepted plans
  3. Performs test executions of as many plans as possible during the maintenance window
  4. Selects the lowest-cost plan to compare against each unaccepted plan
  5. Accepts automatically any unaccepted plan that performs sufficiently better, using a cost-based algorithm, than the existing accepted plan

  _* 需要注意的是，没有单独针对Automatic SPM Evolve Advisor task的scheduler client，Automatic SQL Tuning Advisor 和 Automatic SPM Evolve Advisor共用一个client，因此它们两个同时启用或同时禁用。_

  自动任务属性可以通过*DBMS_SPM.SET_EVOLVE_TASK_PARAMETER*来配置，下面列举几个重要的属性：

  ```sql
  COL PARAMETER_NAME FORMAT a25
  COL VALUE FORMAT a42
  SELECT PARAMETER_NAME, PARAMETER_VALUE AS "VALUE"
  FROM   DBA_ADVISOR_PARAMETERS
  WHERE  ( (TASK_NAME = 'SYS_AUTO_SPM_EVOLVE_TASK') AND
         ( (PARAMETER_NAME = 'ACCEPT_PLANS') OR
           (PARAMETER_NAME LIKE '%ALT%') OR
           (PARAMETER_NAME = 'TIME_LIMIT') ) );
  ```
  12.2的默认值如下(各个属性的意义详见官方文档)：

  ```SQL
  PARAMETER_NAME		    VALUE
  ------------------------- ------------------------------------------
  TIME_LIMIT		        3600
  ALTERNATE_PLAN_LIMIT	    10
  ALTERNATE_PLAN_SOURCE	    CURSOR_CACHE+AUTOMATIC_WORKLOAD_REPOSITORY
  ALTERNATE_PLAN_BASELINE   EXISTING
  ACCEPT_PLANS		        TRUE
  ```
  修改属性：

  ```SQL
  BEGIN
  DBMS_SPM.SET_EVOLVE_TASK_PARAMETER(
    task_name => 'SYS_AUTO_SPM_EVOLVE_TASK'
,   parameter => 'TIME_LIMIT'
,   value     => '1200'
);
  DBMS_SPM.SET_EVOLVE_TASK_PARAMETER(
    task_name => 'SYS_AUTO_SPM_EVOLVE_TASK'
,   parameter => 'ACCEPT_PLANS'
,   value     => 'true'
);
  DBMS_SPM.SET_EVOLVE_TASK_PARAMETER(
    task_name => 'SYS_AUTO_SPM_EVOLVE_TASK'
,   parameter => 'ALTERNATE_PLAN_LIMIT'
,   value     => '500'
);
END;
/
  ```

 查看结果：

 ```SQL
 PARAMETER_NAME            VALUE
------------------------- ------------------------------------------
ALTERNATE_PLAN_LIMIT      500
ALTERNATE_PLAN_SOURCE     CURSOR_CACHE+AUTOMATIC_WORKLOAD_REPOSITORY
ALTERNATE_PLAN_BASELINE   EXISTING
ACCEPT_PLANS              true
TIME_LIMIT                1200
 ```

 自动演化的报告可以通过函数*DBMS_SPM.REPORT_AUTO_EVOLVE_TASK*来查询：

 ```
SQL> SET LONG 1000000 PAGESIZE 1000 LONGCHUNKSIZE 100 LINESIZE 100
SQL> SELECT DBMS_SPM.report_auto_evolve_task FROM   dual;

REPORT_AUTO_EVOLVE_TASK
----------------------------------------------------------------------------------------------------
GENERAL INFORMATION SECTION
---------------------------------------------------------------------------------------------

 Task Information:
 ---------------------------------------------
 Task Name	          : SYS_AUTO_SPM_EVOLVE_TASK
 Task Owner	          : SYS
 Description	      : Automatic SPM Evolve Task
 Execution Name       : EXEC_1173
 Execution Type       : SPM EVOLVE
 Scope		          : COMPREHENSIVE
 Status 	          : COMPLETED
 Started	          : 04/20/2017 22:00:05
 Finished	          : 04/20/2017 22:00:09
 Last Updated	      : 04/20/2017 22:00:09
 Global Time Limit    : 3600
 Per-Plan Time Limit  : UNUSED
 Number of Errors     : 0
---------------------------------------------------------------------------------------------

SUMMARY SECTION
---------------------------------------------------------------------------------------------
  Number of plans processed  : 0
  Number of findings	     : 0
  Number of recommendations  : 0
  Number of errors	         : 0
---------------------------------------------------------------------------------------------
 ```

##### Manually Evolve Task

  手动演化的过程大致如下图：

  ![Evolving SQL Plan Baselines](http://qiniu.liupzmin.com/Evolving%20SQL%20Plan%20Baselines.png)

  1. Create an evolve task
  2. Optionally, set evolve task parameters(在12.2.0.1中仅TIME_LIMIT有效)
  3. Execute the evolve task
  4. Implement the recommendations in the task
  5. Report on the task outcome

  个人认为4、5无绝对先后顺序

  具体操作，本文暂不讨论，请浏览[Manually Evolving SQL Plan Baselines in Oracle Database 12c Release 2](https://liupzmin.com/2017/04/24/oracle/perf/Manually-Evolving-SQL-Plan-Baselines-in-Oracle-Database-12c-Release-2/)

### Managing the SQL Management Base   

   利用*DBMS_SPM.CONFIGURE*存储过程可以配置SMB，视图*DBA_SQL_MANAGEMENT_CONFIG*显示了各个配置项的状态。

   参数有如下几个：

   - SPACE_BUDGET_PERCENT（SYSAUX表空间的最大使用率）
   - PLAN_RETENTION_WEEKS（没被使用的plan的最大保留weeks，默认是53周）
   - AUTO_CAPTURE_PARSING_SCHEMA_NAME（自动捕获过滤schema）
   - AUTO_CAPTURE_MODULE
   - AUTO_CAPTURE_MODULE
   - AUTO_CAPTURE_SQL_TEXT（自动捕获指定的sql）
#### 修改SMB磁盘使用限额

  1.查询当前配置

  ```SQL
  
  SELECT PARAMETER_NAME, PARAMETER_VALUE AS "%_LIMIT", 
       ( SELECT sum(bytes/1024/1024) FROM DBA_DATA_FILES 
         WHERE TABLESPACE_NAME = 'SYSAUX' ) AS SYSAUX_SIZE_IN_MB,
       PARAMETER_VALUE/100 * 
       ( SELECT sum(bytes/1024/1024) FROM DBA_DATA_FILES 
         WHERE TABLESPACE_NAME = 'SYSAUX' ) AS "CURRENT_LIMIT_IN_MB"
FROM DBA_SQL_MANAGEMENT_CONFIG
WHERE PARAMETER_NAME = 'SPACE_BUDGET_PERCENT';

PARAMETER_NAME          %_LIMIT SYSAUX_SIZE_IN_MB CURRENT_LIMIT_IN_MB
-------------------- ---------- ----------------- -------------------
SPACE_BUDGET_PERCENT         10          211.4375            21.14375
  ```

  2.修改配额

  ```SQL
  EXECUTE DBMS_SPM.CONFIGURE('space_budget_percent',30);
  ```

  3.查看结果

  ```SQL
  SELECT PARAMETER_NAME, PARAMETER_VALUE AS "%_LIMIT", 
       ( SELECT sum(bytes/1024/1024) FROM DBA_DATA_FILES 
         WHERE TABLESPACE_NAME = 'SYSAUX' ) AS SYSAUX_SIZE_IN_MB,
       PARAMETER_VALUE/100 * 
       ( SELECT sum(bytes/1024/1024) FROM DBA_DATA_FILES 
         WHERE TABLESPACE_NAME = 'SYSAUX' ) AS "CURRENT_LIMIT_IN_MB"
FROM   DBA_SQL_MANAGEMENT_CONFIG
WHERE  PARAMETER_NAME = 'SPACE_BUDGET_PERCENT';

PARAMETER_NAME          %_LIMIT SYSAUX_SIZE_IN_MB CURRENT_LIMIT_IN_MB
-------------------- ---------- ----------------- -------------------
SPACE_BUDGET_PERCENT         30          211.4375            63.43125
  ```

#### 修改保留策略

1.查看当前配置

```SQL
SQL> SELECT PARAMETER_NAME, PARAMETER_VALUE
  2  FROM   DBA_SQL_MANAGEMENT_CONFIG
  3  WHERE  PARAMETER_NAME = 'PLAN_RETENTION_WEEKS';
 
PARAMETER_NAME                 PARAMETER_VALUE
------------------------------ ---------------
PLAN_RETENTION_WEEKS                        53
```

2.修改配置

```SQL
EXECUTE DBMS_SPM.CONFIGURE('plan_retention_weeks',105);
```

3.查看结果

```SQL
SQL> SELECT PARAMETER_NAME, PARAMETER_VALUE
  2  FROM   DBA_SQL_MANAGEMENT_CONFIG
  3  WHERE  PARAMETER_NAME = 'PLAN_RETENTION_WEEKS';
 
PARAMETER_NAME                 PARAMETER_VALUE
------------------------------ ---------------
PLAN_RETENTION_WEEKS                       105
```


### Monitoring SQL plan baselines

  视图DBA_SQL_PLAN_BASELINES展示了当前SQL plan baselines的信息：

  ```SQL
  select SIGNATURE,
       SQL_HANDLE,
       SQL_TEXT,
       PLAN_NAME,
       PARSING_SCHEMA_NAME,
       LAST_EXECUTED,
       ENABLED,
       ACCEPTED,
       REPRODUCED,
       EXECUTIONS
  from dba_sql_plan_baselines;
  ```

  结果如下：

  ```SQL
                SIGNATURE SQL_HANDLE           SQL_TEXT                       PLAN_NAME                      PARSING_SC LAST_EXECUTED                       ENA ACC REP EXECUTIONS
----------------------- -------------------- ------------------------------ ------------------------------ ---------- ----------------------------------- --- --- --- ----------
                                             here id=1

    1962643652257108320 SQL_1b3cb600d175d160 select /*liu*/ *  from test wh SQL_PLAN_1qg5q038rbnb025a3834b SCOTT      13-APR-17 02.23.12.000000 PM        YES YES YES          0
                                             ere id=1

    1962643652257108320 SQL_1b3cb600d175d160 select /*liu*/ *  from test wh SQL_PLAN_1qg5q038rbnb097bbe3d0 HR         13-APR-17 02.35.09.000000 PM        YES YES YES          0
                                             ere id=1

  ```

  > 需要注意的一点是，该视图中LAST_EXECUTED和EXECUTIONS并不是实时更新的。

  函数DBMS_XPLAN.DISPLAY_SQL_PLAN_BASELINE可以查看baseline中的执行计划，语法如下：

  ```SQL
  DBMS_XPLAN.DISPLAY_SQL_PLAN_BASELINE (
   sql_handle      IN VARCHAR2 := NULL,
   plan_name       IN VARCHAR2 := NULL,
   format          IN VARCHAR2 := 'TYPICAL')
 RETURN dbms_xplan_type_table;
  ```

  例如：

  ```SQL
select 
   * 
from table( 
    dbms_xplan.display_sql_plan_baseline( 
        sql_handle=>'SQL_5671036d51fd678f', 
        format=>'basic'));
--或者        
select * from table(dbms_xplan.display_sql_plan_baseline('SQL_17574e83c195631c',null,'BASIC +NOTE'));
  ```

  也可以通过V$SQL视图查看一条SQL是否使用SQL Base Line，如果使用了baseline，那么它的sql_plan_baseline列将会显示plan_name，因此可以连接DBA_SQL_PLAN_BASELINES和V$SQL视图：

  ```SQL
 select s.SQL_ID, s.SQL_TEXT, b.plan_name, b.origin, b.accepted
  from dba_sql_plan_baselines b, v$sql s
 where s.EXACT_MATCHING_SIGNATURE = b.signature
   and s.SQL_PLAN_BASELINE = b.plan_name;
  ```

  查看v$sql中的sql是否有baseline：

  ```SQL
SELECT sql_handle,
  plan_name
FROM dba_sql_plan_baselines
WHERE signature IN
  ( SELECT exact_matching_signature FROM v$sql WHERE sql_id='1s6a8wn4p6rym'
  );
  ```

### Dropping SQL Plan Baselines

1.查询要删除的baseline

```SQL
SQL> SELECT SQL_HANDLE, SQL_TEXT, PLAN_NAME, ORIGIN,
  2         ENABLED, ACCEPTED
  3  FROM   DBA_SQL_PLAN_BASELINES
  4  WHERE  SQL_TEXT LIKE 'SELECT /* repeatable_sql%';
 
SQL_HANDLE           SQL_TEXT             PLAN_NAME                      ORIGIN         ENA ACC
-------------------- -------------------- ------------------------------ -------------- --- ---
SQL_b6b0d1c71cd1807b SELECT /* repeatable SQL_PLAN_bdc6jswfd303v2f1e9c20 AUTO-CAPTURE   YES YES
                     _sql */ count(*) fro
                     m hr.jobs

```

2.删除sql plan baseline

```SQL
DECLARE
  v_dropped_plans number;
BEGIN
  v_dropped_plans := DBMS_SPM.DROP_SQL_PLAN_BASELINE (
     sql_handle => 'SQL_b6b0d1c71cd1807b'
);
  DBMS_OUTPUT.PUT_LINE('dropped ' || v_dropped_plans || ' plans');
END;
/
```

3.确认删除

```SQL
SELECT SQL_HANDLE, SQL_TEXT, PLAN_NAME, ORIGIN,
       ENABLED, ACCEPTED
FROM   DBA_SQL_PLAN_BASELINES
WHERE  SQL_TEXT LIKE 'SELECT /* repeatable_sql%';
 
no rows selected
```

**参考文献：**

1. [Oracle 11g 针对SQL性能的新特性（三）- SQL Plan Management ](https://blogs.oracle.com/Database4CN/entry/oracle_11g_%E9%92%88%E5%AF%B9sql%E6%80%A7%E8%83%BD%E7%9A%84%E6%96%B0%E7%89%B9%E6%80%A7_%E4%B8%89_sql)
2. [Database SQL Tuning Guide](http://docs.oracle.com/database/122/TGSQL/toc.htm)
3. [ White Paper:SQL Plan Management in Oracle Database 11g](http://www.oracle.com/technetwork/database/focus-areas/bi-datawarehousing/twp-sql-plan-management-11gr2-133099.pdf)
4. [White Paper:SQL Plan Management with Oracle Database 12c Release 2](http://www.oracle.com/technetwork/database/bi-datawarehousing/twp-sql-plan-mgmt-12c-1963237.pdf)