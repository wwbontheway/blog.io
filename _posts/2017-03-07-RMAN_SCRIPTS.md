---
layout: post
title: RMAN备份脚本
date: 2017-03-07
categories: OracleScripts
tags: [Oracle,Scripts]
description: 简易的Rman备份脚本。
---


有位朋友找我要一份RMAN的备份脚本，其实整体很简单，只要清楚都需要备份什么，什么步骤就可以了。
那么今天就分享一个简易的RMAN备份脚本。

## 全备脚本fullbackup.sh

```sql
 
#!/bin/bash

. /home/oracle/.bash_profile 

rman target /  log /backupsets/rman_full.log append<<EOF
run{
allocate channel c1 type disk;
allocate channel c2 type disk;
allocate channel c3 type disk;
allocate channel c4 type disk;
backup  database format '/backupsets/full_%d_%T_%s_%p';
SQL 'alter system archive log current';
SQL 'alter system archive log current';
backup archivelog  all delete input  format '/backupsets/arch_%d_%t_%s_%p'; 
backup current controlfile format '/backupsets/ctl_%d_%T_%s_%p';
backup spfile format '/backupsets/%d_%U.spfile';
}

EOF

rman target /  log /backupsets/rman_delete.log append<<EOF
allocate channel  for maintenance type disk;
crosscheck backup;
crosscheck archivelog all;
delete noprompt obsolete;
delete noprompt expired backup;
delete noprompt expired archivelog all;
EOF
exit

```

## 归档备份脚本backarch.sh

```sql
 
#!/bin/bash

. /home/oracle/.bash_profile 

rman target /  log /backupsets/rman_arch.log append<<EOF
run{
allocate channel c1 type disk;
allocate channel c2 type disk;
allocate channel c3 type disk;
SQL 'alter system archive log current';
backup archivelog  all delete input  format '/backupsets/arch_%d_%t_%s_%p'; 
backup current controlfile format '/backupsets/ctl_%d_%T_%s_%p';
}
EOF

rman target /  log /backupsets/rman_delete.log append<<EOF
allocate channel  for maintenance type disk;
crosscheck backup;
crosscheck archivelog all;
delete noprompt obsolete;
delete noprompt expired backup;
delete noprompt expired archivelog all;
EOF
exit

```


### 注意

使用之前，一定要根据自己的环境进行更改相对应的路径等，不要直接拿去用！！！

否则出现的任何问题，小w不会负责的，嘿嘿~





![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com



