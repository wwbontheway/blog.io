---
layout: post
title: SQL_PROFILE的绑定及测试(四)
date: 2016-03-21
categories: OracleScripts
tags: [Oracle,Scripts]
description: 优化基础。
---

继续进行测试。

上一篇博文测试了使用coe_xfr_sql_profile.sql脚本绑定执行计划之后，Sql Profile的自动修复性，这篇将接着上一篇继续进行别的情况的测试~


## 测试SQLPROFILE自动修复性

### 3.将索引enable/disable

这两个选项仅针对函数索引，这里重新绑定个新索引进行测试。

```sql

SQL> set lines 180 
SQL> col index_owner for a12 
SQL> col column_name for a16 
SQL> select b.INDEX_OWNER,
  2         b.INDEX_NAME,
  3         b.TABLE_NAME,
  4         b.COLUMN_NAME,
  5         b.COLUMN_POSITION,
  6         a.LAST_ANALYZED,a.status,a.INDEX_TYPE
  7    from dba_indexes a, dba_ind_columns b 
  8   where a.table_name = b.table_name 
  9     and a.index_name = b.index_name 
 10     and a.owner = b.index_owner 
 11     and b.table_name = upper('&table_name') 
 12   order by index_owner, index_name, column_position; 
Enter value for table_name: employees
old  11:    and b.table_name = upper('&table_name')
new  11:    and b.table_name = upper('employees')
 
INDEX_OWNER  INDEX_NAME          TABLE_NAME   COLUMN_NAME      COLUMN_POSITION LAST_ANAL STATUS   INDEX_TYPE
------------ ------------------- ------------ ---------------- --------------- --------- -------- ---------------------
HR           EMP_DEPARTMENT_IX   EMPLOYEES    DEPARTMENT_ID                  1 21-DEC-16 VALID    NORMAL
HR           EMP_EMAIL_UK        EMPLOYEES    EMAIL                          1 21-DEC-16 VALID    NORMAL
HR           EMP_EMP_ID_PK       EMPLOYEES    EMPLOYEE_ID                    1 21-DEC-16 VALID    NORMAL
HR           EMP_EMP_ID_PK_DESC  EMPLOYEES    SYS_NC00012$                   1 21-DEC-16 VALID    FUNCTION-BASED NORMAL
HR           EMP_IDX_TESTSQLP    EMPLOYEES    EMPLOYEE_ID                    1 28-DEC-16 VALID    NORMAL
HR           EMP_IDX_TESTSQLP    EMPLOYEES    DEPARTMENT_ID                  2 28-DEC-16 VALID    NORMAL
HR           EMP_JOB_IX          EMPLOYEES    JOB_ID                         1 21-DEC-16 VALID    NORMAL
HR           EMP_MANAGER_IX      EMPLOYEES    MANAGER_ID                     1 21-DEC-16 VALID    NORMAL
HR           EMP_NAME_IX         EMPLOYEES    LAST_NAME                      1 21-DEC-16 VALID    NORMAL
HR           EMP_NAME_IX         EMPLOYEES    FIRST_NAME                     2 21-DEC-16 VALID    NORMAL
OE           EMP_MANID_DEID      EMPLOYEES    MANAGER_ID                     1 21-DEC-16 VALID    NORMAL
OE           EMP_MANID_DEID      EMPLOYEES    DEPARTMENT_ID                  2 21-DEC-16 VALID    NORMAL
 
12 rows selected.
 
SQL> set long 1000
SQL> select dbms_metadata.get_ddl('INDEX','EMP_EMP_ID_PK_DESC','HR') from dual;
 
DBMS_METADATA.GET_DDL('INDEX','EMP_EMP_ID_PK_DESC','HR')
--------------------------------------------------------------------------------
 
  CREATE INDEX "HR"."EMP_EMP_ID_PK_DESC" ON "HR"."EMPLOYEES" ("EMPLOYEE_ID" DESC
)
  PCTFREE 10 INITRANS 2 MAXTRANS 255 COMPUTE STATISTICS
  STORAGE(INITIAL 65536 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645
  PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1
  BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT)
  TABLESPACE "USERS"
  
```
  
如上图，EMP_EMP_ID_PK_DESC为employees表的employee_id列的一个函数索引，下面将他绑定到之前的SQL上。

```sql

SQL> select /*+index(employees EMP_EMP_ID_PK_DESC)*/* from hr.employees where employee_id=196;
 
 
Execution Plan
----------------------------------------------------------
Plan hash value: 394882801
 
--------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name               | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |                    |     1 |    72 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEES          |     1 |    72 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | EMP_EMP_ID_PK_DESC |     1 |       |     1   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access(SYS_OP_DESCEND("EMPLOYEE_ID")=HEXTORAW('3DFD9EFF') )
       filter(SYS_OP_UNDESCEND(SYS_OP_DESCEND("EMPLOYEE_ID"))=196)
 
 
Statistics
----------------------------------------------------------
        197  recursive calls
          0  db block gets
        374  consistent gets
          1  physical reads
          0  redo size
       1304  bytes sent via SQL*Net to client
        524  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
         33  sorts (memory)
          0  sorts (disk)
          1  rows processed
		  
```
		  
获取plan_hash_value：394882801，绑定到sql_id：bvxjvrwqcq3pn上。

```sql

SQL>  @/home/oracle/coe_xfr_sql_profile.sql
 
Parameter 1:
SQL_ID (required)
 
Enter value for 1: bvxjvrwqcq3pn
 
 
PLAN_HASH_VALUE AVG_ET_SECS
--------------- -----------
     1613276963         .02
 
Parameter 2:
PLAN_HASH_VALUE (required)
 
Enter value for 2: 394882801
 
Values passed to coe_xfr_sql_profile:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
SQL_ID         : "bvxjvrwqcq3pn"
PLAN_HASH_VALUE: "394882801"
 
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
 
Execute coe_xfr_sql_profile_bvxjvrwqcq3pn_394882801.sql
on TARGET system in order to create a custom SQL Profile
with plan 394882801 linked to adjusted sql_text.
 
COE_XFR_SQL_PROFILE completed.

```

如上，生成脚本的时候，他当前的plan_hash_value为之前绑定的SQLPROFILE的值，现在更改脚本中force match为true之后，覆盖绑定。

```sql

SQL>@/home/oracle/coe_xfr_sql_profile_bvxjvrwqcq3pn_394882801.sql
SQL>REM
SQL>REM $Header: 215187.1 coe_xfr_sql_profile_bvxjvrwqcq3pn_394882801.sql 11.4.4.4 2016/12/28 carlos.sierra $
SQL>REM
SQL>REM Copyright (c) 2000-2012, Oracle Corporation. All rights reserved.
SQL>REM
SQL>REM AUTHOR
SQL>REM   carlos.sierra@oracle.com
SQL>REM
SQL>REM SCRIPT
SQL>REM   coe_xfr_sql_profile_bvxjvrwqcq3pn_394882801.sql
SQL>REM
SQL>REM DESCRIPTION
SQL>REM   This script is generated by coe_xfr_sql_profile.sql
SQL>REM   It contains the SQL*Plus commands to create a custom
SQL>REM   SQL Profile for SQL_ID bvxjvrwqcq3pn based on plan hash
SQL>REM   value 394882801.
SQL>REM   The custom SQL Profile to be created by this script
SQL>REM   will affect plans for SQL commands with signature
SQL>REM   matching the one for SQL Text below.
SQL>REM   Review SQL Text and adjust accordingly.
SQL>REM
SQL>REM PARAMETERS
SQL>REM   None.
SQL>REM
SQL>REM EXAMPLE
SQL>REM   SQL> START coe_xfr_sql_profile_bvxjvrwqcq3pn_394882801.sql;
SQL>REM
SQL>REM NOTES
SQL>REM   1. Should be run as SYSTEM or SYSDBA.
SQL>REM   2. User must have CREATE ANY SQL PROFILE privilege.
SQL>REM   3. SOURCE and TARGET systems can be the same or similar.
SQL>REM   4. To drop this custom SQL Profile after it has been created:
SQL>REM  EXEC DBMS_SQLTUNE.DROP_SQL_PROFILE('coe_bvxjvrwqcq3pn_394882801');
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
 23  q'[INDEX_RS_ASC(@"SEL$1" "EMPLOYEES"@"SEL$1" "EMP_EMP_ID_PK_DESC")]',
 24  q'[END_OUTLINE_DATA]');
 25  :signature := DBMS_SQLTUNE.SQLTEXT_TO_SIGNATURE(sql_txt);
 26  :signaturef := DBMS_SQLTUNE.SQLTEXT_TO_SIGNATURE(sql_txt, TRUE);
 27  DBMS_SQLTUNE.IMPORT_SQL_PROFILE (
 28  sql_text    => sql_txt,
 29  profile     => h,
 30  name        => 'coe_bvxjvrwqcq3pn_394882801',
 31  description => 'coe bvxjvrwqcq3pn 394882801 '||:signature||' '||:signaturef||'',
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
 
 
COE_XFR_SQL_PROFILE_bvxjvrwqcq3pn_394882801 completed

```

如上，新的SQLPROFILE已经绑定完成。
测试一下：

```sql

SQL> select * from hr.employees where employee_id=196;
 
 
Execution Plan
----------------------------------------------------------
Plan hash value: 394882801
 
--------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name               | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |                    |     1 |    72 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEES          |     1 |    72 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | EMP_EMP_ID_PK_DESC |     1 |       |     1   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access(SYS_OP_DESCEND("EMPLOYEE_ID")=HEXTORAW('3DFD9EFF') )
       filter(SYS_OP_UNDESCEND(SYS_OP_DESCEND("EMPLOYEE_ID"))=196)
 
Note
-----
   - SQL profile "coe_bvxjvrwqcq3pn_394882801" used for this statement
 
 
Statistics
----------------------------------------------------------
          8  recursive calls
          0  db block gets
          5  consistent gets
          0  physical reads
          0  redo size
       1304  bytes sent via SQL*Net to client
        524  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed

```

现在这条SQL已经选择新的SQLPROFILE中的索引了，下面将这个索引disable掉。
 
```sql

SQL> alter index hr.EMP_EMP_ID_PK_DESC disable;
 
Index altered.
 
SQL> select * from hr.employees where employee_id=196;
select * from hr.employees where employee_id=196
*
ERROR at line 1:
ORA-30554: function-based index HR.EMP_EMP_ID_PK_DESC is disabled
 
 
SQL> alter index hr.EMP_EMP_ID_PK_DESC enable;
 
Index altered.
 
SQL> select * from hr.employees where employee_id=196;
 
 
Execution Plan
----------------------------------------------------------
Plan hash value: 394882801
 
--------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name               | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |                    |     1 |    72 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEES          |     1 |    72 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | EMP_EMP_ID_PK_DESC |     1 |       |     1   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access(SYS_OP_DESCEND("EMPLOYEE_ID")=HEXTORAW('3DFD9EFF') )
       filter(SYS_OP_UNDESCEND(SYS_OP_DESCEND("EMPLOYEE_ID"))=196)
 
Note
-----
   - SQL profile "coe_bvxjvrwqcq3pn_394882801" used for this statement
 
 
Statistics
----------------------------------------------------------
          8  recursive calls
          0  db block gets
          5  consistent gets
          0  physical reads
          0  redo size
       1304  bytes sent via SQL*Net to client
        524  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed
		  
```
		  
如上所示，函数索引disable之后，绑定SQLPROFILE的SQL会报错：ORA-30554: function-based index HR.EMP_EMP_ID_PK_DESC is disabled，将索引enable之后，恢复正常。
进一步测试：

```sql

SQL> alter index hr.EMP_EMP_ID_PK_DESC disable;
 
Index altered.
 
SQL> select * from hr.employees where employee_id=196;
select * from hr.employees where employee_id=196
*
ERROR at line 1:
ORA-30554: function-based index HR.EMP_EMP_ID_PK_DESC is disabled
 
 
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
          1  recursive calls
          0  db block gets
          1  consistent gets
          0  physical reads
          0  redo size
        530  bytes sent via SQL*Net to client
        524  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed

```
		  
如上，将函数索引disable之后，只有绑定了有关这个索引SQLPROFILE的sql会受到影响并报错，其他sql会根据剩下的可用索引进行执行计划的选择而不会收到影响。




由于篇幅原因，先测试到这里，下一篇将继续进行测试~


### To be continued...



![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com
