---
date: 2017-04-26 16:19
categories: oracle
tags: SQL Plan Management
title: SPM Using SPM to Correct Regressed SQL Statements
---

在*LOAD_PLANS_FROM_CURSOR_CACHE*函数中有一种语法组合如下：

```SQL
DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE (
   sql_id            IN  VARCHAR2,
   plan_hash_value   IN  NUMBER   := NULL,
   sql_handle        IN  VARCHAR2,  --注意这里
   fixed             IN  VARCHAR2 := 'NO',
   enabled           IN  VARCHAR2 := 'YES')
 RETURN PLS_INTEGER;
```

Oracle对其中*sql_handle*的解释为：

*SQL handle to use in identifying the SQL plan baseline into which the plans are loaded. The sql_handle must denote an existing SQL plan baseline. The use of handle is crucial when the user tunes a SQL statement by adding hints to its text and then wants to load the resulting plan(s) into the SQL plan baseline of the original SQL statement.*

意思是说，使用sql_handle可以将cursor cache中的执行计划load进一个已存在的SQL Plan Baseline，这就意味着你可以不需要更改应用的代码，就可以让SQL跑出你期望的执行计划，前提是你有一个已经tuning好的plan在cacahe里，你就可以将这个plan load进原SQL的Baseline，下面将针对此调优方法展开详细论述：

> 我要进行的实验是要一个原本走索引的sql，在不改变原sql的基础上，让它走全表扫描

1. 创建测试环境

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
	--创建索引
	CREATE INDEX spm_test_tab_idx ON spm_test_tab(id);
	--收集统计信息
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
	```

2. 执行例句，查看其执行计划

	```SQL
	
	SELECT description FROM  spm_test_tab WHERE  id = 1113;
	
	SELECT * FROM TABLE (SELECT DBMS_XPLAN.DISPLAY_CURSOR(null,null,'advanced') from dual);
	
    SQL_ID  gsttbra9z0ddw, child number 0
    -------------------------------------
    SELECT description FROM  spm_test_tab WHERE  id = 1113

    Plan hash value: 3121206333

    ------------------------------------------------------------------------------------------------
    | Id  | Operation                   | Name             | Rows  | Bytes | Cost (%CPU)| Time     |
    ------------------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT            |                  |       |       |     2 (100)|          |
    |   1 |  TABLE ACCESS BY INDEX ROWID| SPM_TEST_TAB     |     1 |    25 |     2   (0)| 00:00:01 |
    |*  2 |   INDEX RANGE SCAN          | SPM_TEST_TAB_IDX |     1 |       |     1   (0)| 00:00:01 |
    ------------------------------------------------------------------------------------------------

    Query Block Name / Object Alias (identified by operation id):
    -------------------------------------------------------------

      1 - SEL$1 / SPM_TEST_TAB@SEL$1
      2 - SEL$1 / SPM_TEST_TAB@SEL$1

    Outline Data
    -------------

    /*+
        BEGIN_OUTLINE_DATA
        IGNORE_OPTIM_EMBEDDED_HINTS
        OPTIMIZER_FEATURES_ENABLE('11.2.0.4')
        DB_VERSION('11.2.0.4')
        ALL_ROWS
        OUTLINE_LEAF(@"SEL$1")
        INDEX_RS_ASC(@"SEL$1" "SPM_TEST_TAB"@"SEL$1" ("SPM_TEST_TAB"."ID"))
        END_OUTLINE_DATA
        */

        Predicate Information (identified by operation id):
        ---------------------------------------------------

        2 - access("ID"=1113)

        Column Projection Information (identified by operation id):
        -----------------------------------------------------------

          1 - "DESCRIPTION"[VARCHAR2,50]
          2 - "SPM_TEST_TAB".ROWID[ROWID,10]
	```

3. 从v$sql中查找语句的sql_id，并使用DBMS_SPM.LOAD_PLAN_FROM_CURSOR_CACHE创建SQL Plan Baseline

	```SQL
	SYS@DB41 2017/04/26 14:57:55> SELECT SQL_ID,SQL_FULLTEXT FROM V$SQL WHERE SQL_FULLTEXT LIKE  '%1113%';

	SQL_ID	      SQL_FULLTEXT
	------------- --------------------------------------------------------------------------------
	gsttbra9z0ddw SELECT description FROM  spm_test_tab WHERE  id = 1113
	
	declare
    		v_sql_plan_id  pls_integer;
	begin
    		v_sql_plan_id := dbms_spm.load_plans_from_cursor_cache(sql_id => 'gsttbra9z0ddw');
	end;
	/
	
	SELECT SQL_HANDLE,PLAN_NAME,ENABLED,ACCEPTED
	FROM dba_sql_plan_baselines
	WHERE signature IN
    	( SELECT exact_matching_signature FROM v$sql WHERE sql_id='gsttbra9z0ddw'
    	);
   ```
  ```

  	SQL_HANDLE		       		   PLAN_NAME		      		  ENA ACC
	------------------------------ ------------------------------ --- ---
	SQL_c1c9aa52fd90f3ae	       SQL_PLAN_c3kdaabyt1wxfed3324c0 YES YES
  ```

4. 使用ALTER_SQL_PLAN_BASELINE禁用原执行计划

	```SQL
	declare
    		v_sql_plan_id  pls_integer;
	begin
    		v_sql_plan_id := dbms_spm.ALTER_SQL_PLAN_BASELINE(sql_handle => 'SQL_c1c9aa52fd90f3ae',
    														  plan_name => 'SQL_PLAN_c3kdaabyt1wxfed3324c0',
    														  attribute_name => 'enabled',
    														  attribute_value => 'NO');
	end;
	/
	
	SQL_HANDLE		       		   PLAN_NAME		      		  ENA ACC
	------------------------------ ------------------------------ --- ---
	SQL_c1c9aa52fd90f3ae	       SQL_PLAN_c3kdaabyt1wxfed3324c0  NO YES
	```
	
5. 加入hint使sql走全表扫描

	```SQL
	SELECT /*+ FULL(SPM_TEST_TAB) */description FROM  spm_test_tab WHERE  id = 1113;
	
	SELECT * FROM TABLE (SELECT DBMS_XPLAN.DISPLAY_CURSOR(null,null,'advanced') from dual);
	
	SQL_ID	fr5dd23pfbjfz, child number 0
	-------------------------------------
	SELECT /*+ FULL(SPM_TEST_TAB) */description FROM  spm_test_tab WHERE
	id = 1113

	Plan hash value: 1107868462

	----------------------------------------------------------------------------------
	| Id  | Operation	  		| Name	 	   | Rows  | Bytes | Cost (%CPU)| Time	  |
	----------------------------------------------------------------------------------
	|   0 | SELECT STATEMENT  	|		 	   |	   |	   |    13 (100)| 	      |
	|*  1 |  TABLE ACCESS FULL	| SPM_TEST_TAB |     1 |    25 |    13   (0)| 00:00:01|
	----------------------------------------------------------------------------------

	Query Block Name / Object Alias (identified by operation id):
	-------------------------------------------------------------

   	1 - SEL$1 / SPM_TEST_TAB@SEL$1

	Outline Data
	-------------
	```

 	 /*+
      	BEGIN_OUTLINE_DATA
      	IGNORE_OPTIM_EMBEDDED_HINTS
      	OPTIMIZER_FEATURES_ENABLE('11.2.0.4')
      	DB_VERSION('11.2.0.4')
      	ALL_ROWS
      	OUTLINE_LEAF(@"SEL$1")
      	FULL(@"SEL$1" "SPM_TEST_TAB"@"SEL$1")
      	END_OUTLINE_DATA
  	*/

	Predicate Information (identified by operation id):
	---------------------------------------------------

   		1 - filter("ID"=1113)

	Column Projection Information (identified by operation id):
	-----------------------------------------------------------

   		1 - "DESCRIPTION"[VARCHAR2,50]
	```
	
6. 在V$SQL中找到加入hint的语句的sql_id和plan_hash_value

	```SQL
	SELECT SQL_ID,PLAN_HASH_VALUE,SQL_FULLTEXT FROM V$SQL WHERE SQL_FULLTEXT LIKE  '%1113%';
	
	SQL_ID	      PLAN_HASH_VALUE SQL_FULLTEXT
	------------- --------------- --------------------------------------------------------------------------------
	fr5dd23pfbjfz	   1107868462 SELECT /*+ FULL(SPM_TEST_TAB) */description FROM	spm_test_tab WHERE  id = 1113
	08vxbgd0qrm8s	    903671040 SELECT SQL_ID,SQL_FULLTEXT FROM V$SQL WHERE SQL_FULLTEXT LIKE  '%1113%'
	```
	
7. 利用SQL_ID和PLAN_HASH_VALUE创建一个新的accepted的plan，并通过SQL_HANDLE将修改过的plan与原SQL联系起来

	```SQL
	declare
    		v_sql_plan_id  pls_integer;
	begin
    		v_sql_plan_id := dbms_spm.load_plans_from_cursor_cache(sql_id => 'fr5dd23pfbjfz',
    														  plan_hash_value => 1107868462,
    														  sql_handle => 'SQL_c1c9aa52fd90f3ae');
	end;
	/
	```
	
8. 查询DBA_SQL_PLAN_BASELINES会看到2个plan

	```SQL
	SQL_HANDLE		       		   PLAN_NAME		              ENA ACC
	------------------------------ ------------------------------ --- ---
	SQL_c1c9aa52fd90f3ae	       SQL_PLAN_c3kdaabyt1wxfb65c37c8 YES YES
	SQL_c1c9aa52fd90f3ae	       SQL_PLAN_c3kdaabyt1wxfed3324c0 NO  YES
	```
	
9. 现在执行原sql查看其执行计划

	```SQL
	SYS@DB41 2017/04/26 15:50:44> SET AUTOTRACE TRACE
	SYS@DB41 2017/04/26 15:50:57> SELECT description FROM  spm_test_tab WHERE  id = 1113;
	```


	Execution Plan
	----------------------------------------------------------
	Plan hash value: 1107868462
	
	----------------------------------------------------------------------------------
	| Id  | Operation	  	  | Name	 	 | Rows  | Bytes | Cost (%CPU)| Time	 |
	----------------------------------------------------------------------------------
	|   0 | SELECT STATEMENT  |		 	 	 |     1 |    25 |    13   (0)| 00:00:01 |
	|*  1 |  TABLE ACCESS FULL| SPM_TEST_TAB |     1 |    25 |    13   (0)| 00:00:01 |
	----------------------------------------------------------------------------------
	
	Predicate Information (identified by operation id):
	---------------------------------------------------

   		1 - filter("ID"=1113)

	Note
	-----
   		- SQL plan baseline "SQL_PLAN_c3kdaabyt1wxfb65c37c8" used for this statement


	Statistics
	----------------------------------------------------------
	7  recursive calls
	0  db block gets
	45  consistent gets
	0  physical reads
	0  redo size
	367  bytes sent via SQL*Net to client
	476  bytes received via SQL*Net from client
	2  SQL*Net roundtrips to/from client
	0  sorts (memory)
	0  sorts (disk)
	1  rows processed
	```
	
	> 可见，执行计划已改变
	
	> 注： 测试过程中发现，使用SQL Plan Baseline的sql无法再用SELECT * FROM TABLE (SELECT DBMS_XPLAN.DISPLAY_CURSOR(null,null,'advanced') from dual)获取其上次的执行计划。