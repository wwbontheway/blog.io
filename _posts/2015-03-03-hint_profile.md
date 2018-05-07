---
layout: post
title: HINT概述
date: 2015-3-02
categories: oracle优化
tags: [oracle优化,oracle基础知识]
description: 优化基础。
---

## HINT的简介
HINT在很多时候被称为“提示”，但是我还是习惯叫他“HINT”，它是在SQL语句中用来传递指示给oracle数据库优化器的注释，简而言之就是一种注释，只不过有自己的固定的格式。在SQL中，除了存在阻止优化器选择的一些条件之外，其余情况优化器都会采用这些HINT来给语句选择一个执行计划。

HINT在Oracle 7才引入的，为的在优化器生成了差强人意的执行计划时给用户些帮助。现在oracle提供了若干的工具，包括SQL优化顾问（SQL Tuning Advisor，STA），SQL计划管理（SQL plan management，SPM），和SQL性能分析器（SQL Performance Analyzer，SPA），来帮助你定位优化器不能解决的性能问题。Oracle强烈建议采用这些工具而不是用HINT。这些工具要远远优于HINT，因为在持续变化的基础上，伴随着你的数据量和数据库环境的变化，它们会提供最新的解决方案。

HINT应该谨慎使用，仅当你将相关的表的统计信息都收集完，并且用EXPLAIN PLAN语句评估了不带HINT的优化计划之后，才建议使用。在后续版本中更改数据库环境以及查询的性能的增强功能可能会对代码中带有HINT的部分的性能产生重大影响。即是说ORACLE官方并不是很建议使用HINT这个工具，但是为什么还要使用呢，在于很多时候，使用HINT可以带来立竿见影的效果，不过需要注意的是任何短期内用HINT方式带来的好处或许并不会长久的提升性能。（笑）


下面就介绍一下HINT的基本使用条件
## HINT的使用
一个语句块中只能有一个包含HINT的注释，而且这个注释必须紧跟在*SELECT*、*UPDATE*、*INSERT*、*MERGE*或*DELETE*关键字后面

下面的语法图展示了Oracle所支持的两种包含HINT的语法块的注释格式:

*hint：==*
![hint用法](https://docs.oracle.com/cd/E11882_01/server.112/e41084/img/hint.gif)

**需要注意的是： 加号（+）使Oracle将注释当作HINT的列表来解析。加号必须紧跟在注释定界符后面，不可以有空格（如--+ 或者/\*+ \*/）**
- 上图中*hint*是这段中需要考虑的hint的其中一个。加号和hint中间的空格是可写可不写的。如果注释包含多个hint，那么要用最少一个空格来分隔开这些hint。
- 上图中*string*是其他可以修饰补充hint的注释文本。
- 上图中--+语法要求全部注释都要在一行

## HINT被忽略的情况
在一下场景中，ORACLE数据库将会忽略HINT而且也不会返回任何报错：
- 1.HINT中包括拼错以及语法错误。不过，数据库会考虑在同一注释中其他正确的指定的HINT。*（曾经见过一个客户，将append这个HINT错误的写成了aaaapend，结果当然是没有生效啦～）*
- 2.包含HINT的注释没有紧跟在DELETE、INSERT、MERGE、SELECT或UPDATE关键字后面。
- 3.HINT的组合相互冲突。不过，数据库会考虑同一个注释中其他的HINT。
- 4.数据库环境采用PL\SQL 版本1，比如 Forms version 3 triggers, Oracle Forms 4.5, 和Oracle Reports 2.5。
- 5.全局HINT引用多个查询块。（查询块的概念后面会说到）

## 在HINT中指定查询块
你可以在多个hint中指定一个可选的查询块的名字（qb_name），用来指定HINT适用于哪个查询块。这种语法可以让你在外部查询中指定一个适用于内联视图中的HINT。
这个查询块参数的语法格式为@queryblock，其中queryblock是指定查询中某个查询块的标识符。查询块标识符可以是系统生成的或用户自定义的。 当在查询块中执行要应用到这个查询块本身的HINT，那么@queryblock的语法是可以忽略不写的。
至于系统生成的标识符的查看方式，可以用EXPLAIN PLAN这个查询语句的方法来获得。通过使用带有NO_QUERY_TRANSFORMATION这个HINT的EXPLAIN PLAN语句进行查询，可以确定预转换查询块名称。(关于这个HINT，后面的文章会说到)
用户指定的名字可以用QB_NAME这个HINT来设置，详见QB_NAME Hint（后面的文章会说到）

## 指定全局HINT
许多HINT既可以应用于特定的表或索引，也可以应用于视图内的表或作为索引列的一部分的列。 语法元素tablespec和indexspec定义了这些全局提示。

*tablespec：==*
![tablespec用法](https://docs.oracle.com/cd/E11882_01/server.112/e41084/img/tablespec.gif)

- 你必须明确指定在语句中出现表。如果语句里的表使用了一个别名，那么在HINT中就要使用别名而不是表名。然而，HINT中的表名是不需要包括其schema的名字，哪怕在语句中出现了schema名。（schema的意思
