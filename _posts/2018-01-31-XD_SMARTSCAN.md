---
layout: post
title: Exadata的特性Smart Scan引起的问题
date: 2018-01-31
categories: OracleDiagnosis
tags: [Oracle,Diagnosis]
description: Smart Scan引起的问题
---

今天分享一个Oracle Exadata的案例，在分享之前，我先大概介绍一下什么叫做Smart Scan。

## Smart Scan （智能扫描）

Oracle Exadata一体机之所以性能非常流弊，并不是它的硬件多么的好，而是它独有的管理软件的众多特性，此篇博文只介绍涉及到的其中一个特性\-Smart Scan。

Smart Scan，也就是智能扫描。它是OFFLOAD的一种，包含行过滤、列映像、存储索引、布隆过滤（bloom filters）。给数据库返回的是数据的结果集，不是数据块，传统的数据库是返还的是数据块。
Smart Scan给数据库返回的并非如传统数据库一样，是数据块，而是数据的结果集。可以简单的理解就是在存储节点（cell）把数据根据SQL中的谓词条件提前过滤掉了。

- 行过滤：只把满足条件的where条件部分的部分行记录传输给数据库服务器进行后期数据处理。利用“谓词过滤“

- 列映射：只会把SQL语句中的select和from之间的列的数据返还到数据库服务器进行后期数据处理。

可以将参数cell_offload_processing设置为false则关闭Smart Scan特性。

### Smart Scan的先决条件

1. 必须是全表扫描、索引快速全扫描，即必须为多块读
2. 必须直接路径读direct path read。11g之前并行查询就是直路径读，11g之后，串行的执行的sql也可能也采用直路径读。
3. 数据存在exadata上

**需要注意的是，如果是cluster表、IOT表等情况即是满足条件也是不会进行Smart Scan的**

另外Exadata中的SQL的执行计划中就算出现了table storage full scan，也不表示一定发生了Smart Scan。具体可以查看10046，查看等待事件是否为cell smart table scan，或者使用sql monitor等工具来确定。

对于direct path read这里多说几句，传统的方式读取数据的顺序是：disk \> buffer cache \> pga，而直接路径读则是直接从磁盘读到pga，即：disk \> pga。
 
这种读取方式是受三个隐藏参数影响的

1. \_small\_table\_threshold：单位是数据块个数。当表或索引块超过这个参数值，Oracle将考虑使用direct path read代替buffer read，这个隐藏参数用来界定大表和小表。

2. \_very\_large\_object\_threshold：单位是百分比，默认是500，即是500倍。当表大于500倍的buffer cache，Oracle就认为这个表示超级大表。

3. \_direct\_read\_decision\_statistics\_driven：来决定从哪方面获取数据块的数量。默认是true，表示判断是否要进行direct path read的时候是从表的统计信息来决定的，而非表的实际数据块数量。如果关闭此参数，则会由表的段头信息中的实际数据块数量来决定。

那么经过上面简单的介绍，应该对Smart Scan有了个初步的了解，在深入的介绍这篇博文就不写了，感兴趣的可以自行google或者继续关注我未来会写一个专题（有可能吧，完全看心情，嘿嘿~~）


那么下面就说一下相关的一个案例。

## 案例分享

由于涉及到的客户是一个财险的企业，所以具体数据信息这里考虑到保密的原因，就不展示了，只大概表述一下表象和思路。（我才不会告诉你们，我是懒得贴图片呢~）

#### 一、现象

客户夜间批处理程序中间数据加工层运行的时间相比平时延时了3个多小时。由于这层中的这个存储过程处于批处理程序的关键路径上，最终导致了整个批处理的延时。


#### 二、思路&过程

1. **首先根据客户反应的信息，定位到具体哪个存储过程，以及它运行的时间周期。**

2. **在存储过程中定位到异常的SQL，可以通过对比正常时段的情况进行排查。**

3. **最终定位了一个SQL，如下格式：**

```sql
INSERT /*+APPEND*/
INTO TAB_MID_TEMP (COL1,COL2,COL3,...)
SELECT COL1,COL2,FUNC1(COL3),...
FROM
(SELECT COLA,SUM(COLB) FROM TAB_MID WHERE PRED1 <= :B1 GROUP BY COLA)
```

其中涉及表如下：
TAB_MID_TEMP为实体表，作为中间表用处，每次存储过程最后进行truncate。
TAB_MID为实体表，非分区表，数据量大概在420w左右，SEGMENT SIZE为7G多，高水位，并且DML非常频繁。

FUNC1是一个自定义的函数，结构如下。

```sql
SQL> select text from dba_source where name ='FN_DEFINITE_REGION' order by line;

TEXT
------------------------------------------------------------------------------
FUNCTION FN_DEFINITE_REGION
(
    P_IN_NUMBER IN NUMBER, 
    P_IN_TYPE   IN VARCHAR2 
) RETURN VARCHAR2 IS
    P_OUT_RESULT VARCHAR2(20);

BEGIN
    IF P_IN_NUMBER IS NULL THEN
        P_OUT_RESULT :='';
      return P_OUT_RESULT;
    END IF;
    SELECT CODE
      INTO P_OUT_RESULT
      FROM CD_CODE_INTERVAL INTER
     WHERE INTER.TYPE_ID =  P_IN_TYPE
       AND P_IN_NUMBER >= INTER.SCOPE_START
       AND P_IN_NUMBER < INTER.SCOPE_END
       ;
    RETURN P_OUT_RESULT;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN P_OUT_RESULT;
END;
```

如上，这个函数是在查询一个CD_CODE_INTERVAL的表，对应的SQL在数据库中为如下样子：

```sql
SELECT CODE 
  FROM CD_CODE_INTERVAL INTER 
 WHERE INTER.TYPE_ID = :B2 
   AND :B1 >= INTER.SCOPE_START 
   AND :B1 < INTER.SCOPE_END
```

这个表行数不足1000行，SEGMENT SIZE不足1M。


4. **SQL中涉及对象的信息收集到了，下面查看一下当时的等待事件。**

问题时段同一条SQL的等待事件：

```sql
EVENT                            COUNT(0) SUM(TIME_WAITED)
------------------------------ ---------- ----------------
                                      394                0
cell smart table scan                 140            83817
reliable message                       98         35737546
enq: KO - fast object checkpoint        8              697
cursor: pin S                           1                0

```

正常时段同一条SQL的等待事件：

```sql
EVENT             COUNT(0) SUM(TIME_WAITED)
--------------- ---------- ----------------
                       282                0
cursor: pin S            1            21876
```

5. **对比同SQL在问题时段和正常时段的AWRSQRPT，观察资源消耗**

INSERT这条SQL执行计划是一样的，但是发现问题：

- 问题时段：执行3663秒，花费时间主要在IO上，disk reads=7831175 *8k /1024/1024= 59GB

- 正常时段：执行195秒，花费时间主要在CPU上，disk reads=965996 *8k /1024/1024= 7GB

标量函数里面的SQL执行计划也是一样的，但是发现问题：

- 问题时段：这个SQL才执行了1百万多次，但已经总共花费了3374秒，同时，物理IO占了35.6%

- 正常时段：这个SQL执行了6百万多次，但需要注意的是它总共才花费了477秒，同时，没有物理IO。


#### 三、结论分析

经过多次抓取历史报告对比，发现了一个现象，当这条sql存在物理IO的时候，相应的语句执行时间会比没有物理IO的慢。同时伴随着smart scan以及enq KO :fast object checkpoint和reliable message等待事件，这确实说明在这个等待事件的时间点上，oracle读取这个表的数据是从磁盘读取的。
由于这个函数在insert into select语句的select部分，作为标量子查询的函数，最终数据有多少条，就意味着会调用这个函数多少次，而且每次都需要进行物理IO请求，每一次请求的延时累积几百万次，就造成了整体的延时。
CD_CODE_INTERVAL表总大小才13个数据块，正常几次查询之后，这个数据块就会全部在buffer cache中，并处于lru的热端，正常情况下不需要再次进行物理读盘，但是发现在这个存储过程中，在它的前面将隐藏参数_serial_direct_path强制always，执行结束之后在恢复默认值。为了是强制Smart Scan而提升执行速度减少IO资源消耗。
根因是在批处理资源紧张的时间段，将buffer cache中CD_CODE_INTERVAL表的数据库全部刷出去之后，由于隐藏参数的参与，进行了强制的直接路径读，从而导致整体延时，其他时段没有发生这种数据全部刷出buffer cache的情况，整体上还有有很多数据在buffer cache中，所以并没有强制进行直路径读。








![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com




