---
layout: post
title: Oracle 12.2静默安装之后需要做的调整
date: 2017-10-10
categories: Oracle
tags: [Oracle]
description: 12c静默安装
---


前两篇博文介绍了如何静默安装12c，还需要做一些必要的配置。

## 修改spfile参数

修改前，记得先备份一份，下面这些参数只是个例子，可以根据实际情况考虑更改

```sql
create pfile from spfile;

alter system set memory_target=0 scope=spfile;
alter system set sga_max_size=1g scope=spfile;
alter system set sga_target=1g scope=spfile;
alter system set pga_aggregate_target=1g scope=spfile;
alter system set processes=6000 scope=spfile;
alter system set DB_FILES=2000 scope=spfile;

alter system set log_archive_max_processes=2 scope=spfile;
alter system set nls_date_format='YYYY-MM-DD HH24:MI:SS' scope=spfile;
alter system set open_cursors=3000 scope=spfile;
alter system set open_links_per_instance=48 scope=spfile;
alter system set open_links=100 scope=spfile;
alter system set parallel_max_servers=20 scope=spfile;
alter system set session_cached_cursors=200 scope=spfile;
alter system set undo_retention=10800 scope=spfile;
alter system set fast_start_mttr_target=300 scope=spfile;
alter system set deferred_segment_creation=false scope=spfile;
alter system set "_external_scn_logging_threshold_seconds"=600 scope=spfile;
alter system set "_external_scn_rejection_threshold_hours"=24 scope=spfile;
alter system set result_cache_max_size=0 scope=spfile;
alter system set "_cleanup_rollback_entries"=2000 scope=spfile;
alter system set parallel_force_local=true scope=spfile;
alter system set "_gc_policy_time"=0 scope=spfile;
alter system set "_clusterwide_global_transactions"=false scope=spfile;	
alter system set "_library_cache_advice"=false scope=both;
alter system set db_cache_advice=off scope=both;
alter system set disk_asynch_io=false scope=spfile;
```

## 开启归档模式

正式生产环境，强烈推荐开启归档。

1. 首先创建归档文件目录

```shell
mkdir /backup/arch
chown oracle:dba /backup/arch
```

2. 设置归档格式及路径

```sql
alter system set log_archive_format='%d_%t_%s_%r.arc' scope=spfile;
alter system set log_archive_dest_1='location=/home/arch';
```

3. 开启归档

```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
```

4. 验证是否处于归档模式

```sql
archive log list;
alter system archive log current;
```

5. 查看操作系统是否有文件生成

```shell
[oracle@w12c ~]$ ls -ltr /backup/arch
total 166300
-rw-r----- 1 oracle oinstall 150238100 Oct 09 19:59 1_1_977070699.arc
```
 


## 改变控制文件备份路径

```sql
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/home/backup/backup/control%F';
```

## 附带一个单机环境全备脚本

其中的路径按照真实环境进行更改

```shell

BKDATE=`date +"%Y%m%d"` 
BKPATH=/backup/backup
export ORACLE_BASE=/oracle/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/12.2.0.1/db_1
export ORACLE_SID=wwb12c
export PATH=$ORACLE_HOME/bin:$PATH:$HOME/bin

rman target / log ${BKPATH}/rman_full_${BKDATE}.log append<<EOF
run
{
allocate channel c1 type disk;
allocate channel c2 type disk;
backup database filesperset 4 format '${BKPATH}/full_%d_%T_%U';
sql 'alter system archive log current';
backup archivelog all format '${BKPATH}/arch_%d_%T_%U' delete input;
backup current controlfile format '${BKPATH}/ctl_%d_%T_%U';
crosscheck backup;
crosscheck archivelog all;
delete noprompt obsolete;
delete noprompt expired backup;
delete noprompt expired archivelog all;
}
EOF

for tfile in $(find ${BKPATH}/ -mtime +14)
do
if [ -d $tfile ];then
rmdir $tfile
elif [ -f $tfile ];then
rm -f $tfile
fi

echo -e "---- Delete backup file: $tfile ------"

done
```
 

- 单机环境备归档脚本:

```shell

BKDATE=`date +"%Y%m%d"` 
BKPATH=/backup/backup
export ORACLE_BASE=/oracle/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/12.2.0.1/db_1
export ORACLE_SID=wwb12c
export PATH=$ORACLE_HOME/bin:$PATH:$HOME/bin
rman target / log ${BKPATH}/rman_arch_${BKDATE}.log append<<EOF
run
{
allocate channel c1 type disk;
allocate channel c2 type disk;
sql 'alter system archive log current';
backup archivelog all format '${BKPATH}/arch_%d_%T_%U' delete input;
backup current controlfile format '${BKPATH}/ctl_%d_%T_%U';
crosscheck backup;
crosscheck archivelog all;
delete noprompt expired backup;
delete noprompt expired archivelog all;
}
EOF

```

![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com




