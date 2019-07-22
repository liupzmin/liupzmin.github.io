---
date: 2016-08-04 16:24
tags: oracle
categories: oracle
title: Dropping Table Columns With Unused
---

**日常管理中经常会碰到需要drop表中字段的情况，如果表中数据很少时还可以直接drop，但是如果是一张大表，那么drop一个列是很耗时且耗资源的事，那么我们可以先逻辑上删除此字段，再在空闲的时候彻底drop掉以释放空间。**
>注：本文摘自官方文档

##1. Marking Columns Unused

If you are concerned about the length of time it could take to drop column data from all of the rows in a large table, you can use the ALTER TABLE...SET UNUSED statement. This statement marks one or more columns as unused, but does not actually remove the target column data or restore the disk space occupied by these columns. However, a column that is marked as unused is not displayed in queries or data dictionary views, and its name is removed so that a new column can reuse that name. All constraints, indexes, and statistics defined on the column are also removed.

To mark the hiredate and mgr columns as unused, execute the following statement:

```sql
ALTER TABLE hr.admin_emp SET UNUSED (hiredate, mgr);
```

You can later remove columns that are marked as unused by issuing an ALTER TABLE...DROP UNUSED COLUMNS statement. Unused columns are also removed from the target table whenever an explicit drop of any particular column or columns of the table is issued.

The data dictionary views USER_UNUSED_COL_TABS, ALL_UNUSED_COL_TABS, or DBA_UNUSED_COL_TABS can be used to list all tables containing unused columns. The COUNT field shows the number of unused columns in the table.

```sql
SELECT * FROM DBA_UNUSED_COL_TABS;

OWNER                       TABLE_NAME                  COUNT
--------------------------- --------------------------- -----
HR                          ADMIN_EMP                       2
```

*For external tables, the SET UNUSED statement is transparently converted into an ALTER TABLE DROP COLUMN statement. Because external tables consist of metadata only in the database, the DROP COLUMN statement performs equivalently to the SET UNUSED statement.*
##2. Removing Unused Columns

The ALTER TABLE...DROP UNUSED COLUMNS statement is the only action allowed on unused columns. It physically removes unused columns from the table and reclaims disk space.

In the ALTER TABLE statement that follows, the optional clause CHECKPOINT is specified. This clause causes a checkpoint to be applied after processing the specified number of rows, in this case 250. Checkpointing cuts down on the amount of undo logs accumulated during the drop column operation to avoid a potential exhaustion of undo space.

```sql
ALTER TABLE hr.admin_emp DROP UNUSED COLUMNS CHECKPOINT 250;
```