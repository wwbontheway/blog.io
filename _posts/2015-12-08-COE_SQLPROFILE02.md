---
layout: post
title: SQL_PROFILE的绑定及测试(二)
date: 2015-12-08
categories: OracleScripts
tags: [Oracle,Scripts]
description: 优化基础。
---

上一篇博文将coe_xfr_sql_profile.sql脚本的正文发了上来，那么这篇就开始进行这个脚本的使用方法的演示测试。


## SQLPROFILE绑定测试

```sql

SQL> create index hr.emp_idx_testsqlp on hr.employees(employee_id,department_id);
 
Index created.
 
SQL> set autot trace 
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
 
 
Statistics
----------------------------------------------------------
        100  recursive calls
          0  db block gets
        211  consistent gets
          3  physical reads
          0  redo size
       1172  bytes sent via SQL*Net to client
        513  bytes received via SQL*Net from client
          1  SQL*Net roundtrips to/from client
         23  sorts (memory)
          0  sorts (disk)
          1  rows processed
 
SQL> select /*+index(employees emp_idx_testsqlp)*/* from hr.employees where employee_id=196;
 
 
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
		  
如上，先建立了一个组合索引，然后跑SQL，默认跑的是单列索引，然后使用hint让SQL运行时候采用新建立的组合索引。
下面绑定使用组合索引的SQLPROFILE
先查出sql_id：

```sql

SQL> select sql_id,sql_text from v$sqlarea where sql_text like '%employee_id%';
 
SQL_ID
-------------
SQL_TEXT
---------------------------------------------------------
 
bvxjvrwqcq3pn
select * from hr.employees where employee_id=196
 
gyxyv4tuxfanh
select /*+index(employees emp_idx_testsqlp)*/* from hr.employees where employee_id=196


```

如上，得到了需要绑定的sql_id为bvxjvrwqcq3pn，然后之前的sql中看到组合索引执行计划的plan_hash_value为1613276963，下面运行脚本：

```sql

SQL> !ls -lrt ~/    
total 20
-rwxrwxrwx 1 oracle dba 18732 Dec 28 10:50 coe_xfr_sql_profile.sql
SQL> @/home/oracle/coe_xfr_sql_profile.sql
 
Parameter 1:
SQL_ID (required)
 
Enter value for 1: bvxjvrwqcq3pn
 
 
PLAN_HASH_VALUE AVG_ET_SECS
--------------- -----------
     1833546154        .012
 
Parameter 2:
PLAN_HASH_VALUE (required)
 
Enter value for 2: 1613276963
 
Values passed to coe_xfr_sql_profile:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
SQL_ID         : "bvxjvrwqcq3pn"
PLAN_HASH_VALUE: "1613276963"
 
SQL>BEGIN
  2    IF :sql_text IS NULL THEN
  3      RAISE_APPLICATION_ERROR(-20100, 'SQL_TEXT for SQL_ID &&sql_id. was not found in memory (gv$sqltext_with_newlines) or AWR (dba_hist_sqltext).');
  4    END IF;
  5  END;
  6  /
SQL>SET TERM OFF;
SQL>BEGIN
  2    IF :other_xml IS NULL THEN
  3      RAISE_APPLICATION_ERROR(-20101, 'PLAN for SQL_ID &&sql_id. and PHV &&plan_hash_value. was not found in memory (gv$sql_plan) or AWR (dba_hist_sql_plan).');
  4    END IF;
  5  END;
  6  /
SQL>SET TERM OFF;
 
Execute coe_xfr_sql_profile_bvxjvrwqcq3pn_1613276963.sql
on TARGET system in order to create a custom SQL Profile
with plan 1613276963 linked to adjusted sql_text.
 
 
COE_XFR_SQL_PROFILE completed.

```

如上，生成了coe_xfr_sql_profile_bvxjvrwqcq3pn_1613276963.sql脚本，下面使用vi编辑这个脚本，将force_match => FALSE改为force_match => TRUE（更改这个的原因是这个参数值为FLASE，就等于是初始化参数CURSOR_SHARING为EXACT。但是这个脚本中的SQL文本并不是严格与数据库中SQL一样，可能会存在多空格、换行等待情况，所以为了生效，需要改为TURE，即相当于初始化参数CURSOR_SHARING为FORCE，即只要SQL文本中的文字是相同的就进行匹配）
然后运行这个脚本后，再次执行SQL：

```sql

COE_XFR_SQL_PROFILE completed.
SQL>@/home/oracle/coe_xfr_sql_profile_bvxjvrwqcq3pn_1613276963.sql
SQL>REM
SQL>REM $Header: 215187.1 coe_xfr_sql_profile_bvxjvrwqcq3pn_1613276963.sql 11.4.4.4 2016/12/28 carlos.sierra $
SQL>REM
SQL>REM Copyright (c) 2000-2012, Oracle Corporation. All rights reserved.
SQL>REM
SQL>REM AUTHOR
SQL>REM   carlos.sierra@oracle.com
SQL>REM
SQL>REM SCRIPT
SQL>REM   coe_xfr_sql_profile_bvxjvrwqcq3pn_1613276963.sql
SQL>REM
SQL>REM DESCRIPTION
SQL>REM   This script is generated by coe_xfr_sql_profile.sql
SQL>REM   It contains the SQL*Plus commands to create a custom
SQL>REM   SQL Profile for SQL_ID bvxjvrwqcq3pn based on plan hash
SQL>REM   value 1613276963.
SQL>REM   The custom SQL Profile to be created by this script
SQL>REM   will affect plans for SQL commands with signature
SQL>REM   matching the one for SQL Text below.
SQL>REM   Review SQL Text and adjust accordingly.
SQL>REM
SQL>REM PARAMETERS
SQL>REM   None.
SQL>REM
SQL>REM EXAMPLE
SQL>REM   SQL> START coe_xfr_sql_profile_bvxjvrwqcq3pn_1613276963.sql;
SQL>REM
SQL>REM NOTES
SQL>REM   1. Should be run as SYSTEM or SYSDBA.
SQL>REM   2. User must have CREATE ANY SQL PROFILE privilege.
SQL>REM   3. SOURCE and TARGET systems can be the same or similar.
SQL>REM   4. To drop this custom SQL Profile after it has been created:
SQL>REM  EXEC DBMS_SQLTUNE.DROP_SQL_PROFILE('coe_bvxjvrwqcq3pn_1613276963');
SQL>REM   5. Be aware that using DBMS_SQLTUNE requires a license
SQL>REM  for the Oracle Tuning Pack.
SQL>REM   6. If you modified a SQL putting Hints in order to produce a desired
SQL>REM  Plan, you can remove the artifical Hints from SQL Text pieces below.
SQL>REM  By doing so you can create a custom SQL Profile for the original
SQL>REM  SQL but with the Plan captured from the modified SQL (with Hints).
SQL>REM
SQL>WHENEVER SQLERROR EXIT SQL.SQLCODE;
SQL>REM
SQL>VAR signature NUMBER;
SQL>VAR signaturef NUMBER;
SQL>REM
SQL>DECLARE
  2  sql_txt CLOB;
  3  h       SYS.SQLPROF_ATTR;
  4  PROCEDURE wa (p_line IN VARCHAR2) IS
  5  BEGIN
  6  DBMS_LOB.WRITEAPPEND(sql_txt, LENGTH(p_line), p_line);
  7  END wa;
  8  BEGIN
  9  DBMS_LOB.CREATETEMPORARY(sql_txt, TRUE);
 10  DBMS_LOB.OPEN(sql_txt, DBMS_LOB.LOB_READWRITE);
 11  -- SQL Text pieces below do not have to be of same length.
 12  -- So if you edit SQL Text (i.e. removing temporary Hints),
 13  -- there is no need to edit or re-align unmodified pieces.
 14  wa(q'[select * from hr.employees where employee_id=196]');
 15  DBMS_LOB.CLOSE(sql_txt);
 16  h := SYS.SQLPROF_ATTR(
 17  q'[BEGIN_OUTLINE_DATA]',
 18  q'[IGNORE_OPTIM_EMBEDDED_HINTS]',
 19  q'[OPTIMIZER_FEATURES_ENABLE('11.2.0.4')]',
 20  q'[DB_VERSION('11.2.0.4')]',
 21  q'[ALL_ROWS]',
 22  q'[OUTLINE_LEAF(@"SEL$1")]',
 23  q'[INDEX_RS_ASC(@"SEL$1" "EMPLOYEES"@"SEL$1" ("EMPLOYEES"."EMPLOYEE_ID" "EMPLOYEES"."DEPARTMENT_ID"))]',
 24  q'[END_OUTLINE_DATA]');
 25  :signature := DBMS_SQLTUNE.SQLTEXT_TO_SIGNATURE(sql_txt);
 26  :signaturef := DBMS_SQLTUNE.SQLTEXT_TO_SIGNATURE(sql_txt, TRUE);
 27  DBMS_SQLTUNE.IMPORT_SQL_PROFILE (
 28  sql_text    => sql_txt,
 29  profile     => h,
 30  name        => 'coe_bvxjvrwqcq3pn_1613276963',
 31  description => 'coe bvxjvrwqcq3pn 1613276963 '||:signature||' '||:signaturef||'',
 32  category    => 'DEFAULT',
 33  validate    => TRUE,
 34  replace     => TRUE,
 35  force_match => TRUE /* TRUE:FORCE (match even when different literals in SQL). FALSE:EXACT (similar to CURSOR_SHARING) */ );
 36  DBMS_LOB.FREETEMPORARY(sql_txt);
 37  END;
 38  /
 
PL/SQL procedure successfully completed.
 
SQL>WHENEVER SQLERROR CONTINUE
SQL>SET ECHO OFF;
 
            SIGNATURE
---------------------
 11153841996594160133
 
 
           SIGNATUREF
---------------------
 18334186158832369273
 
 
... manual custom SQL Profile has been created
 
 
COE_XFR_SQL_PROFILE_bvxjvrwqcq3pn_1613276963 completed
 
SQL>set autot trace
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
        131  recursive calls
          0  db block gets
        152  consistent gets
          1  physical reads
          0  redo size
       1308  bytes sent via SQL*Net to client
        524  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          7  sorts (memory)
          0  sorts (disk)
          1  rows processed

```
		  
注意NOTE部分的提示，表示SQLPROFILE已经绑定成功并被采用了。

这就是这个脚本的使用方法，关于其他测试，我将在下一篇博文中进行测试。

### To be continued...



![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com

