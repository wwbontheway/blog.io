---
layout: post
title: 【优化】SQL中关联列的运算
date: 2016-08-03
categories: OracleTuning
tags: [Oracle,Tuning]
description: 优化案例。
---

在开发人员编写SQL的时候常常忽略一个重要的问题，就是查询列的数据类型，当遇见不同的数据类型的条件时候，开发人员往往会对列进行函数运算，这样会导致就算此列存在索引而且效率非常高的时候，ORACLE的优化器也不会选择索引使用（当然，有函数索引除外，这一面只讨论通常情况），下面这条SQL就是由于对列进行运算导致索引失效的一个小例子

SQL文本如下

```sql

select a.login_accept,
       a.op_code,
       a.phone_no,
       a.unit_code,
       a.login_no,
       a.login_name,
       a.ip,
       to_char(a.op_time, 'YYYY-MM-DD HH24:MI:SS'),
       a.op_note
  from wGrpLoginOpr a
 where 1 = 1
   and substr(login_no, 1, 1) = 'k'
   and a.login_no in (Select service_no
                        From dgrpmanagermsg
                       Where system_type In ('0', '2'))
   and to_char(a.op_time, 'YYYY-MM-DD') >= '2016-07-15'
   and to_char(a.op_time, 'YYYY-MM-DD') <= '2016-07-17'
 order by op_time desc
 
``` 
 
这条SQL的执行计划如下：

```sql

Plan hash value: 2141399410

----------------------------------------------------------------------
| Id  | Operation           | Name           | Rows  | Bytes | Cost  |
----------------------------------------------------------------------
|   0 | SELECT STATEMENT    |                |   636 |   121K| 90659 |
|   1 |  SORT ORDER BY      |                |   636 |   121K| 90659 |
|*  2 |   HASH JOIN         |                |   636 |   121K| 90493 |
|*  3 |    TABLE ACCESS FULL| DGRPMANAGERMSG |  3568 | 32112 |    20 |
|*  4 |    TABLE ACCESS FULL| WGRPLOGINOPR   |   863 |   156K| 90472 |
----------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("A"."LOGIN_NO"="SERVICE_NO")
   3 - filter("SYSTEM_TYPE"='0' OR "SYSTEM_TYPE"='2')
   4 - filter(SUBSTR("LOGIN_NO",1,1)='k' AND
              TO_CHAR(INTERNAL_FUNCTION("A"."OP_TIME"),'YYYY-MM-DD')>='2016-07-15'
              AND TO_CHAR(INTERNAL_FUNCTION("A"."OP_TIME"),'YYYY-MM-DD')<='2016-07-17'
              )
			  
```			  
			  
当时开发同事反馈给我这条SQL的时候，它的运行时间将近6分钟，逻辑读为 943307

这个SQL有些优化基础的朋友一眼就能看出问题所在，不过这里还是按照一般的处理思路去处理：

首先查看执行计划，两张进行关联的表均进行全表扫描，此时查看SQL中a表设计到的过滤列，发现均存在索引：

```sql

INDEX_OWNER INDEX_NAME                 TABLE_NAME   COLUMN_NAME  COL_POS LAST_ANALYZED  STATUS
----------- -------------------------- ------------ ------------ ------- -------------- ------
DBVIPADM    IDX_WGRPLOGINOPR           WGRPLOGINOPR LOGIN_ACCEPT       1 20160715015214 VALID
DBVIPADM    IDX_WGRPLOGINOPRLOGIN      WGRPLOGINOPR LOGIN_NO           1 20160715015239 VALID
DBVIPADM    IDX_WGRPLOGINOPROPCODE     WGRPLOGINOPR OP_CODE            1 20160715015251 VALID
DBVIPADM    IDX_WGRPLOGINOPROPTIME     WGRPLOGINOPR OP_TIME            1 20160715015202 VALID
DBVIPADM    IDX_WGRPLOGINOPRPHONE      WGRPLOGINOPR PHONE_NO           1 20160715015225 VALID
DBVIPADM    IDX_WGRPLOGINOPRUNIT       WGRPLOGINOPR UNIT_CODE          1 20160715015227 VALID
                                                                   
DBVIPADM     IDX_DGRPMANAGERMSGORGCODE DGRPMANAGERMSG ORG_CODE         1 20160715002118 VALID
DBVIPADM     IDX_DGRPMANAGERMSGSERVICE DGRPMANAGERMSG SERVICE_NO       1 20160715002118 VALID
DBVIPADM     IDX_DGRPMANAGERMSGSYSTEM  DGRPMANAGERMSG SYSTEM_TYPE      1 20160715002118 VALID

```

但是ORACLE为什么没有选择索引呢？原因是在SQL中对列进行了运算：

```sql

substr(login_no, 1, 1) = 'k'

```

和

```sql

and to_char(a.op_time, 'YYYY-MM-DD') >= '2016-07-15' and to_char(a.op_time, 'YYYY-MM-DD') <= '2016-07-17'

```

这两个条件。那么这条SQL的优化思路就有了，那就是对SQL进行改写，将对列的操作消除掉，剩下的就交给ORACLE自己来就可以了。
改写之后的SQL如下：

```sql

select a.login_accept,
       a.op_code,
       a.phone_no,
       a.unit_code,
       a.login_no,
       a.login_name,
       a.ip,
       to_char(a.op_time,'YYYY-MM-DD HH24:MI:SS'),
       a.op_note
  from wGrpLoginOpr a
 where 1 = 1
   and login_no like 'k%'
   and a.login_no in (Select service_no
                        From dgrpmanagermsg
                       Where system_type In ('0', '2'))
   and a.op_time >= to_date('2016-07-15 00:00:00','YYYY-MM-DD HH24:MI:SS')  
   and a.op_time < to_date('2016-07-18 00:00:00','YYYY-MM-DD HH24:MI:SS') 
 order by op_time desc

``` 
 
此时的执行计划为：

```sql

Plan hash value: 3129095520

-------------------------------------------------------------------------------------------
| Id  | Operation                     | Name                      | Rows  | Bytes | Cost  |
-------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |                           |     1 |   195 |     8 |
|   1 |  NESTED LOOPS                 |                           |     1 |   195 |     8 |
|*  2 |   TABLE ACCESS BY INDEX ROWID | WGRPLOGINOPR              |     1 |   186 |     7 |
|*  3 |    INDEX RANGE SCAN DESCENDING| IDX_WGRPLOGINOPROPTIME    |     7 |       |     4 |
|*  4 |   TABLE ACCESS BY INDEX ROWID | DGRPMANAGERMSG            |     1 |     9 |     1 |
|*  5 |    INDEX UNIQUE SCAN          | IDX_DGRPMANAGERMSGSERVICE |     1 |       |       |
-------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter("LOGIN_NO" LIKE 'k%')
   3 - access("A"."OP_TIME">=TO_DATE(' 2016-07-15 00:00:00', 'syyyy-mm-dd
              hh24:mi:ss') AND "A"."OP_TIME"<TO_DATE(' 2016-07-18 00:00:00', 'syyyy-mm-dd
              hh24:mi:ss'))
   4 - filter("SYSTEM_TYPE"='0' OR "SYSTEM_TYPE"='2')
   5 - access("A"."LOGIN_NO"="SERVICE_NO")
   
```sql   
   
看执行计划中，已经采用索引扫描了。看来在SQL中只要不要对列操作，ORACLE还是很智能的。

此时的执行时间为0.01秒，逻辑读仅为4。

![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com



