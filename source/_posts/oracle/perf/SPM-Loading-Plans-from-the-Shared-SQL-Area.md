---
date: 2017-04-26 11:29
categories: oracle
tags: SQL Plan Management
title: [SPM] Loading Plans from the Shared SQL Area
---

> 从Cursor Cache中加载执行计划到SPM中，是最常用的一种方式，下面将介绍整个加载的步骤。

1. 确认sql在Cursor Cache中

	```SQL
    select s.CHILD_NUMBER,
         s.PLAN_HASH_VALUE,
         s.HASH_VALUE,
         s.PARSING_SCHEMA_NAME,
         s.LAST_ACTIVE_TIME,
         s.OPTIMIZER_COST,
         s.sql_plan_baseline
   from v$sql s
   where sql_id = '1s6a8wn4p6rym'
   order by s.LAST_ACTIVE_TIME desc;
	```
	
	_*注意，不是所有在v$sql中看到的plan都被加载到SPM中，其中有个规律是，如果plan是相同的，即便是不同的schema，在SPM中都只有一条plan，而且这不同的schema执行同样的sql时，会使用同一个执行计划基线（前提是基线的plan能够在各自的schema中重新生成）_
	
2. 使用DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE加载执行计划
	
	```SQL
	declare
    		v_sql_plan_id  pls_integer;
	begin
    		v_sql_plan_id := dbms_spm.load_plans_from_cursor_cache(sql_id => '59t5p5xv2bjjy');
	end;
	/
	```
	
	该匿名块会将CURSOR_CACHE中所有的plan都加入到SQL Plan Baseline中，并默认为accepted，如需改变默认效果，请参阅[Oracle Database PL/SQL Packages and Types Reference](http://docs.oracle.com/database/122/ARPLS/DBMS_SPM.htm#GUID-4EFE728B-8A2A-4DF4-ABE6-E7B133CDB5DA)

3. 查看DBA_SQL_PLAN_BASELINES确认load结果
	
	```SQL
	SELECT *
	FROM dba_sql_plan_baselines
	WHERE signature IN
    	( SELECT exact_matching_signature FROM v$sql WHERE sql_id='59t5p5xv2bjjy'
    	);
	```
4. 删除掉不想要的Baseline

	如果能确定不想要的plan（当然，也可以在load的时候进行选择），那么可以通过*SPM.DROP_SQL_PLAN_BASELINE*删除
	
	先找到sql_handle,再查找该sql_handle下所有的plan，例如：
	
	```SQL
	SELECT sql_handle
	FROM dba_sql_plan_baselines
	WHERE signature IN
    	( SELECT exact_matching_signature FROM v$sql WHERE sql_id='59t5p5xv2bjjy'
    	);
   ```
  ```

  	select SIGNATURE,
       SQL_HANDLE,
       SQL_TEXT,
       PLAN_NAME,
       PARSING_SCHEMA_NAME,
       LAST_EXECUTED,
       ENABLED,
       ACCEPTED,
       REPRODUCED,
       EXECUTIONS,
       optimizer_cost
    from dba_sql_plan_baselines;
	where sql_handle ='SQL_fca270c30418ee86';
  ```

	确定不需要的plan有很多方式，如果你正在tuning该条sql，那么你对不好的执行计划的cost肯定印象深刻，那么就可以根据optimizer_cost确定PLAN_NAME，也可以通过*dbms_xplan.display_sql_plan_baseline*来浏览执行计划确定，接下来就可以删除该plan了：
	
	```SQL
	DECLARE
  		v_dropped_plans number;
	BEGIN
  		v_dropped_plans := DBMS_SPM.DROP_SQL_PLAN_BASELINE (
     	sql_handle => 'SQL_971c23013e9cf52a',
     	plan_name => 'SQL_PLAN_9f71304z9tx9a42ef5257'
		);
  		DBMS_OUTPUT.PUT_LINE('dropped ' || v_dropped_plans || ' plans');
	END;
	```
	
5. 演化（evolve）新的plan

	11g默认为已存在Baseline的SQL自动捕捉新的plan，但新的plan为unaccepted的状态，要使用新的plan，需要演化（evolve）为accepted，下面介绍11g的演化方法，12c请参阅[MANUALLY EVOLVING SQL PLAN BASELINES IN ORACLE DATABASE 12C RELEASE 2](https://liupzmin.com/2017/04/24/manually-evolving-sql-plan-baselines-in-oracle-database-12c-release-2/)
	
	```SQL
	DECLARE  
	 		l_plans_altered  clob;  
	BEGIN  
		l_plans_altered := dbms_spm.evolve_sql_plan_baseline(  
		sql_handle      => 'SQL_fca270c30418ee86',  
		plan_name       => 'SQL_PLAN_gt8mhsc21jvn62e14fda0',  
		verify           =>'YES',  
		commit      =>'YES');  
		DBMS_OUTPUT.put_line('Plans Altered: ' || l_plans_altered);  
	END;  
	/  
	```

	该匿名块会评估测试执行plan_name为*SQL_PLAN_gt8mhsc21jvn62e14fda0*的plan，并生成报告，如果它的性能优于现存的accepted的plan，那么它会被自动接受为accepted，否则便不会被接受。