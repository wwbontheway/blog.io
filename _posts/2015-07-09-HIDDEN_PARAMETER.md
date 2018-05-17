---
layout: post
title: ORACLE隐藏参数查询
date: 2015-07-09
categories: OracleScripts
tags: [Oracle,Scripts]
description: oracle脚本。
---

ORACLE中以下划线开头的参数为隐藏参，如“_complex_view_merging”，正常在sqlplus中进行show parameter <para_name> 查询是查不到的。可以借助两个基表来联合查询：

```sql


SQL> desc x$ksppi

名称         是否为空? 类型
------------ -------- ---------------
ADDR                  RAW(4)           --内存地址
INDX                  NUMBER           --序号，从0开始
INST_ID               NUMBER           --instance number
KSPPINM               VARCHAR2(64)     --参数名称  
KSPPITY               NUMBER           --参数类型 1,'boolean' 2,'string', 3,'number',4,'file'
KSPPDESC              VARCHAR2(64)     --描述
KSPPIFLG              NUMBER           --标志字段（用来说明是isses_modifiable or issys_modifiable

SQL> desc x$ksppcv

名称          是否为空? 类型
------------- -------- -------------
ADDR                   RAW(4)         --内存地址
INDX                   NUMBER         --序号，从0开始
INST_ID                NUMBER         --instance number
KSPPSTVL               VARCHAR2(512)  --当前值
KSPPSTDF               VARCHAR2(9)    --缺省值
KSPPSTVF               NUMBER         --标志字段，用来说明('Modified' or 'System Modified' or is_adjusted)
KSPPSTCMNT             VARCHAR2(255)  --comment



```

查看隐藏参数的脚本：（将x$ksppi和x$ksppcv进行联合查询）：

```sql

 select nam.indx + 1 numb,
           nam.ksppinm name,
           val.ksppstvl value,
           nam.ksppity type,
           val.ksppstdf is_default,
           decode(bitand(nam.ksppiflg / 256, 1), 1, 'True', 'False') is_session_modifiable,
           decode(bitand(nam.ksppiflg / 65536, 3),
                 1,
                 'Immediate',
                 2,
                 'Deferred',
                 3,
                 'Immediate',
                 'False') is_system_modifiable,
          decode(bitand(val.ksppstvf, 7),
                 1,
                 'Modified',
                 4,
                 'System Modified',
                 'False') is_modified,
          decode(bitand(val.ksppstvf, 2), 2, 'True', 'False') is_adjusted,
          nam.ksppdesc description
     from x$ksppi nam, x$ksppsv val
    where nam.indx = val.indx
      and nam.ksppinm like '%view%';
 
 ```
 
输出结果如下：

```sql
 
NUMB NAME                           VALUE  TYPE IS_DEFAUL IS_SE IS_SYSTEM IS_MOD IS_AD DESCRIPTION
---- ------------------------------ -----  ---- --------- ----- --------- ------ ----- ---------------------------------------------
 997 _partition_view_enabled        TRUE      1 TRUE      True  Immediate False  False enable/disable partitioned views
1009 _complex_view_merging          TRUE      1 TRUE      True  Immediate False  False enable complex view merging
1010 _simple_view_merging           TRUE      1 TRUE      True  Immediate False  False control simple view merging performed by the
                                                                                       optimizer
                                           
1019 _distinct_view_unnesting       FALSE     1 TRUE      True  Immediate False  False enables unnesting of in subquery into distinc
                                                                                       t view
                                           
1025 _push_join_union_view          TRUE      1 TRUE      True  Immediate False  False enable pushing join predicate inside a union
                                                                                       all view
                                           
1026 _push_join_union_view2         TRUE      1 TRUE      True  Immediate False  False enable pushing join predicate inside a union
                                                                                       view
                                           
1067 _project_view_columns          TRUE      1 TRUE      True  Immediate False  False enable projecting out unreferenced columns of
                                                                                        a view
                                           
1285 optimizer_secure_view_merging  FALSE     1 FALSE     False Immediate False  False optimizer secure view merging and predicate p
                                                                                         ushdown/movearound
																						 
```


![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用来自网络资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com
