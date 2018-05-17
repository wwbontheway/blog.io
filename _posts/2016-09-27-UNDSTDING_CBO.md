---
layout: post
title: 【优化】通过理解CBO进行优化
date: 2016-09-27
categories: OracleTuning
tags: [Oracle,Tuning]
description: 优化案例。
---

开发人员书写SQL时的思路和构想的执行顺序，往往和oracle真实执行的顺序有所不同，因为oracle会考虑改写sql进而达到最小陈本，不过这种情况有时候会导致sql执行失败

本文中的例子就是一个十多年开发老司机写的SQL，逻辑上完全没问题，但是在数据库中运行就是一直报错，实在费解，最后求助我帮忙看看，那么下面就看一起这个SQL的问题在哪里~

SQL文本如下：

```sql

select *
  from --select substr(end_store_no,11,7),substr(store_no,11,7) from 
        (select /*+ index(a,WSTORERECORD_IDX_TIME) */
          LOGIN_NO,
          LOGIN_ACCEPT,
          TOTAL_DATE,
          STORE_TYPE,
          store_no,
          end_store_no,
          num,
          should_pay,
          to_number(substr(end_store_no, 11, 7)) -
          to_number(substr(store_no, 11, 7)) + 1 should_num
           from Wstorerecord a
          where trim(store_no) is not null
            and length(trim(store_no)) = 17
            and length(trim(end_store_no)) = 17
            and op_time >=
                to_date('20100128 15:50:00', 'yyyymmdd hh24:mi:ss')
            and op_time < to_date('20100128 15:51:00', 'yyyymmdd hh24:mi:ss'))
 where should_num = 1
 
```
 
此SQL文本直观的看上去是没有任何问题的，但是每次运行的时候会报错：

```sql

,to_number(substr(end_store_no,11,7))-to_number(substr(store_no,11,7))+1 should_num
           *
ERROR at line 4:
ORA-01722: invalid number

```

之所以报这个错误呢，是因为这一列存在null值，而且oracle没有按照你写的逻辑先去构建结果集然后过滤这样运行的，而发生了谓词推入，针对Oracle CBO这种“智能”的改写，当时我给了他三种优化调整方法，具体如下：

### 方法1：添加hint让其不进行谓词推入

这里添加的是NO_MERGE，其实也可以添加 NO_QUERY_TRANSFORMATION，这些HINT在我之前的博文都有介绍过~

```sql

select /*+no_merge(t)*/*
  from (select 
          LOGIN_NO,
          LOGIN_ACCEPT,
          TOTAL_DATE,
          STORE_TYPE,
          store_no,
          end_store_no,
          num,
          should_pay,
          to_number(substr(end_store_no, 11, 7)) - to_number(substr(store_no, 11, 7)) + 1  should_num
           from Wstorerecord a
          where trim(store_no) is not null
            and length(trim(store_no)) = 17
            and length(trim(end_store_no)) = 17
            and op_time >= to_date('20100128 15:50:00', 'yyyymmdd hh24:mi:ss')
            and op_time < to_date('20100128 15:51:00', 'yyyymmdd hh24:mi:ss')) t
 where should_num = 1; 

```

### 方法2：将inline视图单独提出来作为一个临时表并添加hint固化

```sql

with t as
 (select /*+ index(a,WSTORERECORD_IDX_TIME) materialize*/
   LOGIN_NO,
   LOGIN_ACCEPT,
   TOTAL_DATE,
   STORE_TYPE,
   store_no,
   end_store_no,
   num,
   should_pay,
   to_number(substr(end_store_no, 11, 7)) -
   to_number(substr(store_no, 11, 7)) + 1 should_num
    from Wstorerecord a
   where trim(store_no) is not null
     and length(trim(store_no)) = 17
     and length(trim(end_store_no)) = 17
     and op_time >= to_date('20100128 15:50:00', 'yyyymmdd hh24:mi:ss')
     and op_time < to_date('20100128 15:51:00', 'yyyymmdd hh24:mi:ss'))
select * from t where should_num = 1;

```

### 第三种：改写SQL

```sql

select *
  from --select substr(end_store_no,11,7),substr(store_no,11,7) from 
        (select /*+ index(a,WSTORERECORD_IDX_TIME) */
          LOGIN_NO,
          LOGIN_ACCEPT,
          TOTAL_DATE,
          STORE_TYPE,
          store_no,
          end_store_no,
          num,
          should_pay,
          to_number(substr(trim(end_store_no), 11, 7)) -
          to_number(substr(trim(store_no), 11, 7)) + 1 should_num
           from Wstorerecord a
          where trim(store_no) is not null
            and length(trim(store_no)) = 17
            and length(trim(end_store_no)) = 17
            and op_time >=
                to_date('20100128 15:50:00', 'yyyymmdd hh24:mi:ss')
            and op_time < to_date('20100128 15:51:00', 'yyyymmdd hh24:mi:ss'))
 where should_num = 1
 
```
 
以上三种方法都可以将sql成功执行。虽然使用了三个方法，但是核心优化思想是一致的。



![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com



