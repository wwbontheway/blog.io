---
layout: post
title: 分区表全局索引失效的锅
date: 2013-09-02
categories: OracleDiagnosis
tags: [Oracle,Diagnosis]
description: 全局索引失效。
---

分享一起故障处理。。。

事情大概是这样的，月初上班，报表人员反应月初有一套报表生成很慢，请求信息部门协助分析。

公司的数据库是小w负责运维，遇到这种问题，首先的思路就是抓取头一天问题时段的AWR与ASH报告，定位报表存储过程中是哪些sql执行效率较低，根据经验我认为应该是有些sql语句运行有问题。并没有发现异常的等待事件，基本都是和IO有关系。观察TOP SQL发现有些sql运行很长，抓取其当时时刻的执行计划发现走了全表扫描，而sql中涉及到的se\_orders为一个月度分区表，并且执行计划中走了全分区的扫描，经验告诉我，这个sql的执行计划走错了，应该是可以使用索引等路径进行访问的。

经过查询，确实发现这个分区表有全局索引，但是发现这个索引失效了！询问开发发现夜间进行过truncate分区操作（报表人员有对表操作的权限），询问之后发现这个表为近期新上业务而使用的报表，只保留上一个月的数据的按月的分区表，表上只有一个全局索引，结果索引失效导致问题出现。

由于为新上线的业务，而且在上线时候也没有通知到我，其中的各种原因就不细说了。不过在发生这个事情之后，小w强烈建议将全局索引改为本地索引，这样即使后期再次进行truncate分区后没有维护索引，索引也不会失效。

至于开发对表的操作权限问题，是属于历史遗留问题，已经收不回来了，不要问我为什么，领导不让动，很无奈啊～～～


## 小知识点

- 抓取历史执行计划可以使用以下方法：
1. awrsqrpt
2. dbms_xplan.display_awr

- 养成一个习惯，当数据库或者业务人员反映数据库出现了之前没出现过的问题，第一点要分清是恢复业务优先还是诊断问题优先，不要闷着头找原因而耽搁了业务的恢复。在这篇博文的例子中，因为并不是很急，所以小w才按照平时养成的思路进行层层的排查。而且数据库（指的是和数据库有关的问题）出现问题，首先要明白为什么会出现这个问题，是否是业务开发有改动，有东西上线等等。当然也不排除是硬件或者真是数据库的问题，这里只是多提供一些思路和思考的角度。






![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com




