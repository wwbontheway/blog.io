---
layout: post
title: HINT概述
date: 2015-3-02
categories: oracle优化
tags: [oracle优化,oracle基础知识]
description: 优化基础。
---

# HINT
HINT在很多时候被称为“提示”，但是我还是习惯叫他“hint”，它是在SQL语句中用来传递指示给oracle数据库优化器的注释，简而言之就是一种注释，只不过有自己的固定的格式。在sql中，除了存在阻止优化器选择的一些条件之外，其余情况优化器都会采用这些HINT来给语句选择一个执行计划。


