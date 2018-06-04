---
layout: post
title: SQL中截取指定的字符串
date: 2018-06-04
categories: OracleSQL
tags: [Oracle,SQL]
description: SQL中截取指定的字符串
---

有一个朋友咨询我一个问题，想截取指定结构体中夹着的一个字符串，而且这个字符串的长度不固定。这就用到了Oracle SQL中的三个函数：instr、substr和length

### INSTR

查找字符串位置。语法图如下：

![instr](https://docs.oracle.com/cd/E11882_01/server.112/e41084/img/instr.gif)



上图中只需要关注instr即可，其他的这里不做介绍，还是老话，感兴趣的可以自行google了解～

instr后面的参数：

- string：表示元字符串
- substring：表示要查找的子字符串
- position：表示要查找的开始位置，为可选项（默认为1），注意在这里字符串索引从1开始，如果此参数为正，则从左到右检索，如果此参数为负，则从右到左检索
- occurrence：表示元字符串中第几次出现的子字符串，此参数可选，缺省默认为1，如果是负数则系统报错。



### SUBSTR

截取字符串，语法图如下：

![substr](https://docs.oracle.com/cd/E11882_01/server.112/e41084/img/substr.gif)

还是那句话，只需要关注substr即可。

substr后面的参数：

- char是元字符串
- position为开始位置
- substring_length是可选项，表示子字符串的位数。



### LENGTH

计算字符长度，单位是字符，语法图如下：

![length](https://docs.oracle.com/cd/E11882_01/server.112/e41084/img/length.gif)

也是一样，只需要关注length就行（因为本博文只涉及到这个，其他的感兴趣可以自己深入研究一下），其中参数char表示需要计算长度的元字符串。



## 示例

了解到上面的几个函数的用法之后，来看详细一下我这位朋友的需求。

需要截取一串xml代码结构题中被QueryNum标识符扩起来的一串字符串，为了演示方便，我寻求了2条数据，构建了一个简单的临时行源：

```sql

SQL> create table t2(a1 varchar2(4000));

Table created.


SQL> insert into t2 values ('<?xml version="1.0" encoding="UTF-8"?><Query><QueryNum>n0451zhy171244</QueryNum><Mac>null</Mac><NumType>2</NumType><FaultModel>2</FaultModel><FaultType>1001</FaultType><Area>451</Area><DeviceId>1000000004</DeviceId><MeStartTime></MeStartTime><MeRealTime></MeRealTime><MeEndTime></MeEndTime></Query>');

1 row created.

SQL> insert into t2 values ('<?xml version="1.0" encoding="UTF-8"?><Query><QueryNum>n0451wjf1217051</QueryNum><Mac>null</Mac><NumType>2</NumType><FaultModel>2</FaultModel><FaultType>1001</FaultType><Area>451</Area><DeviceId>1000000004</DeviceId><MeStartTime></MeStartTime><MeRealTime></MeRealTime><MeEndTime></MeEndTime></Query>');

1 row created.

SQL> commit;

Commit complete.

SQL> select * from t2;

A1
-----------------------------------------------------------------------
<?xml version="1.0" encoding="UTF-8"?><Query><QueryNum>n0451zhy171244</QueryNum>
<Mac>null</Mac><NumType>2</NumType><FaultModel>2</FaultModel><FaultType>1001</Fa
ultType><Area>451</Area><DeviceId>1000000004</DeviceId><MeStartTime></MeStartTim
e><MeRealTime></MeRealTime><MeEndTime></MeEndTime></Query>

<?xml version="1.0" encoding="UTF-8"?><Query><QueryNum>n0451wjf1217051</QueryNum
><Mac>null</Mac><NumType>2</NumType><FaultModel>2</FaultModel><FaultType>1001</F
aultType><Area>451</Area><DeviceId>1000000004</DeviceId><MeStartTime></MeStartTi
me><MeRealTime></MeRealTime><MeEndTime></MeEndTime></Query>
```

如上，截取字段，需要用到substr函数，需要知道的参数是起始位置和长度。这两个参数可以通过instr和length来得到：


```sql

SQL> select instr(t2.a1,'<QueryNum>') from t2;

INSTR(T2.A1,'<QUERYNUM>')
-------------------------
                       46
                       46

SQL> select instr(t2.a1,'</QueryNum>') from t2;

INSTR(T2.A1,'</QUERYNUM>')
--------------------------
                        70
                        71

SQL> select length('<QueryNum>') from dual;

LENGTH('<QUERYNUM>')
--------------------
                  10
```

通过上面的查询，我们就知道了开始位置应该是\<QueryNum\>的位置加上它的长度，截取的长度就是\</QueryNum\>的位置减去开始位置，那么这条截取的SQL就可以很简单的写出来了：

```sql
SQL> select * from t2;

A1
-----------------------------------------------------------------------
<?xml version="1.0" encoding="UTF-8"?><Query><QueryNum>n0451zhy171244</QueryNum>
<Mac>null</Mac><NumType>2</NumType><FaultModel>2</FaultModel><FaultType>1001</Fa
ultType><Area>451</Area><DeviceId>1000000004</DeviceId><MeStartTime></MeStartTim
e><MeRealTime></MeRealTime><MeEndTime></MeEndTime></Query>

<?xml version="1.0" encoding="UTF-8"?><Query><QueryNum>n0451wjf1217051</QueryNum
><Mac>null</Mac><NumType>2</NumType><FaultModel>2</FaultModel><FaultType>1001</F
aultType><Area>451</Area><DeviceId>1000000004</DeviceId><MeStartTime></MeStartTi
me><MeRealTime></MeRealTime><MeEndTime></MeEndTime></Query>


SQL> select substr(t2.a1,instr(t2.a1,'<QueryNum>')+10,instr(t2.a1,'</QueryNum>')-instr(t2.a1,'<QueryNum>')-10) c1 from t2;

C1
----------------------------------------------------------------------
n0451zhy171244
n0451wjf1217051
```




![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com




