---
layout: post
title:  MySQL的timestamp数据类型测试
date: 2018-08-07
categories: MySQL
tags: [MySQL]
description: MySQL的timestamp数据类型测试
---

今天开发同事咨询我一个关于timestamp的问题，给他处理完，正好顺便测试关于timestamp格式在不同sql_mode下的区别。

当前MySQL版本和sql_mode：

```sql
root@localhost [wwbtest]>show variables like 'sql_mode' \G
*************************** 1. row ***************************
Variable_name: sql_mode
        Value: ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
1 row in set (0.00 sec)
```

首先改变sql_mode的格式，去掉STRICT_TRANS_TABLES、NO_ZERO_IN_DATE、NO_ZERO_DATE这三个：

```sql
root@localhost [wwbtest]>set session sql_mode='ONLY_FULL_GROUP_BY,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';

Query OK, 0 rows affected, 1 warning (0.01 sec)
```

warning不用管他，主要是提示这几个应该用于严格模式（strict mode）

如果不去掉，建立空日期会报下面的错误（主要是由于当前默认值会给0值，但是NO_ZERO_IN_DATE、NO_ZERO_DATE这两个根据字面意思也知道是不可以在日期中给0值的）：

```sql
root@localhost [wwbtest]>create table wtab1(c1 timestamp,c2 timestamp,c3 timestamp);  ERROR 1067 (42000): Invalid default value for 'c2'
```



这三个的含义是：

- STRICT_TRANS_TABLES：strict mode（严格模式），在该模式下，如果一个值不能插入到一个事务表中，则中断当前的操作，对非事务表不做任何限制。即是说，在这个模式下，MySQL会严格的进行数据的校验，当发现插入列值未满足要求，直接报告error错误，保证了错误数据无法插入到数据库中。

- NO_ZERO_IN_DATE：在严格模式，不接受月或日部分为0的日期。如果使用IGNORE选项，我们为类似的日期插入'0000-00-00'。在非严格模式，可以接受该日期，但会报warning。

- NO_ZERO_DATE：在严格模式，不要将 '0000-00-00'做为合法日期。你仍然可以用IGNORE选项插入零日期。在非严格模式，可以接受该日期，但会报warning 。

目前explicit_defults_for_timestamp参数是关闭的。

```sql
root@localhost [wwbtest]>show variables like 'explicit%';

+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| explicit_defaults_for_timestamp | OFF   |
+---------------------------------+-------+
1 row in set (0.00 sec)
```



创建测试1表：


```sql
root@localhost [wwbtest]>create table wtab1(c1 timestamp,c2 timestamp,c3 timestamp);  Query OK, 0 rows affected (0.18 sec)

root@localhost [wwbtest]>show create table wtab1\G
*************************** 1. row ***************************
       Table: wtab1
Create Table: CREATE TABLE `wtab1` (
  `c1` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `c2` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `c3` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00'
) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row in set (0.00 sec)
```

如上，三个timestamp类型的字段都默认被自动设置了not null 属性，即使我没有在建表语句中显式的指定。并且第一个字段的默认值与后两个默认值不同。第一个字段默认值伟当前时间戳，后两个默认值为0000-00-00 00:00:00，这也就是为什么在前面的sql_mode中要将NO_ZERO_IN_DATE、NO_ZERO_DATE这两个去掉的原因。

那么现在向表中插入数据，观察一下：

```sql

root@localhost [wwbtest]>insert into wtab1 values (null,null,null);
Query OK, 1 row affected (0.12 sec)

root@localhost [wwbtest]>select * from wtab1;
+---------------------+---------------------+---------------------+
| c1                  | c2                  | c3                  |
+---------------------+---------------------+---------------------+
| 2018-08-07 11:47:19 | 2018-08-07 11:47:19 | 2018-08-07 11:47:19 |
+---------------------+---------------------+---------------------+
1 row in set (0.00 sec)

root@localhost [wwbtest]>insert into wtab1(c1) values (null);
Query OK, 1 row affected (0.00 sec)

root@localhost [wwbtest]>select * from wtab1;
+---------------------+---------------------+---------------------+
| c1                  | c2                  | c3                  |
+---------------------+---------------------+---------------------+
| 2018-08-07 11:47:19 | 2018-08-07 11:47:19 | 2018-08-07 11:47:19 |
| 2018-08-07 11:48:06 | 0000-00-00 00:00:00 | 0000-00-00 00:00:00 |
+---------------------+---------------------+---------------------+
2 rows in set (0.00 sec)

root@localhost [wwbtest]>insert into wtab1(c2) values (null);
Query OK, 1 row affected (0.00 sec)

root@localhost [wwbtest]>select * from wtab1;
+---------------------+---------------------+---------------------+
| c1                  | c2                  | c3                  |
+---------------------+---------------------+---------------------+
| 2018-08-07 11:47:19 | 2018-08-07 11:47:19 | 2018-08-07 11:47:19 |
| 2018-08-07 11:48:06 | 0000-00-00 00:00:00 | 0000-00-00 00:00:00 |
| 2018-08-07 11:48:23 | 2018-08-07 11:48:23 | 0000-00-00 00:00:00 |
+---------------------+---------------------+---------------------+
3 rows in set (0.00 sec)

root@localhost [wwbtest]>insert into wtab1(c3) values (null);
Query OK, 1 row affected (0.00 sec)

root@localhost [wwbtest]>select * from wtab1;
+---------------------+---------------------+---------------------+
| c1                  | c2                  | c3                  |
+---------------------+---------------------+---------------------+
| 2018-08-07 11:47:19 | 2018-08-07 11:47:19 | 2018-08-07 11:47:19 |
| 2018-08-07 11:48:06 | 0000-00-00 00:00:00 | 0000-00-00 00:00:00 |
| 2018-08-07 11:48:23 | 2018-08-07 11:48:23 | 0000-00-00 00:00:00 |
| 2018-08-07 11:48:40 | 0000-00-00 00:00:00 | 2018-08-07 11:48:40 |
+---------------------+---------------------+---------------------+
4 rows in set (0.00 sec)
```



如上，当给timestamp列插入null值得时候，会默认更新为当前时间戳。而不指定插入值得时候，会根据他得default值进行插入。



下面将explicit_defults_for_timestamp参数打开

```sql
root@localhost [wwbtest]>set session explicit_defaults_for_timestamp=1;
Query OK, 0 rows affected (0.00 sec)

root@localhost [wwbtest]>show variables like 'explicit_defaults_for_timestamp';
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| explicit_defaults_for_timestamp | ON    |
+---------------------------------+-------+
1 row in set (0.00 sec)
```



建立测试2表：

```sql
root@localhost [wwbtest]>create table wtab2 (c1 timestamp,c2 timestamp,c3 timestamp);
Query OK, 0 rows affected (0.42 sec)

root@localhost [wwbtest]>show create table wtab2 \G
*************************** 1. row ***************************
       Table: wtab2
Create Table: CREATE TABLE `wtab2` (
  `c1` timestamp NULL DEFAULT NULL,
  `c2` timestamp NULL DEFAULT NULL,
  `c3` timestamp NULL DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row in set (0.00 sec)
```

在启用explicit_defaults_for_timestamp参数之后，建立测试表的时候没有指定null属性和默认值，所以表中这三个字段可以插入null值，而默认值设置为null。



测试插入数据：

```sql
root@localhost [wwbtest]>insert into wtab2(c1) values(null);

Query OK, 1 row affected (0.00 sec)

root@localhost [wwbtest]>select * from wtab2;
+------+------+------+
| c1   | c2   | c3   |
+------+------+------+
| NULL | NULL | NULL |
+------+------+------+
1 row in set (0.00 sec)

root@localhost [wwbtest]>insert into wtab2(c2) values(null);
Query OK, 1 row affected (0.00 sec)

root@localhost [wwbtest]>select * from wtab2;
+------+------+------+
| c1   | c2   | c3   |
+------+------+------+
| NULL | NULL | NULL |
| NULL | NULL | NULL |
+------+------+------+
2 rows in set (0.00 sec)

root@localhost [wwbtest]>insert into wtab2(c3) values(null);
Query OK, 1 row affected (0.00 sec)

root@localhost [wwbtest]>select * from wtab2;
+------+------+------+
| c1   | c2   | c3   |
+------+------+------+
| NULL | NULL | NULL |
| NULL | NULL | NULL |
| NULL | NULL | NULL |
+------+------+------+
3 rows in set (0.00 sec)
```

如上，在不给定值得时候，默认插入null。

下面将sql_mode调整回严格模式，看一下如果不符合严格模式得要求，会怎样。



创建测试表3，并插入数据：

```sql
root@localhost [wwbtest]>select @@sql_mode \G
*************************** 1. row ***************************
@@sql_mode: ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
1 row in set (0.00 sec)

root@localhost [wwbtest]>show variables like 'explicit%';
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| explicit_defaults_for_timestamp | ON    |
+---------------------------------+-------+
1 row in set (0.00 sec)

root@localhost [wwbtest]>create table wtab3 (c1 timestamp,c2 timestamp not null);
Query OK, 0 rows affected (0.01 sec)

root@localhost [wwbtest]>show create table wtab3 \G
*************************** 1. row ***************************
       Table: wtab3
Create Table: CREATE TABLE `wtab3` (
  `c1` timestamp NULL DEFAULT NULL,
  `c2` timestamp NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row in set (0.00 sec)

root@localhost [wwbtest]>insert into wtab3(c1) values (null);
ERROR 1364 (HY000): Field 'c2' doesn't have a default value
```

如上，当sql_mode中有STRICT_TRANS_TABLES的时候，操作不满足要求会直接报错。所以为了生产环境中的数据严谨，还是建议使用strict mode。



![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com




