---
layout: post
title: 错误的表连接方式-例2
date: 2016-09-09
categories: OracleTuning
tags: [Oracle,Tuning]
description: 优化案例。
---

HINT，往往可以提升很多SQL运行的性能，有时候ORACLE不是不智能，只是给他的计算的数据是陈旧的，所以他就会算错，就会估计错，执行顺序就会错。但是在统计信息没有办法实时更新到最新的情况下，HINT是最快的纠正ORACLE错误的方法（个人感觉哈~）。
SQL文本如下：

```sql

select to_char(a.login_accept) || '~' || trim(a.phone_no) || '~' ||
        a.sm_code || '
~' || a.op_code || '~' || a.pay_money || '~' || a.op_time || '~' ||
        a.login_no || '~' || substr(b.org_code, 1, 2)
  from wLoginOpr201608 a, dloginmsg b, dchngroupmsg c
 where a.op_code in ('1302', '1300', '1310')
   and (a.login_no like 'a%' or a.login_no like 'b%' or
       a.login_no like 'c%' or a.login_no like 'd%' or
       a.login_no like 'e%' or a.login_no like 'f%' or
       a.login_no like 'g%' or a.login_no like 'h%' or
       a.login_no like 'i%' or a.login_no like 'j%' or
       a.login_no like 'k%' or a.login_no like 'l%' or
       a.login_no like 'm%')
   and a.login_no = b.login_no
   and b.group_id = c.group_id
   and a.op_time between to_date(to_char(sysdate - 1 / 24, 'YYYYMMDD hh24'), 'YYYYMMDD hh24:mi:ss') and to_date(to_char(sysdate - 1 / 24, 'YYYYMMDD hh24') || ':59:59','YYYYMMDD hh24:mi:ss')
   
```
   
这条SQL的执行计划如下：

```sql

Plan hash value: 1214013918

------------------------------------------------------------------------
| Id  | Operation            | Name            | Rows  | Bytes | Cost  |
------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |                 |     9 |   747 | 11321 |
|*  1 |  FILTER              |                 |       |       |       |
|   2 |   NESTED LOOPS       |                 |     9 |   747 | 11321 |
|*  3 |    HASH JOIN         |                 |     9 |   675 | 11321 |
|*  4 |     TABLE ACCESS FULL| WLOGINOPR201608 |     9 |   459 | 10979 |
|   5 |     TABLE ACCESS FULL| DLOGINMSG       |   112K|  2646K|   341 |
|*  6 |    INDEX UNIQUE SCAN | DCHNGROUPMSG_PK |     1 |     8 |       |
------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter(TO_DATE(TO_CHAR(SYSDATE@!-.0416666666666666666666666666666
              666666667,'YYYYMMDD hh24'),'YYYYMMDD
              hh24:mi:ss')<=TO_DATE(TO_CHAR(SYSDATE@!-.0416666666666666666666666666666
              666666667,'YYYYMMDD hh24')||':59:59','YYYYMMDD hh24:mi:ss'))
   3 - access("A"."LOGIN_NO"="B"."LOGIN_NO")
   4 - filter(("A"."OP_CODE"='1300' OR "A"."OP_CODE"='1302' OR
              "A"."OP_CODE"='1310') AND ("A"."LOGIN_NO" LIKE 'a%' OR "A"."LOGIN_NO"
              LIKE 'b%' OR "A"."LOGIN_NO" LIKE 'c%' OR "A"."LOGIN_NO" LIKE 'd%' OR
              "A"."LOGIN_NO" LIKE 'e%' OR "A"."LOGIN_NO" LIKE 'f%' OR "A"."LOGIN_NO"
              LIKE 'g%' OR "A"."LOGIN_NO" LIKE 'h%' OR "A"."LOGIN_NO" LIKE 'i%' OR
              "A"."LOGIN_NO" LIKE 'j%' OR "A"."LOGIN_NO" LIKE 'k%' OR "A"."LOGIN_NO"
              LIKE 'l%' OR "A"."LOGIN_NO" LIKE 'm%') AND
              "A"."OP_TIME">=TO_DATE(TO_CHAR(SYSDATE@!-.041666666666666666666666666666
              6666666667,'YYYYMMDD hh24'),'YYYYMMDD hh24:mi:ss') AND
              "A"."OP_TIME"<=TO_DATE(TO_CHAR(SYSDATE@!-.041666666666666666666666666666
              6666666667,'YYYYMMDD hh24')||':59:59','YYYYMMDD hh24:mi:ss'))
   6 - access("B"."GROUP_ID"="C"."GROUP_ID")
   
```   
   
此条SQL运行时间将近2分钟，逻辑读为 350139

查看这个SQL的where条件部分，OP_IIME部门虽然写的复杂，但是其实归根结底只是要查询一个小时的数据（准确来说是59分59秒的数据），但从表命名来看，wLoginOpr201608更像一个月度表，在月度表里面查询一小时的数据，按照常理就不需要去查询全表，可是执行计划中进行的全表扫描，可见这条SQL其实是可以更快的。那就尝试用HINT告诉ORACLE，表的连接方式是什么。

```sql

select /*+leading(a b c) use_nl(b) use_nl(c) */to_char(a.login_accept) || '~' || trim(a.phone_no) || '~' ||
        a.sm_code || '
~' || a.op_code || '~' || a.pay_money || '~' || a.op_time || '~' ||
        a.login_no || '~' || substr(b.org_code, 1, 2)
  from wLoginOpr201608 a, dloginmsg b, dchngroupmsg c
 where a.op_code in ('1302', '1300', '1310')
   and (a.login_no like 'a%' or a.login_no like 'b%' or
       a.login_no like 'c%' or a.login_no like 'd%' or
       a.login_no like 'e%' or a.login_no like 'f%' or
       a.login_no like 'g%' or a.login_no like 'h%' or
       a.login_no like 'i%' or a.login_no like 'j%' or
       a.login_no like 'k%' or a.login_no like 'l%' or
       a.login_no like 'm%')
   and a.login_no = b.login_no
   and b.group_id = c.group_id
   and a.op_time between to_date(to_char(sysdate - 1 / 24, 'YYYYMMDD hh24'),'YYYYMMDD hh24:mi:ss') and to_date(to_char(sysdate - 1 / 24,'YYYYMMDD hh24') || ':59:59', 'YYYYMMDD hh24:mi:ss')
   
```   
   
这条SQL现在的执行计划如下：

```sql

Plan hash value: 2228789382

-----------------------------------------------------------------------------------------
| Id  | Operation                      | Name                   | Rows  | Bytes | Cost  |
-----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |                        |     9 |   747 |    13 |
|*  1 |  FILTER                        |                        |       |       |       |
|   2 |   NESTED LOOPS                 |                        |     9 |   747 |    13 |
|   3 |    NESTED LOOPS                |                        |     9 |   675 |    13 |
|*  4 |     TABLE ACCESS BY INDEX ROWID| WLOGINOPR201608        |     9 |   459 |     4 |
|*  5 |      INDEX RANGE SCAN          | WLOGINOPR201608_OPTIME |     1 |       |     3 |
|   6 |     TABLE ACCESS BY INDEX ROWID| DLOGINMSG              |     1 |    24 |     1 |
|*  7 |      INDEX UNIQUE SCAN         | DLOGINMSG_PK           |     1 |       |       |
|*  8 |    INDEX UNIQUE SCAN           | DCHNGROUPMSG_PK        |     1 |     8 |       |
-----------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter(TO_DATE(TO_CHAR(SYSDATE@!-.0416666666666666666666666666666666666667
              ,'YYYYMMDD hh24'),'YYYYMMDD hh24:mi:ss')<=TO_DATE(TO_CHAR(SYSDATE@!-.041666666666
              6666666666666666666666666667,'YYYYMMDD hh24')||':59:59','YYYYMMDD hh24:mi:ss'))
   4 - filter(("A"."OP_CODE"='1300' OR "A"."OP_CODE"='1302' OR
              "A"."OP_CODE"='1310') AND ("A"."LOGIN_NO" LIKE 'a%' OR "A"."LOGIN_NO" LIKE 'b%'
              OR "A"."LOGIN_NO" LIKE 'c%' OR "A"."LOGIN_NO" LIKE 'd%' OR "A"."LOGIN_NO" LIKE
              'e%' OR "A"."LOGIN_NO" LIKE 'f%' OR "A"."LOGIN_NO" LIKE 'g%' OR "A"."LOGIN_NO"
              LIKE 'h%' OR "A"."LOGIN_NO" LIKE 'i%' OR "A"."LOGIN_NO" LIKE 'j%' OR
              "A"."LOGIN_NO" LIKE 'k%' OR "A"."LOGIN_NO" LIKE 'l%' OR "A"."LOGIN_NO" LIKE
              'm%'))
   5 - access("A"."OP_TIME">=TO_DATE(TO_CHAR(SYSDATE@!-.0416666666666666666666666
              666666666666667,'YYYYMMDD hh24'),'YYYYMMDD hh24:mi:ss') AND
              "A"."OP_TIME"<=TO_DATE(TO_CHAR(SYSDATE@!-.041666666666666666666666666666666666666
              7,'YYYYMMDD hh24')||':59:59','YYYYMMDD hh24:mi:ss'))
   7 - access("A"."LOGIN_NO"="B"."LOGIN_NO")
   8 - access("B"."GROUP_ID"="C"."GROUP_ID")
   
```   
   
此时的SQL执行时间从原来的2分钟降到2秒左右，逻辑读也有所下降，为207153。那么为何逻辑读没有下降很多呢？主要原因出现在索引回表上。但对于这种SQL，这种程度的优化就完全可以了。


![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com



