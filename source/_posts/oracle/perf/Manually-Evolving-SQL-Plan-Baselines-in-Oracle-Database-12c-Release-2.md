---
date: 2017-04-24 21:29
categories: oracle
tags: SQL Plan Management
title: Manually Evolving SQL Plan Baselines in Oracle Database 12c Release 2
---

在oracle 12.1之前，是使用*EVOLVE_SQL_PLAN_BASELINE*函数来演化SQL plan baselines，从12c开始，演化的过程就被基于task的方法代替了，12c的演化过程主要包括以下几个阶段：

- CREATE_EVOLVE_TASK
- EXECUTE_EVOLVE_TASK
- REPORT_EVOLVE_TASK
- IMPLEMENT_EVOLVE_TASK

下面将演示以下12c执行计划基线演化的过程：

先来创建一个测试表：

```SQL
CREATE TABLE spm_test_tab (
  id           NUMBER,
  description  VARCHAR2(50)
);

INSERT /*+ APPEND */ INTO spm_test_tab
SELECT level,
       'Description for ' || level
FROM   dual
CONNECT BY level <= 10000;
COMMIT;
```

查询一条记录，并查看其执行计划

```SQL
SELECT description
FROM   spm_test_tab
WHERE  id = 99;

SELECT * FROM TABLE (SELECT DBMS_XPLAN.DISPLAY_CURSOR(null,null,'advanced') from dual);

PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
SQL_ID	gat6z1bc6nc2d, child number 0
-------------------------------------
SELECT description FROM   spm_test_tab WHERE  id = 99

Plan hash value: 1107868462

----------------------------------------------------------------------------------
| Id  | Operation	      | Name	     | Rows  | Bytes | Cost (%CPU)| Time	 |
----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |		         |	     |	     |    13 (100)| 	     |
|*  1 |  TABLE ACCESS FULL| SPM_TEST_TAB |     1 |    40 |    13   (0)| 00:00:01 |
----------------------------------------------------------------------------------

Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------

   1 - SEL$1 / SPM_TEST_TAB@SEL$1

Outline Data
-------------

  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('12.2.0.1')
      DB_VERSION('12.2.0.1')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$1")
      FULL(@"SEL$1" "SPM_TEST_TAB"@"SEL$1")
      END_OUTLINE_DATA
  */

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter("ID"=99)

Column Projection Information (identified by operation id):
-----------------------------------------------------------

   1 - "DESCRIPTION"[VARCHAR2,50]

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
```

查询sql_id

```SQL

SELECT sql_id
FROM   v$sql
WHERE  plan_hash_value = 1107868462;

SQL_ID
-------------
gat6z1bc6nc2d

```
将该sql的执行计划从cursor cache中加载到SQL Plan Baseline

```SQL
SET SERVEROUTPUT ON
DECLARE
  l_plans_loaded  PLS_INTEGER;
BEGIN
  l_plans_loaded := DBMS_SPM.load_plans_from_cursor_cache(
    sql_id => 'gat6z1bc6nc2d');
    
  DBMS_OUTPUT.put_line('Plans Loaded: ' || l_plans_loaded);
END;
/
Plans Loaded: 1

PL/SQL procedure successfully completed.
```

在DBA_SQL_PLAN_BASELINES中查询baseline的信息

```SQL
COLUMN sql_handle FORMAT A20
COLUMN plan_name FORMAT A30

SELECT sql_handle, plan_name, enabled, accepted 
FROM   dba_sql_plan_baselines;

SQL_HANDLE	     			PLAN_NAME			    ENA ACC
-------------------- ------------------------------ --- ---
SQL_7b76323ad90440b9 SQL_PLAN_7qxjk7bch8h5tb65c37c8 YES YES
```
创建一条索引，再次执行该sql，查看执行计划结果

```SQL
CREATE INDEX spm_test_tab_idx ON spm_test_tab(id);
begin
 dbms_stats.gather_table_stats(ownname => 'SYS',
 tabname =>'SPM_TEST_TAB',
 estimate_percent =>DBMS_STATS.AUTO_SAMPLE_SIZE,
 method_opt => 'FOR ALL COLUMNS SIZE AUTO',
 granularity=>'ALL',
 cascade => TRUE,
 no_invalidate => false);
end;
/

SELECT description
FROM   spm_test_tab
WHERE  id = 99;

Execution Plan
----------------------------------------------------------
Plan hash value: 1107868462

----------------------------------------------------------------------------------
| Id  | Operation	  		| Name	 	   | Rows  | Bytes | Cost (%CPU)| Time	  |
----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  	|		 	   |     1 |    25 |    13   (0)| 00:00:01|
|*  1 |  TABLE ACCESS FULL	| SPM_TEST_TAB |     1 |    25 |    13   (0)| 00:00:01|
----------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter("ID"=99)

Note
-----
   - SQL plan baseline "SQL_PLAN_7qxjk7bch8h5tb65c37c8" used for this statement


Statistics
----------------------------------------------------------
0  recursive calls
3  db block gets
52  consistent gets
0  physical reads
0  redo size
365  bytes sent via SQL*Net to client
484  bytes received via SQL*Net from client
2  SQL*Net roundtrips to/from client
0  sorts (memory)
0  sorts (disk)
1  rows processed
```

可见，在有索引的情况下执行计划依然走了全表扫描，说明SPM已发挥作用，现在查看一下DBA_SQL_PLAN_BASELINES

```SQL
SELECT sql_handle, plan_name, enabled, accepted 
FROM   dba_sql_plan_baselines
WHERE  sql_handle = 'SQL_7b76323ad90440b9';

SQL_HANDLE	     	 PLAN_NAME			    		ENA ACC
-------------------- ------------------------------ --- ---
SQL_7b76323ad90440b9 SQL_PLAN_7qxjk7bch8h5t3652c362 YES NO
SQL_7b76323ad90440b9 SQL_PLAN_7qxjk7bch8h5tb65c37c8 YES YES
```

现在执行计划基线中已经有了一条未被accepted的计划，我们可以查看一下该计划的内容：

```SQL
select 
   * 
from table( 
    dbms_xplan.display_sql_plan_baseline( 
        sql_handle=>'SQL_7b76323ad90440b9', 
        plan_name=>'SQL_PLAN_7qxjk7bch8h5t3652c362',
        format=>'basic +NOTE'));
        
 --------------------------------------------------------------------------------
SQL handle: SQL_7b76323ad90440b9
SQL text: SELECT description FROM   spm_test_tab WHERE	id = 99
--------------------------------------------------------------------------------

--------------------------------------------------------------------------------
Plan name: SQL_PLAN_7qxjk7bch8h5t3652c362	  Plan id: 911393634
Enabled: YES	 Fixed: NO	Accepted: NO	  Origin: AUTO-CAPTURE
Plan rows: From dictionary
--------------------------------------------------------------------------------

Plan hash value: 2338891031

----------------------------------------------------------------
| Id  | Operation			    			| Name	       	   |
----------------------------------------------------------------
|   0 | SELECT STATEMENT		    		|		       	   |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| SPM_TEST_TAB     |
|   2 |   INDEX RANGE SCAN		    		| SPM_TEST_TAB_IDX |
----------------------------------------------------------------
```

可见新的plan是走索引的，现在可以等待维护窗口期间由自动维护任务自动evolve，或者手动进行evolve，现在演示手动evolve，创建一个evolve task

```SQL
SET SERVEROUTPUT ON
DECLARE
  l_return VARCHAR2(32767);
BEGIN
  l_return := DBMS_SPM.create_evolve_task(sql_handle => 'SQL_7b76323ad90440b9');
  DBMS_OUTPUT.put_line('Task Name: ' || l_return);
END;
/
Task Name: TASK_1169

PL/SQL procedure successfully completed.
```

执行此任务

```SQL
SET SERVEROUTPUT ON
DECLARE
  l_return VARCHAR2(32767);
BEGIN
  l_return := DBMS_SPM.execute_evolve_task(task_name => 'TASK_1169');
  DBMS_OUTPUT.put_line('Execution Name: ' || l_return);
END;
/
Execution Name: EXEC_1281

PL/SQL procedure successfully completed.
```

查看任务报告

```SQL
SET LONG 1000000 PAGESIZE 1000 LONGCHUNKSIZE 100 LINESIZE 100
SELECT DBMS_SPM.report_evolve_task(task_name => 'TASK_1169', execution_name => 'EXEC_1281') AS output
FROM   dual;

OUTPUT
----------------------------------------------------------------------------------------------------
GENERAL INFORMATION SECTION
---------------------------------------------------------------------------------------------

 Task Information:
 ---------------------------------------------
 Task Name	      	  : TASK_1169
 Task Owner	          : SYS
 Execution Name       : EXEC_1281
 Execution Type       : SPM EVOLVE
 Scope		          : COMPREHENSIVE
 Status 	          : COMPLETED
 Started	          : 04/24/2017 15:04:17
 Finished	          : 04/24/2017 15:04:25
 Last Updated	      : 04/24/2017 15:04:25
 Global Time Limit    : 2147483646
 Per-Plan Time Limit  : UNUSED
 Number of Errors     : 0
---------------------------------------------------------------------------------------------

SUMMARY SECTION
---------------------------------------------------------------------------------------------
  Number of plans processed  : 1
  Number of findings	     : 1
  Number of recommendations  : 1
  Number of errors	         : 0
---------------------------------------------------------------------------------------------

DETAILS SECTION
---------------------------------------------------------------------------------------------
 Object ID	        : 2
 Test Plan Name     : SQL_PLAN_7qxjk7bch8h5t3652c362
 Base Plan Name     : SQL_PLAN_7qxjk7bch8h5tb65c37c8
 SQL Handle	        : SQL_7b76323ad90440b9
 Parsing Schema     : SYS
 Test Plan Creator  : SYS
 SQL Text	        : SELECT description FROM spm_test_tab WHERE id = 99

Execution Statistics:
-----------------------------
		    Base Plan			          Test Plan
		    ----------------------------  ----------------------------
 Elapsed Time (s):  .000037			      .000005
 CPU Time (s):	    .000044			      0
 Buffer Gets:	    5				      0
 Optimizer Cost:    13				      2
 Disk Reads:	    0				      0
 Direct Writes:     0				      0
 Rows Processed:    0				      0
 Executions:	    10				      10


FINDINGS SECTION
---------------------------------------------------------------------------------------------

Findings (1):
-----------------------------
 1. The plan was verified in 0.03200 seconds. It passed the benefit criterion
    because its verified performance was 18.01480 times better than that of the
    baseline plan.

Recommendation:
-----------------------------
 Consider accepting the plan. Execute
 dbms_spm.accept_sql_plan_baseline(task_name => 'TASK_1169', object_id => 2,
 task_owner => 'SYS');


EXPLAIN PLANS SECTION
---------------------------------------------------------------------------------------------

Baseline Plan
-----------------------------
 Plan Id	  : 1
 Plan Hash Value  : 3059496904

-----------------------------------------------------------------------------
| Id  | Operation	        | Name	       | Rows | Bytes | Cost | Time     |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |		       |	1 |    25 |   13 | 00:00:01 |
| * 1 |   TABLE ACCESS FULL | SPM_TEST_TAB |	1 |    25 |   13 | 00:00:01 |
-----------------------------------------------------------------------------

Predicate Information (identified by operation id):
------------------------------------------
* 1 - filter("ID"=99)


Test Plan
-----------------------------
 Plan Id	  : 2
 Plan Hash Value  : 911393634

---------------------------------------------------------------------------------------------------
| Id  | Operation			                 | Name		        | Rows | Bytes | Cost | Time	 |
---------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT		              | 		         |   1 |    25 |    2 | 00:00:01 |
|   1 |   TABLE ACCESS BY INDEX ROWID BATCHED | SPM_TEST_TAB	 |   1 |    25 |    2 | 00:00:01 |
| * 2 |    INDEX RANGE SCAN		              | SPM_TEST_TAB_IDX |   1 |	   |    1 | 00:00:01 |
---------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
------------------------------------------
* 2 - access("ID"=99)

---------------------------------------------------------------------------------------------
```

报告中建议执行dbms_spm.accept_sql_plan_baseline接受新的plan，但我们应该使用IMPLEMENT_EVOLVE_TASK来接受新的plan

```SQL
SET SERVEROUTPUT ON
DECLARE
  l_return NUMBER;
BEGIN
  l_return := DBMS_SPM.implement_evolve_task(task_name => 'TASK_1169');
  DBMS_OUTPUT.put_line('Plans Accepted: ' || l_return);
END;
/
Plans Accepted: 1

PL/SQL procedure successfully completed.
```

查看DBA_SQL_PLAN_BASELINES，新的plan应该已经是accepted了

```SQL
SELECT sql_handle, plan_name, enabled, accepted 
FROM   dba_sql_plan_baselines
WHERE  sql_handle = 'SQL_7b76323ad90440b9';

SQL_HANDLE	     	 PLAN_NAME			    		ENA ACC
-------------------- ------------------------------ --- ---
SQL_7b76323ad90440b9 SQL_PLAN_7qxjk7bch8h5t3652c362 YES YES
SQL_7b76323ad90440b9 SQL_PLAN_7qxjk7bch8h5tb65c37c8 YES YES
```

再次执行该sql，查看执行计划

```SQL
SET AUTOTRACE TRACE LINESIZE 130

SELECT description
FROM   spm_test_tab
WHERE  id = 99;

Execution Plan
----------------------------------------------------------
Plan hash value: 2338891031

--------------------------------------------------------------------------------------------------------
| Id  | Operation			               | Name	          | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT		            |		           |     1 |    25 |     2	 (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| SPM_TEST_TAB     |     1 |    25 |     2	 (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN		            | SPM_TEST_TAB_IDX |     1 |       |     1	 (0)| 00:00:01 |
--------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("ID"=99)

Note
-----
   - SQL plan baseline "SQL_PLAN_7qxjk7bch8h5t3652c362" used for this statement


Statistics
----------------------------------------------------------
0  recursive calls
0  db block gets
4  consistent gets
0  physical reads
0  redo size
372  bytes sent via SQL*Net to client
484  bytes received via SQL*Net from client
2  SQL*Net roundtrips to/from client
0  sorts (memory)
0  sorts (disk)
1  rows processed
```

由此可见，新的执行计划基线已被使用。

参考资料：[Adaptive SQL Plan Management (SPM) in Oracle Database 12c Release 1 (12.1)](https://oracle-base.com/articles/12c/adaptive-sql-plan-management-12cr1)