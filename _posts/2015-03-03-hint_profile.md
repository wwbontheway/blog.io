---
layout: post
title: HINT概述
date: 2015-3-02
categories: oracle优化
tags: [oracle优化,oracle基础知识]
description: 优化基础。
---

# HINT
HINT在很多时候被称为“提示”，但是我还是习惯叫他“hint”，它是在SQL语句中用来传递指示给oracle数据库优化器的注释，简而言之就是一种注释，只不过有自己的固定的格式。在SQL中，除了存在阻止优化器选择的一些条件之外，其余情况优化器都会采用这些HINT来给语句选择一个执行计划。

HINT在Oracle 7才引入的，为了在优化器生产了差强人意的执行计划时给用户些帮助。现在oracle提供了若干的工具，包括SQL优化顾问（SQL Tuning Advisor，STA），SQL计划管理（SQL plan management，SPM），和SQL性能分析器（SQL Performance Analyzer，SPA），来帮助你定位优化器不能解决的性能问题。Oracle强烈建议采用这些工具而不是用HINT。这些工具要远远优于HINT，因为它们在持续使用时会随着您的数据和数据库环境的变化而提供全新的解决方案。
