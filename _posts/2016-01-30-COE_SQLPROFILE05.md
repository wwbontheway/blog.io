---
layout: post
title: SQL_PROFILE的绑定及测试(五)
date: 2016-01-30
categories: OracleScripts
tags: [Oracle,Scripts]
description: 优化基础。
---


上一篇博文测试了使用coe_xfr_sql_profile.sql脚本绑定执行计划之后，Sql Profile的自动修复性，这篇将对drop索引这一操作对Sql Profile的影响来进行测试~

## 测试SQLPROFILE自动修复性

### 4.将索引DROP

```sql

SQL> drop index hr.emp_idx_testsqlp;
 
Index dropped.
 
SQL> select * from hr.employees where employee_id=196;
 
 
Execution Plan
----------------------------------------------------------
Plan hash value: 1833546154
 
---------------------------------------------------------------------------------------------
| Id  | Operation                   | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |               |     1 |    72 |     1   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEES     |     1 |    72 |     1   (0)| 00:00:01 |
|*  2 |   INDEX UNIQUE SCAN         | EMP_EMP_ID_PK |     1 |       |     0   (0)| 00:00:01 |
---------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("EMPLOYEE_ID"=196)
 
Note
-----
   - SQL profile "coe_bvxjvrwqcq3pn_1613276963" used for this statement
 
 
Statistics
----------------------------------------------------------
         95  recursive calls
          0  db block gets
        140  consistent gets
          0  physical reads
          0  redo size
       1172  bytes sent via SQL*Net to client
        513  bytes received via SQL*Net from client
          1  SQL*Net roundtrips to/from client
          6  sorts (memory)
          0  sorts (disk)
          1  rows processed
 
 
SQL> create index hr.emp_idx_testsqlp on hr.employees(employee_id,department_id);
 
Index created.
 
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
          8  recursive calls
          0  db block gets
          5  consistent gets
          0  physical reads
          0  redo size
       1308  bytes sent via SQL*Net to client
        524  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed
		  
```		  
		  
如上所见，drop索引之后，SQL照常运行，note也提示SQLPROFILE被采用，但是执行计划不会按照 SQLPROFILE中的方式运行。重新建立同名的索引之后，SQLPROFILE自动按照绑定之后的情况运行。


### To be continued...



![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com
