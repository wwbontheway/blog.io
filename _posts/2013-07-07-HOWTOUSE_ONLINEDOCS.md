---
layout: post
title: Oracle官方文档的用法
date: 2013-07-07
categories: Oracle
tags: [Oracle]
description: 官方文档。
---

分享一下自己的学习心得

一般的小问题，查一下百度啊，google啊，都没问题，不过要小心别人blog中的坑，毕竟当你是小白的时候，是分不出来哪些是坑的。

所以个人建议，还是查询官方文档吧，或者查询MOS，不过MOS不是每个公司都有账号的。这个有点尴尬。官方文档也不是从第一篇就开始看，小w个人一般是当作字典来用，比如我要查一下**移动分区**的命令是啥，那么我首先需要看哪篇官方文档呢？肯定是SQL LANGUAGE REFERENCE啊，这里面是对应的Oracle中的 SQL语法。这本文档在Oracle各个版本中是都有的，不过可能由于版本问题，语法有细微的差距。

那么首先，登陆联机文档，当然，也可以下载文档集合到本地，不过online doc有一点好处就是他可能会不定期的更新修正其中的错误，所以，有网络的情况下，还是用online的吧。联机文档的地址是：docs.oracle.com

进去之后当然是选择databases了，然后选择对应的数据库版本，这里面还是用我最爱的Oracle 11.2为例。

进去之后需要找到我们要看的语法文档，上面说了，叫做SQL LANGUAGE REFERENCE，在booklist中查找SQL开头的，会很快找到。进去文档之后，如果选择pdf格式的，可以直接查找alter table关键字，如果是使用网页格式的话，需要点击那个expand all将目录全展开，才能更快的查找到（Oracle这里做的还不错，检索目录就能找到想要的东西，很贴心），进去ALTER TABLE之后，在Additional Topics下面有个Syntax，这个就是语法树，当然，下面还有Example选项，这里面是一些语法使用的小例子，可以参考。

点击Syntax之后，会看到下面这张图片

![alter_table](https://docs.oracle.com/cd/E11882_01/server.112/e41084/img/alter_table.gif)


这个图片就是对应的语法树的主干，大写默认是关键字的表示，在语法中不能省略的。里面有很多分支，那么我上面举的例子是查询**移动分区**的语法，众所周知，分区肯定是分区表，所以继续点击下面的alter\_table\_partitioning子句，进入分区的语法，之后会看到下面这张图：

![partitoning](https://docs.oracle.com/cd/E11882_01/server.112/e41084/img/alter_table_partitioning.gif)

看着这么长，不要被吓到了，既然是要移动分区，那自然要查看这个语法树中的move\_table\_partition语法，注意这里说的是一级分区，不是二级分区（subpartition）。之后继续点击进去：

![move_part](https://docs.oracle.com/cd/E11882_01/server.112/e41084/img/move_table_partition.gif)


这个图片中，很多命令是可选的，即使在向右的直线上面，而直线上经过的选项是必选的，所以我们在看一下partition\_extanded\_name这个选项：


![part_ext_name](https://docs.oracle.com/cd/E11882_01/server.112/e41084/img/partition_extended_name.gif)

需要注意的是，移动分区，全局索引是会失效的，所以这里面要保证语法正确还要兼顾表结构的完整性，需要继续点击update\_index\_clauses子句：

![update_idx](https://docs.oracle.com/cd/E11882_01/server.112/e41084/img/update_index_clauses.gif)

进入之后再次点击update\_global\_index\_clause子句，看到下面语法树：

![update_G_idx](https://docs.oracle.com/cd/E11882_01/server.112/e41084/img/update_global_index_clause.gif)

这就很好选择了，然后move是可以并行操作的，可以返回上一层查看并行语法parallel\_clause:

![paral](https://docs.oracle.com/cd/E11882_01/server.112/e41084/img/parallel_clause.gif)

那么现在假设用户是wwb，表名字是wtab01，分区名是wp01，准备开四个并行，同时维护全局索引，那么满足我以上需求的移动分区完整的语法就应该是：

```sql

ALTER TABLE wwb.wtab01 MOVE PARTITION wtab01 UPDATE GLOBAL INDEXES PARALLEL 4;

```

这篇博文只是通过 SQL LANGUAGE REFERENCE 这个文档大概介绍了一下官方文档的用处，再比如如果想查询一个参数（比如db_block_size）是什么意思，有哪些可选值，亦或者查询某个动态性能视图，都可以查看REFERENCE文档，这里就不再一一介绍了。


![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com




