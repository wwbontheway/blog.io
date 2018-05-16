---
layout: post
title: SQL_PROFILE的绑定及测试(六)
date: 2016-02-05
categories: OracleScripts
tags: [Oracle,Scripts]
description: 优化基础。
---


上一篇博文测试了使用coe_xfr_sql_profile.sql脚本绑定执行计划之后，Sql Profile的自动修复性，这篇将对MOVE表这一操作对Sql Profile的影响来进行测试，同时也会介绍一下如何删除SQL PROFILE，毕竟在大多数情况下，让Oracle自动感知最佳执行计划才是最推荐的~

## 测试SQLPROFILE自动修复性

### 5.测试MOVE表对SQLPROFILE的影响

新建立一个测试表，建立2个索引，生成两个执行计划：

```sql

SQL> create table hr.test_tabmove as select * from hr.employees;
 
Table created.
 
SQL> create index hr.tabmove_idx on hr.test_tabmove(employee_id);
 
Index created.
 
SQL> create index hr.tabmove_idx2 on hr.test_tabmove(employee_id,1);
 
Index created.
 
SQL> exec dbms_stats.gather_table_stats('HR','TEST_TABMOVE');
 
PL/SQL procedure successfully completed.
 
SQL> set autot trace exp
SQL> select * from hr.test_tabmove where employee_id=196;
 
Execution Plan
----------------------------------------------------------
Plan hash value: 615278306
--------------------------------------------------------------------------------------------
| Id  | Operation                   | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |              |     1 |    69 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| TEST_TABMOVE |     1 |    69 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | TABMOVE_IDX  |     1 |       |     1   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("EMPLOYEE_ID"=196)
 
SQL> select /*+index(test_tabmove tabmove_idx2)*/* from hr.test_tabmove where employee_id=196;
 
Execution Plan
----------------------------------------------------------
Plan hash value: 2619857068
 
--------------------------------------------------------------------------------------------
| Id  | Operation                   | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |              |     1 |    69 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| TEST_TABMOVE |     1 |    69 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | TABMOVE_IDX2 |     1 |       |     1   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("EMPLOYEE_ID"=196)
   
```   
   
现在将索引tabmove_idx2的执行计划作为SQLPROFILE（plan_hash_value：2619857068）绑定给select \* from hr.test_tabmove where employee_id=196;（sql_id：27zjv4hbcb9qz）



绑定过程省略，查看绑定之后的执行计划

```sql

SQL> set autot trace exp
SQL> select * from hr.test_tabmove where employee_id=196;
 
Execution Plan
----------------------------------------------------------
Plan hash value: 2619857068
 
--------------------------------------------------------------------------------------------
| Id  | Operation                   | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |              |     1 |    69 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| TEST_TABMOVE |     1 |    69 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | TABMOVE_IDX2 |     1 |       |     1   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("EMPLOYEE_ID"=196)
 
Note
-----
   - SQL profile "coe_27zjv4hbcb9qz_2619857068" used for this statement
   
```   
   
如上，执行计划已经绑定，下面将表TEST_TABMOVE进行MOVE操作。

```sql

SQL>alter table hr.test_tabmove move;
 
Table altered.
 
SQL>set lines 180 
SQL>col index_owner for a12 
SQL>col column_name for a16 
SQL>select b.INDEX_OWNER,
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
Enter value for table_name: test_tabmove
old  11:    and b.table_name = upper('&table_name')
new  11:    and b.table_name = upper('test_tabmove')
                                       
INDEX_OWNER  INDEX_NAME   TABLE_NAME    COLUMN_NAME   COLUMN_POSITION LAST_ANAL STATUS   INDEX_TYPE
------------ ------------ ------------- ------------- --------------- --------- -------- ----------------------
HR           TABMOVE_IDX  TEST_TABMOVE  EMPLOYEE_ID                 1 28-DEC-16 UNUSABLE NORMAL
HR           TABMOVE_IDX2 TEST_TABMOVE  EMPLOYEE_ID                 1 28-DEC-16 UNUSABLE FUNCTION-BASED NORMAL
HR           TABMOVE_IDX2 TEST_TABMOVE  SYS_NC00012$                2 28-DEC-16 UNUSABLE FUNCTION-BASED NORMAL

```

众所周知，move操作之后的表，索引会为unusable，需要重建，其实这个测试与之前将索引至为unusable类似。

```sql

SQL> set autot trace exp
SQL> select * from hr.test_tabmove where employee_id=196;
select * from hr.test_tabmove where employee_id=196
*
ERROR at line 1:
ORA-01502: index 'HR.TABMOVE_IDX2' or partition of such index is in unusable state
 
 
SQL> select employee_id from hr.test_tabmove where employee_id=196;
 
Execution Plan
----------------------------------------------------------
Plan hash value: 855004495
 
----------------------------------------------------------------------------------
| Id  | Operation         | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |              |     1 |     4 |     3   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| TEST_TABMOVE |     1 |     4 |     3   (0)| 00:00:01 |
----------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter("EMPLOYEE_ID"=196)
 
SQL> alter index hr.TABMOVE_IDX2 rebuild;
 
Index altered.
 
SQL> select * from hr.test_tabmove where employee_id=196;
 
Execution Plan
----------------------------------------------------------
Plan hash value: 2619857068
 
--------------------------------------------------------------------------------------------
| Id  | Operation                   | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |              |     1 |    69 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| TEST_TABMOVE |     1 |    69 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | TABMOVE_IDX2 |     1 |       |     1   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("EMPLOYEE_ID"=196)
 
Note
-----
   - SQL profile "coe_27zjv4hbcb9qz_2619857068" used for this statement
   
```   
   
如上，表move之后，由于索引失效，所以绑定SQLPROFILE的SQL会返回报错。索引重建之后自动回复正常。

### SQLPROFILE的删除

使用DBMS_SQLTUNE.DROP_SQL_PROFILE包

```sql

BEGIN 
  DBMS_SQLTUNE.DROP_SQL_PROFILE(name => 'PROFILE NAME'); 
END; 
/ 

```

例如：

```sql

SQL> select * from hr.test_tabmove where employee_id=196;
 
Execution Plan
----------------------------------------------------------
Plan hash value: 2619857068
 
--------------------------------------------------------------------------------------------
| Id  | Operation                   | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |              |     1 |    69 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| TEST_TABMOVE |     1 |    69 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | TABMOVE_IDX2 |     1 |       |     1   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("EMPLOYEE_ID"=196)
 
Note
-----
   - SQL profile "coe_27zjv4hbcb9qz_2619857068" used for this statement
 
SQL> BEGIN 
  2    DBMS_SQLTUNE.DROP_SQL_PROFILE(name => 'coe_27zjv4hbcb9qz_2619857068'); 
  3  END; 
  4  / 
 
PL/SQL procedure successfully completed.
 
SQL> select * from hr.test_tabmove where employee_id=196;
 
Execution Plan
----------------------------------------------------------
Plan hash value: 615278306
 
--------------------------------------------------------------------------------------------
| Id  | Operation                   | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |              |     1 |    69 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| TEST_TABMOVE |     1 |    69 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | TABMOVE_IDX  |     1 |       |     1   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("EMPLOYEE_ID"=196)
   
```   
   
如上，SQLPROFILE已经删除，sql的执行计划变为CBO自己选择的执行计划了。

### SQLPROFILE相关视图

dba_sql_profiles

10G：sys.sqlprof$attr   /    sqlprof$  
11G：sys.sqlobj$data  /    sys.sqlobj$

```sql

SQL> col created for a30
SQL> col category for a10
SQL> SELECT name, created, category, sql_Text
  2     from dba_sql_profiles
  3    ORDER BY created DESC;
 
NAME                           CREATED                       CATEGORY     SQL_TEXT
------------------------------ ---------------------------- ----------  -------------------------------------------------
coe_bvxjvrwqcq3pn_394882801    28-DEC-16 07.19.24.000000 PM   DEFAULT    select * from hr.employees where employee_id=196

```

以下基于11G

```sql

SQL> set long 10000
SQL> SELECT sql_attr.comp_data outline_hints
  2    FROM dba_sql_profiles sql_profiles, sys.sqlobj$data sql_attr
  3   WHERE sql_profiles.signature = sql_attr.signature
  4     AND sql_profiles.name = 'coe_bvxjvrwqcq3pn_394882801';
 
OUTLINE_HINTS
--------------------------------------------------------------------------------
<outline_data><hint><![CDATA[BEGIN_OUTLINE_DATA]]></hint><hint><![CDATA[IGNORE_O
PTIM_EMBEDDED_HINTS]]></hint><hint><![CDATA[OPTIMIZER_FEATURES_ENABLE('11.2.0.4'
)]]></hint><hint><![CDATA[DB_VERSION('11.2.0.4')]]></hint><hint><![CDATA[ALL_ROW
S]]></hint><hint><![CDATA[OUTLINE_LEAF(@"SEL$1")]]></hint><hint><![CDATA[INDEX_R
S_ASC(@"SEL$1" "EMPLOYEES"@"SEL$1" "EMP_EMP_ID_PK_DESC")]]></hint><hint><![CDATA
[END_OUTLINE_DATA]]></hint></outline_data>

```

将如上显示信息和SQL运行的outline对比一下：

```sql

SQL>set lines 300 pages 1000
SQL>select * from hr.employees where employee_id=196;
 
EMPLOYEE_ID FIRST_NAME  LAST_NAME EMAIL   PHONE_NUMBER   HIRE_DATE JOB_ID      SALARY COMMISSION_PCT MANAGER_ID DEPARTMENT_ID
----------- ----------- --------- ------- -------------- --------- ---------- ------- -------------- ---------- -------------
        196 Alana       Walsh     AWALSH  650.507.9811   24-APR-06 SH_CLERK      3100                  124             50
 
SQL>select * from table(dbms_xplan.display_cursor(null,null,'outline'));
 
PLAN_TABLE_OUTPUT
-------------------------------------------------------------------
SQL_ID  bvxjvrwqcq3pn, child number 0
-------------------------------------
select * from hr.employees where employee_id=196
 
Plan hash value: 394882801
 
--------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name               | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |                    |       |       |     2 (100)|          |
|   1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEES          |     1 |    72 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | EMP_EMP_ID_PK_DESC |     1 |       |     1   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------------
 
Outline Data
-------------
 
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.4')
      DB_VERSION('11.2.0.4')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$1")
      INDEX_RS_ASC(@"SEL$1" "EMPLOYEES"@"SEL$1" "EMP_EMP_ID_PK_DESC")
      END_OUTLINE_DATA
  */
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("EMPLOYEES"."SYS_NC00012$"=HEXTORAW('3DFD9EFF') )
       filter(SYS_OP_UNDESCEND("EMPLOYEES"."SYS_NC00012$")=196)
 
Note
-----
   - SQL profile coe_bvxjvrwqcq3pn_394882801 used for this statement
 
 
38 rows selected.

```

查出来的outline信息和sql运行的outline信息是一样的。

至此，关于这个超级好用的COE贡献分享的脚本的测试就告一段落了~


![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com



