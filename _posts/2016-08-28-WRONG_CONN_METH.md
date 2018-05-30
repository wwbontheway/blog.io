---
layout: post
title: 错误的表连接方式
date: 2016-08-28
categories: OracleTuning
tags: [Oracle,Tuning]
description: 优化案例。
---

数据库里运行的SQL，往往涉及多张表的关联过滤，这其中的顺序一般是ORACLE自己评估执行顺序和方式的，但是ORACLE毕竟是一款软件，某些时候会错误评估，这是就要操作人员手动去介入调整，这里常用的一种手段就是HINT（注意，HINT并不是万能的，但是在偶尔用一次的时候，还是比较有效，比如本案例）。


SQL文本如下：

```sql

select *
  from (select row_.*, rownum rownum_
           from (SELECT A.TITLE,
                         A.SEND_PERSON,
                         B.USER_ID,
                         TO_CHAR(A.CREATE_TIME, 'YYYY-MM-DD HH24:MI:SS'),
                         DECODE((SELECT COUNT(*)
                                  FROM DNOTEATTACHMENT D
                                 WHERE A.ID = D.NOTE_ID),
                                '0',
                                '0',
                                '1') AS ATTACHMENT,
                         A.ID
                    FROM DNOTE A, DRNOTEUSER B
                   WHERE A.DEL_FLAG = '0'
                     and A.ID = B.NOTE_ID
                     AND B.STATE = '3'
                     AND B.USER_ID = '806100'
                     AND A.CREATE_TIME >=
                         TO_DATE('2016-08-11 00:00:00', 'YYYY-MM-DD HH24:MI:SS')
                     AND A.CREATE_TIME <=
                         TO_DATE('2016-08-19 23:59:59', 'YYYY-MM-DD HH24:MI:SS')
                  UNION
                  SELECT A.TITLE,
                         A.SEND_PERSON,
                         B.USER_ID,
                         TO_CHAR(A.CREATE_TIME, 'YYYY-MM-DD HH24:MI:SS'),
                         DECODE((SELECT COUNT(*)
                                  FROM DNOTEATTACHMENT D
                                 WHERE A.ID = D.NOTE_ID),
                                '0',
                                '0',
                                '1') AS ATTACHMENT,
                         A.ID
                    FROM DNOTE A, DRNOTEUSERHISTORY B
                   WHERE A.ID = B.NOTE_ID
                     AND B.DEL_TYPE = 'CLEAN_TRASH_ALL'
                     AND B.USER_ID = '806100'
                     AND A.CREATE_TIME >= TO_DATE('2016-08-11 00:00:00', 'YYYY-MM-DD HH24:MI:SS')
                     AND A.CREATE_TIME <=
                         TO_DATE('2016-08-19 23:59:59', 'YYYY-MM-DD HH24:MI:SS')
                   ORDER BY 4 DESC) row_
          where rownum <= 15)
 where rownum_ >= 1
 
```
 
此SQL的执行计划为:

```sql

Plan hash value: 2688111990

------------------------------------------------------------------------------------------------------------
| Id  | Operation                          | Name                  | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                       |     2 |   542 | 55564   (1)| 00:11:07 |
|*  1 |  VIEW                              |                       |     2 |   542 | 55564   (1)| 00:11:07 |
|*  2 |   COUNT STOPKEY                    |                       |       |       |            |          |
|   3 |    VIEW                            |                       |     2 |   516 | 55564   (1)| 00:11:07 |
|*  4 |     SORT UNIQUE STOPKEY            |                       |     2 |   153 | 55563  (96)| 00:11:07 |
|   5 |      UNION-ALL                     |                       |       |       |            |          |
|   6 |       SORT AGGREGATE               |                       |     1 |     6 |            |          |
|*  7 |        INDEX RANGE SCAN            | IDX_DNOTEATTACHMENT   |     1 |     6 |     1   (0)| 00:00:01 |
|   8 |       NESTED LOOPS                 |                       |       |       |            |          |
|   9 |        NESTED LOOPS                |                       |     1 |    71 |  2569   (1)| 00:00:31 |
|* 10 |         TABLE ACCESS BY INDEX ROWID| DNOTE                 |     1 |    52 |     3   (0)| 00:00:01 |
|* 11 |          INDEX RANGE SCAN          | IDX_DNOTE_1           |     1 |       |     2   (0)| 00:00:01 |
|* 12 |         INDEX RANGE SCAN           | IDX_DRNOTEUSER_USERID |  2712 |       |     8   (0)| 00:00:01 |
|* 13 |        TABLE ACCESS BY INDEX ROWID | DRNOTEUSER            |     1 |    19 |  2566   (1)| 00:00:31 |
|  14 |       SORT AGGREGATE               |                       |     1 |     6 |            |          |
|* 15 |        INDEX RANGE SCAN            | IDX_DNOTEATTACHMENT   |     1 |     6 |     1   (0)| 00:00:01 |
|* 16 |       HASH JOIN                    |                       |     1 |    82 | 52992   (1)| 00:10:36 |
|  17 |        TABLE ACCESS BY INDEX ROWID | DNOTE                 |     1 |    50 |     3   (0)| 00:00:01 |
|* 18 |         INDEX RANGE SCAN           | IDX_DNOTE_1           |     1 |       |     2   (0)| 00:00:01 |
|* 19 |        TABLE ACCESS FULL           | DRNOTEUSERHISTORY     |  2912 | 93184 | 52988   (1)| 00:10:36 |
------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter("ROWNUM_">=1)
   2 - filter(ROWNUM<=15)
   4 - filter(ROWNUM<=15)
   7 - access("D"."NOTE_ID"=:B1)
  10 - filter("A"."DEL_FLAG"='0')
  11 - access("A"."CREATE_TIME">=TO_DATE(' 2016-08-11 00:00:00', 'syyyy-mm-dd hh24:mi:ss') AND
              "A"."CREATE_TIME"<=TO_DATE(' 2016-08-19 23:59:59', 'syyyy-mm-dd hh24:mi:ss'))
  12 - access("B"."USER_ID"='806100')
  13 - filter("B"."STATE"='3' AND "A"."ID"=TO_NUMBER("B"."NOTE_ID"))
  15 - access("D"."NOTE_ID"=:B1)
  16 - access("A"."ID"=TO_NUMBER("B"."NOTE_ID"))
  18 - access("A"."CREATE_TIME">=TO_DATE(' 2016-08-11 00:00:00', 'syyyy-mm-dd hh24:mi:ss') AND
              "A"."CREATE_TIME"<=TO_DATE(' 2016-08-19 23:59:59', 'syyyy-mm-dd hh24:mi:ss'))
  19 - filter("B"."USER_ID"='806100' AND "B"."DEL_TYPE"='CLEAN_TRASH_ALL')
  
```  
  
此SQL运行单次时间将近4分钟，逻辑读为 32058867 ，对于一个分页功能的SQL来说，时间未免太长了。

虽然执行计划很长很唬人，但是我们可以慢慢的一点一点的看，发现最大的问题，然后改正它就好。

从执行计划可以看到，这个SQL可能出现问题的地方出现在UNION连接符之前的两张表A和B。在这里，ORACLE的CBO明显估计错了过滤之后行源的行数，于是选择了一种错误的表连接方式。并且由于这种连接方式导致过滤之后的回表消耗也是很大的，为了帮助ORACLE进行正确的连接方式，这里采用一种常用的优化手段：HINT。（为什么我不收集最新的统计信息，因为在生产环境，有些初期没有规划好的表，落掉了很久的统计信息，这时候收集的话，可能会引起性能问题，一是收集的时候占用一定的资源，二是可能导致其他的SQL执行计划改变，而选择更差的执行计划。这里就是这个情况，客户坚决不可以收集统计信息，没办法，只能用HINT来弄）

原SQL添加完HINT之后的SQL文本为：

```sql

select *
  from (select row_.*, rownum rownum_
           from (SELECT /*+use_hash(a,b)*/A.TITLE,
                         A.SEND_PERSON,
                         B.USER_ID,
                         TO_CHAR(A.CREATE_TIME, 'YYYY-MM-DD HH24:MI:SS'),
                         DECODE((SELECT COUNT(*)
                                  FROM DNOTEATTACHMENT D
                                 WHERE A.ID = D.NOTE_ID),
                                '0',
                                '0',
                                '1') AS ATTACHMENT,
                         A.ID
                    FROM DNOTE A, DRNOTEUSER B
                   WHERE A.DEL_FLAG = '0'
                     and A.ID = B.NOTE_ID
                     AND B.STATE = '3'
                     AND B.USER_ID = '806100'
                     AND A.CREATE_TIME >=
                         TO_DATE('2016-08-11 00:00:00', 'YYYY-MM-DD HH24:MI:SS')
                     AND A.CREATE_TIME <=
                         TO_DATE('2016-08-19 23:59:59', 'YYYY-MM-DD HH24:MI:SS')
                  UNION
                  SELECT A.TITLE,
                         A.SEND_PERSON,
                         B.USER_ID,
                         TO_CHAR(A.CREATE_TIME, 'YYYY-MM-DD HH24:MI:SS'),
                         DECODE((SELECT COUNT(*)
                                  FROM DNOTEATTACHMENT D
                                 WHERE A.ID = D.NOTE_ID),
                                '0',
                                '0',
                                '1') AS ATTACHMENT,
                         A.ID
                    FROM DNOTE A, DRNOTEUSERHISTORY B
                   WHERE A.ID = B.NOTE_ID
                     AND B.DEL_TYPE = 'CLEAN_TRASH_ALL'
                     AND B.USER_ID = '806100'
                     AND A.CREATE_TIME >= TO_DATE('2016-08-11 00:00:00', 'YYYY-MM-DD HH24:MI:SS')
                     AND A.CREATE_TIME <=
                         TO_DATE('2016-08-19 23:59:59', 'YYYY-MM-DD HH24:MI:SS')
                   ORDER BY 4 DESC) row_
          where rownum <= 15)
 where rownum_ >= 1

``` 
 
此时再次运行此SQL，执行计划如下：

```sql

Plan hash value: 2498963761

-----------------------------------------------------------------------------------------------------------
| Id  | Operation                         | Name                  | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                  |                       |     2 |   542 | 55565   (1)| 00:11:07 |
|*  1 |  VIEW                             |                       |     2 |   542 | 55565   (1)| 00:11:07 |
|*  2 |   COUNT STOPKEY                   |                       |       |       |            |          |
|   3 |    VIEW                           |                       |     2 |   516 | 55565   (1)| 00:11:07 |
|*  4 |     SORT UNIQUE STOPKEY           |                       |     2 |   153 | 55564  (96)| 00:11:07 |
|   5 |      UNION-ALL                    |                       |       |       |            |          |
|   6 |       SORT AGGREGATE              |                       |     1 |     6 |            |          |
|*  7 |        INDEX RANGE SCAN           | IDX_DNOTEATTACHMENT   |     1 |     6 |     1   (0)| 00:00:01 |
|*  8 |       HASH JOIN                   |                       |     1 |    71 |  2571   (1)| 00:00:31 |
|*  9 |        TABLE ACCESS BY INDEX ROWID| DNOTE                 |     1 |    52 |     3   (0)| 00:00:01 |
|* 10 |         INDEX RANGE SCAN          | IDX_DNOTE_1           |     1 |       |     2   (0)| 00:00:01 |
|* 11 |        TABLE ACCESS BY INDEX ROWID| DRNOTEUSER            |  2350 | 44650 |  2567   (1)| 00:00:31 |
|* 12 |         INDEX RANGE SCAN          | IDX_DRNOTEUSER_USERID |  2712 |       |     9   (0)| 00:00:01 |
|  13 |       SORT AGGREGATE              |                       |     1 |     6 |            |          |
|* 14 |        INDEX RANGE SCAN           | IDX_DNOTEATTACHMENT   |     1 |     6 |     1   (0)| 00:00:01 |
|* 15 |       HASH JOIN                   |                       |     1 |    82 | 52992   (1)| 00:10:36 |
|  16 |        TABLE ACCESS BY INDEX ROWID| DNOTE                 |     1 |    50 |     3   (0)| 00:00:01 |
|* 17 |         INDEX RANGE SCAN          | IDX_DNOTE_1           |     1 |       |     2   (0)| 00:00:01 |
|* 18 |        TABLE ACCESS FULL          | DRNOTEUSERHISTORY     |  2912 | 93184 | 52988   (1)| 00:10:36 |
-----------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter("ROWNUM_">=1)
   2 - filter(ROWNUM<=15)
   4 - filter(ROWNUM<=15)
   7 - access("D"."NOTE_ID"=:B1)
   8 - access("A"."ID"=TO_NUMBER("B"."NOTE_ID"))
   9 - filter("A"."DEL_FLAG"='0')
  10 - access("A"."CREATE_TIME">=TO_DATE(' 2016-08-11 00:00:00', 'syyyy-mm-dd hh24:mi:ss') AND
              "A"."CREATE_TIME"<=TO_DATE(' 2016-08-19 23:59:59', 'syyyy-mm-dd hh24:mi:ss'))
  11 - filter("B"."STATE"='3')
  12 - access("B"."USER_ID"='806100')
  14 - access("D"."NOTE_ID"=:B1)
  15 - access("A"."ID"=TO_NUMBER("B"."NOTE_ID"))
  17 - access("A"."CREATE_TIME">=TO_DATE(' 2016-08-11 00:00:00', 'syyyy-mm-dd hh24:mi:ss') AND
              "A"."CREATE_TIME"<=TO_DATE(' 2016-08-19 23:59:59', 'syyyy-mm-dd hh24:mi:ss'))
  18 - filter("B"."USER_ID"='806100' AND "B"."DEL_TYPE"='CLEAN_TRASH_ALL')

```

由于更换了新的连接方式，执行时间从之前的4分降到7秒，逻辑读为249412，相比之前的32058867来说，降低了百倍有余。至此，这条SQL的优化就暂时完成了，对数据库资源的消耗也降低了很多。


![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com



