---
layout: post
title: SQL_PROFILE的绑定及测试(三)
date: 2016-02-29
categories: OracleScripts
tags: [Oracle,Scripts]
description: 优化基础。
---

上一篇博文展示了coe_xfr_sql_profile.sql脚本的使用方法，但是这个脚本是否是真的无懈可击，这篇开始将进行一些其他的测试~


## 测试SQLPROFILE自动修复性

### 1.将索引至为unusable

```sql

SQL>alter index hr.emp_idx_testsqlp unusable;
 
Index altered.
 
SQL>set autot trace
SQL>select * from hr.employees where employee_id=196;
select * from hr.employees where employee_id=196
*
ERROR at line 1:
ORA-01502: index 'HR.EMP_IDX_TESTSQLP' or partition of such index is in unusable state

```

如上，将索引至为不可用，SQLPROFILE的SQL会报错

```sql

SQL> select employee_id from hr.employees where employee_id=196;
 
 
Execution Plan
----------------------------------------------------------
Plan hash value: 1386472110
 
-----------------------------------------------------------------------------------
| Id  | Operation         | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |               |     1 |     4 |     0   (0)| 00:00:01 |
|*  1 |  INDEX UNIQUE SCAN| EMP_EMP_ID_PK |     1 |     4 |     0   (0)| 00:00:01 |
-----------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - access("EMPLOYEE_ID"=196)
 
 
Statistics
----------------------------------------------------------
          2  recursive calls
          0  db block gets
          4  consistent gets
          0  physical reads
          0  redo size
        530  bytes sent via SQL*Net to client
        524  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed
		  
```		  
		  
如上，其他的sql不会受到这个索引的影响。

```sql

SQL>alter index hr.emp_idx_testsqlp rebuild;
 
Index altered.
 
SQL>set lines 300
SQL>select * from hr.employees where employee_id=196;
 
 
Execution Plan
----------------------------------------------------------
Plan hash value: 1613276963
 
------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name             | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |                  |     1 |    72 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEES        |     1 |    72 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | EMP_IDX_TESTSQLP |     1 |       |     1   (0)| 00:00:01 |
------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("EMPLOYEE_ID"=196)
 
Note
-----
   - SQL profile "coe_bvxjvrwqcq3pn_1613276963" used for this statement
 
 
Statistics
----------------------------------------------------------
         10  recursive calls
          0  db block gets
         12  consistent gets
          0  physical reads
          0  redo size
       1308  bytes sent via SQL*Net to client
        524  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed

```
		  
将索引重建之后，SQLPROFILE正常。
**注意：unusable状态的索引，只可以drop后build或者rebuild。**



### 2.将索引至为INVISIBLE（11G新特性）

```sql

SQL> alter index hr.emp_idx_testsqlp invisible;
 
Index altered.
 
SQL> set autot trace
SQL> select * from hr.employees where employee_id=196;
 
 
Execution Plan
----------------------------------------------------------
Plan hash value: 1445457117
 
-------------------------------------------------------------------------------
| Id  | Operation         | Name      | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |           |     1 |    72 |     3   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| EMPLOYEES |     1 |    72 |     3   (0)| 00:00:01 |
-------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter("EMPLOYEE_ID"=196)
 
Note
-----
   - SQL profile "coe_bvxjvrwqcq3pn_1613276963" used for this statement
 
 
Statistics
----------------------------------------------------------
         96  recursive calls
          0  db block gets
        150  consistent gets
          0  physical reads
          0  redo size
       1304  bytes sent via SQL*Net to client
        524  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          6  sorts (memory)
          0  sorts (disk)
          1  rows processed
 
SQL> alter index hr.emp_idx_testsqlp visible;
 
Index altered.
 
SQL> select * from hr.employees where employee_id=196;
 
 
Execution Plan
----------------------------------------------------------
Plan hash value: 1613276963
 
------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name             | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |                  |     1 |    72 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEES        |     1 |    72 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | EMP_IDX_TESTSQLP |     1 |       |     1   (0)| 00:00:01 |
------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("EMPLOYEE_ID"=196)
 
Note
-----
   - SQL profile "coe_bvxjvrwqcq3pn_1613276963" used for this statement
 
 
Statistics
----------------------------------------------------------
         96  recursive calls
          0  db block gets
        146  consistent gets
          0  physical reads
          0  redo size
       1308  bytes sent via SQL*Net to client
        524  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          6  sorts (memory)
          0  sorts (disk)
          1  rows processed

```
		  
如上所示，invisible索引之后，SQL照常运行，note也提示SQLPROFILE被采用，但是执行计划不会按照 SQLPROFILE中的方式运行。visible之后，SQLPROFILE自动按照绑定之后的情况运行。


由于篇幅原因，先测试到这里，下一篇将继续进行测试~


### To be continued...



![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com

