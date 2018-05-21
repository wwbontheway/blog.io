---
layout: post
title: 都是BITMAP惹的祸
date: 2017-06-07
categories: OracleDiagnosis
tags: [Oracle,Diagnosis]
description: 高频dml环境下的bitmap索引危害
---


分享一次故障诊断~

早上客户反应给我说前一天晚上的跑处理出现异常，请求支援。

远程登录之后，发现全部等待都卡在应用日志表ETL_LOAD_HIS_LOG_DETAIL表上。导致整体跑批全部阻塞。

查当时问题时段的AWR信息，发现大量的“enq:TX - row lock contention”，平均延时高达2w7 ms，马上根据这个异常等待时间，查询对应的sql_id：

```sql

select sql_id,count(0) from dba_hist_active_sess_history
where snap_id between 30336 and 30340
and event ='enq:TX - row lock contention'
group by sql_id

SQL_ID         COUNT(0)
------------- ---------
55q3yr6s7tj7v  14042

--其他输出略，因为其他sql的count数基本在20以下

```

抓到了这个sql：55q3yr6s7tj7v

```sql

SQL> select sql_text from v$sql where sql_id='55q3yr6s7tj7v';

SQL_TEXT
--------------------------------------------------------------------------------
INSERT INTO ETL_LOAD_HIS_LOG_DETAIL (JOBTYPE, JOBNAME, TX_DATE, LOAD_TIME, DEAL_
TYPE, START_DATE, END_DATE, RUN_TIMES, DEAL_ROW, SQL_CODE, SQL_STATE) VALUES (UP
PER(:B11 ), UPPER(:B10 ), :B9 , :B8 , :B7 , :B6 , :B5 , :B4 , :B3 , :B2 , :B1 )

```

由上可见大量的行级锁都集中在对应用日志表ETL_LOAD_HIS_LOG_DETAIL的插入动作上。
查询这个等待的对象：

```sql

select t.object_name,t.object_type,ash.event,count(0) 
from v$active_session_history ash,dba_objects t 
where event='enq: TX - row lock contention' 
and ash.current_obj#=t.object_id 
group by t.object_name,t.object_type,ash.event 
order by 4 desc;

OBJECT_NAME                   OBJECT_TYPE  EVENT                         COUNT(0)
----------------------------- -----------  ----------------------------- ---------
IDX_TEL_LOAD_HIS_LOG_DETAIL01 INDEX        enq:TX - row lock contention  71198
IDX_TEL_LOAD_HIS_LOG_DETAIL02 INDEX        enq:TX - row lock contention  1

```

可知阻塞主要发生在这个索引IDX_TEL_LOAD_HIS_LOG_DETAIL01上。


```sql

SQL> select b.INDEX_OWNER,
  2         b.INDEX_NAME,a.index_type,
  3         a.tablespace_name,
  4         b.COLUMN_NAME,
  5         b.COLUMN_POSITION,
  6         a.LAST_ANALYZED,a.status 
  7    from dba_indexes a, dba_ind_columns b 
  8   where a.table_name = b.table_name 
  9     and a.index_name = b.index_name 
 10     and a.owner = b.index_owner 
 11     and b.table_name = upper('&table_name') 
 12   order by index_owner, index_name, column_position; 
Enter value for table_name: ETL_LOAD_HIS_LOG_DETAIL

INDEX_OWNER  INDEX_NAME                     INDEX_TYPE            TABLESPACE_NAME COLUMN_NAME  COLUMN_POSITION LAST_ANALYZE STATUS
------------ ------------------------------ --------------------- --------------- ------------ --------------- ------------ --------
BISTAT       IDX_ETL_LOAD_HIS_LOG_DETAIL01  BITMAP                DW_TB           TX_DATE                    1 06-JUN-17    VALID
BISTAT       IDX_ETL_LOAD_HIS_LOG_DETAIL02  FUNCTION-BASED BITMAP DW_TB           SYS_NC00012$               1 06-JUN-17    VALID

```

注意看，这个表上的两个索引都是位图索引，这种索引在这种dml操作很频繁的表上是很不适合的。

由于之前没有出现过这种问题，于是严重怀疑这俩个bitmap索引是近期建立的：

```sql

SQL> alter session set nls_date_format='yyyy/mm/dd hh24:mi:ss';

Session altered.

SQL> select created,last_ddl_time 
from dba_objects t 
where t.OBJECT_NAME in('IDX_ETL_LOAD_HIS_LOG_DETAIL01','IDX_ETL_LOAD_HIS_LOG_DETAIL02');

CREATED             LAST_DDL_TIME
------------------- -------------------
2017/06/06 15:03:34 2017/06/07 10:00:50
2017/06/06 15:03:31 2017/06/07 10:00:50

```

确实是事发的前一天（2017年6月6日）下午3点才建立的索引，而正在此时，数据库中仍然存在大量的等待：

```sql

SQL> select sql_id,inst_id,username,event,status from gv$session where event not like '%SQL%' and event not like '%message%' order by 2,5;

SQL_ID           INST_ID USERNAME    EVENT                                                     STATUS
------------- ---------- ----------- --------------------------------------------------------- --------
                       1             Streams AQ: emn coordinator idle wait                     ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
0s3xb8sbqut83          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
                       1             pmon timer                                                ACTIVE
2qbdnkn7qb0z5          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
                       1             VKTM Logical Idle Wait                                    ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
                       1             DIAG idle wait                                            ACTIVE
fa204258jd3t5          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
0kkhhb2w93cx0          1             Space Manager: slave idle wait                            ACTIVE
                       1             PING                                                      ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
                       1             Streams AQ: qmn coordinator idle wait                     ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
                       1             DIAG idle wait                                            ACTIVE
                       1             Streams AQ: qmn slave idle wait                           ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
                       1             class slave wait                                          ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
gp4sp0mcwxrp6          1 SYS         PX Deq: Execute Reply                                     ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
0kkhhb2w93cx0          1             Space Manager: slave idle wait                            ACTIVE
389aptznpu4u8          1 BISTAT      cell multiblock physical read                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
                       1             Streams AQ: waiting for time management or cleanup tasks  ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
                       1             Space Manager: slave idle wait                            ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
                       1             GCR sleep                                                 ACTIVE
                       1             class slave wait                                          ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
2t28667gmj7c3          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
                       1             EMON slave idle wait                                      ACTIVE
                       1             EMON slave idle wait                                      ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
                       1             EMON slave idle wait                                      ACTIVE
                       1             Space Manager: slave idle wait                            ACTIVE
                       1             EMON slave idle wait                                      ACTIVE
                       1             smon timer                                                ACTIVE
                       1             EMON slave idle wait                                      ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
gp4sp0mcwxrp6          1 SYS         PX Deq: Execution Msg                                     ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
                       1             ASM background timer                                      ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
                       1             Streams AQ: qmn slave idle wait                           ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          1 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
8q71s38ukdcb7          2 BISTAT      PX Deq Credit: send blkd                                  ACTIVE
8q71s38ukdcb7          2 BISTAT      cell single block physical read                           ACTIVE
                       2             pmon timer                                                ACTIVE
8q71s38ukdcb7          2 BISTAT      PX Deq Credit: send blkd                                  ACTIVE
                       2             EMON slave idle wait                                      ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
                       2             ASM background timer                                      ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
                       2             VKTM Logical Idle Wait                                    ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
                       2             DIAG idle wait                                            ACTIVE
8q71s38ukdcb7          2 BISTAT      PX Deq Credit: send blkd                                  ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
gp4sp0mcwxrp6          2 SYS         PX Deq: Execution Msg                                     ACTIVE
                       2             PING                                                      ACTIVE
                       2             Space Manager: slave idle wait                            ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
                       2             class slave wait                                          ACTIVE
59mwn9rdjmjw4          2 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
                       2             DIAG idle wait                                            ACTIVE
                       2             Streams AQ: qmn coordinator idle wait                     ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
                       2             Streams AQ: waiting for time management or cleanup tasks  ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
                       2             Streams AQ: qmn slave idle wait                           ACTIVE
                       2             Space Manager: slave idle wait                            ACTIVE
8q71s38ukdcb7          2 BISTAT      PX Deq Credit: send blkd                                  ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
                       2             GCR sleep                                                 ACTIVE
                       2             Streams AQ: emn coordinator idle wait                     ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE
                       2             EMON slave idle wait                                      ACTIVE
4tt2gjx0vqvsy          2 BISTAT      cell multiblock physical read                             ACTIVE
55q3yr6s7tj7v          2 BISTAT      enq: TX - row lock contention                             ACTIVE

--其余输出略

```

由于现在还在阻塞而导致批处理无法进行，紧急处理方案就是先将行锁回话kill掉，然后将索引禁用之后，在重新由客户开发人员调用程序。与客户沟通之后，执行方案，之后恢复正常。

```sql

SQL> alter index BISTAT.IDX_ETL_LOAD_HIS_LOG_DETAIL02 unusable;
alter index BISTAT.IDX_ETL_LOAD_HIS_LOG_DETAIL01 unusable;

Index altered.

SQL> 
Index altered.


SQL>  select b.INDEX_OWNER,                               
  2          b.INDEX_NAME,a.index_type,                   
  3          a.tablespace_name,                           
  4          b.COLUMN_NAME,                               
  5          b.COLUMN_POSITION,                           
  6          a.LAST_ANALYZED,a.status                     
  7     from dba_indexes a, dba_ind_columns b             
  8    where a.table_name = b.table_name                  
  9      and a.index_name = b.index_name                  
 10      and a.owner = b.index_owner                      
 11      and b.table_name = upper('ETL_LOAD_HIS_LOG_DETAIL')          
 12    order by index_owner, index_name, column_position; 


INDEX_OWNER  INDEX_NAME                     INDEX_TYPE             TABLESPACE_NAME  COLUMN_NAME  COLUMN_POSITION LAST_ANALYZE STATUS
------------ ------------------------------ ---------------------- ---------------- ------------ --------------- ------------ --------
BISTAT       IDX_ETL_LOAD_HIS_LOG_DETAIL01  BITMAP                 DW_TB            TX_DATE                    1 06-JUN-17    UNUSABLE
BISTAT       IDX_ETL_LOAD_HIS_LOG_DETAIL02  FUNCTION-BASED BITMAP  DW_TB            SYS_NC00012$               1 06-JUN-17    UNUSABLE

```


至此，这个故障就算处理完了。


**特别注意**

生产环境一定不要给开发DBA权限（除非开发团队有对数据库很了解的人），这个案例中开发就是拥有DBA权限，而在建立索引的时候，由于开发人员对Oracle数据库的理解可能不够深刻，导致建立索引的时候，在这种频繁DML操作的表上建立了BITMAP索引，而导致了这个事故的发生。回头想一想，这种事情其实是完全可以避免的~





![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com




