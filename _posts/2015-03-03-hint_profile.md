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

![hint用法](https://docs.oracle.com/cd/E11882_01/server.112/e41084/img/hint.gif)
