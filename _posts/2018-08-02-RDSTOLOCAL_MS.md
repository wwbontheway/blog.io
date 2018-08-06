---
layout: post
title: 阿里云RDS for MySQL到自建数据库主从配置
date: 2018-08-02
categories: MySQL
tags: [MySQL]
description: 阿里云RDS for MySQL到自建数据库主从配置
---

测试了一下阿里云RDS for MySQL到自建数据库的可行性及主从配置。

注：文中涉及信息,约定如下：

|约定条目|约定信息|
|:--|:--|
|主库地址|masterhost.mysql.rds.aliyuncs.com|
|从库地址|slavehost|
|密码信息|Password!123|
|IP信息|10.10.10.10|

### 环境

主节点：阿里云rds平台，MySQL 5.6.16

从节点：云虚拟机平台，MySQL 5.6.41

注：从库小版本可以高于主库

### 从库准备

```shell
[root@slavehost software]# lsb_release  -a
LSB Version:    :core-4.1-amd64:core-4.1-noarch
Distributor ID: CentOS
Description:    CentOS Linux release 7.4.1708 (Core) 
Release:        7.4.1708
Codename:       Core
```

检查一下虚拟机的防火墙时候关闭并且是否是开机自启：

```
[root@slavehost software]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
[root@slaveshost software]# systemctl is-enabled firewalld
disabled
```

如果开启，可以使用下面命令关闭：

```
systemctl stop firewalld    --关闭
systemctl disable firewalld --关闭开机自启
```

安装必用的包（不安装的话，在MySQL 5.6初始化的时候，会报错）：

```
yum install -y perl-Module-Install.noarch
yum install -y libaio*
```


目前MySQL 5.6最新版本为5.6.41，在官方下载相应介质，进行安装。
介质下载链接：
https://dev.mysql.com/downloads/mysql/5.6.html#downloads

安装MySQL5.6的过程略，请参考之前的博客或官方手册。需要注意的是，云上的主从为GTID方式，需要配置相应参数。

另外需要注意的是，GTID本身有些限制：

- 1.不支持非事务引擎（因为GTID的T就是事务transaction啊~）
- 2.不允许一个SQL同时更新一个事务引擎表和非事务引擎表（和上一条一样的道理）
- 3.不支持CTAS语句复制(主库直接报错，因为会生成两个sql，一个是DDL建表语句，一个是insert into 插入数据语句。但是由于DDL会导致自动提交，所以这个sql至少需要两个GTID才可以，但是GTID模式下，这个sql只会被生成一个GTID，所以就报错了。 ）
- 4.在一个复制组中，必须要求统一开启GTID或者是关闭GTID
- 5.不支持sql_slave_skip_counter参数
- 6.对于create和drop temporary table 不支持

另外在5.7之前，开启GTID功能是需要重启的。而且开启GTID后，就不再使用原来的传统复制方式（即通过binlog和position复制的方式不可以用了）。

小w在做的时候，虚拟机的空间不足，于是挂载了一个500g的数据盘，具体方法在阿里云服务支持上有具体的操作，这里面就不详细介绍了，附上官方链接：

https://help.aliyun.com/document_detail/25426.html?spm=5176.11065259.1996646101.searchclickresult.36154926KKMpZG

挂载到/data目录下，查看：

```
[root@slavehost software]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        40G   32G  5.3G  86% /
devtmpfs        7.8G     0  7.8G   0% /dev
tmpfs           7.8G     0  7.8G   0% /dev/shm
tmpfs           7.8G  340K  7.8G   1% /run
tmpfs           7.8G     0  7.8G   0% /sys/fs/cgroup
tmpfs           1.6G     0  1.6G   0% /run/user/0
/dev/vdb1       493G   18G  450G   4% /data
```

### 主从数据初始化

准备就绪之后，远程到阿里云的rds中备份数据（这里面没有备份三个自带的系统库information_schema、performance_schema、mysql，因为云上面的用户很多读取mysql等系统库的权限是没有的，-A会导致备份报错而终止，使备份集不可靠。）：

云上查询mysql库的部分表：

```sql
mysql> select count(0) from mysql.user;
ERROR 1142 (42000):  command denied to user 'wwb'@'10.10.10.10' for table 'user'
mysql> select count(0) from mysql.db;
ERROR 1142 (42000):  command denied to user 'wwb'@'10.10.10.10' for table 'db'
```

如果-A备份的话，在读取到mysql库的表的时候，会报下面的错

```
mysqldump: Couldn't execute 'show create table `db`': SHOW command denied to user 'wwb'@'10.10.10.10' for table 'db' (1142)
```

备份主库

```
[root@slavehost data]# db=`mysql -hmasterhost.mysql.rds.aliyuncs.com -uwwb -p -BN -e "show databases;"|egrep -v "mysql|information_schema|performance_schema"`

[root@slavehost data]# mysqldump -hmasterhost.mysql.rds.aliyuncs.com -uwwb -p  --single-transaction --master-data=2 -B $db > all.sql

Enter password: 
Warning: A partial dump from a server that has GTIDs will by default include the GTIDs of all transactions, even those that changed suppressed parts of the database. If you don't want to restore GTIDs, pass --set-gtid-purged=OFF. To make a complete dump, pass --all-databases --triggers --routines --events.

[root@slavehost data]# ls -ltrh all.sql 
-rw-r--r-- 1 root root 51G Aug  1 11:50 all.sql
```

由于数据量才几十G，这里使用mysqldump方式进行备份，通过网络的备份，速度取决于网络和本机IO。

待备份完成之后，将数据恢复到从库：

```
[root@slavehost data]# mysql -uroot -p  -S /data/mysql/3306/tmp/3306.sock < all.sql 
Enter password: 
```

恢复过程中，可以使用下面命令监控数据库空间大小：

```
watch df -m
```

注意：如果从库有一些数据库，想清理掉。但是删除数据库的时候，如果数据库的名字含有中划线“-”（例如：w-test这种名字的数据库），需要加反引号将数据库名字括起来（即：drop database \`w-test\`;），不然会报错：

```
ERROR 1064 (42000): You have an error in your SQL syntax;
```

### 创建复制用户

主库中执行下面命令：

```sql
mysql> create user 'repl'@'%' identified by 'Password!123';
mysql> grant replication slave on *.* to 'repl'@'%';
mysql> flush privileges;
```

### 启动主从复制

待从库恢复完成之后，从库中执行下面命令：

```sql
mysql> change master to master_host = 'masterhost.mysql.rds.aliyuncs.com', master_port = 3306, master_user = 'repl', master_password='Password!123', master_auto_position = 1;

mysql> start slave;
```

### 监控数据同步状态

在从库中使用下面命令监控主从复制状态：

```sql
mysql> show slave status \G
```

会看到如下输出：

```sql
*************************** 1. row ***************************
               Slave_IO_State: Queueing master event to the relay log
                  Master_Host: masterhost.mysql.rds.aliyuncs.com
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.001126
          Read_Master_Log_Pos: 56124180
               Relay_Log_File: relay-log.000002
                Relay_Log_Pos: 1157585
        Relay_Master_Log_File: mysql-bin.001125
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: mysql.%,information_schema.%,performance_schema.%
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 489539608
              Relay_Log_Space: 92031977
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 10353
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 2000968120
                  Master_UUID: 22d70b93-c109-11e7-82f2-6c92bf3668a4
             Master_Info_File: /data/mysql/3306/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Reading event from the relay log
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 22d70b93-c109-11e7-82f2-6c92bf3668a4:49023117-49087200
            Executed_Gtid_Set: 22d70b93-c109-11e7-82f2-6c92bf3668a4:1-49024072,
437083b7-c109-11e7-82f3-6c92bf477039:1-148778030
                Auto_Position: 1
                
```

注意观察上面Slave_IO_Running和Slave_SQL_Running显示为 Yes，表示从库复制进程正常并正在工作，Seconds_Behind_Master值不为零，表示从库正在追平数据初始化过程中缺失的差异数据。
等待一段时间再次查看，数据追平：

```sql
mysql> show slave status\G
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    15
Current database: *** NONE ***

*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: masterhost.mysql.rds.aliyuncs.com
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.001130
          Read_Master_Log_Pos: 19697726
               Relay_Log_File: relay-log.000017
                Relay_Log_Pos: 19697896
        Relay_Master_Log_File: mysql-bin.001130
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: mysql.%,information_schema.%,performance_schema.%
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 19697726
              Relay_Log_Space: 19698181
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 2000968120
                  Master_UUID: 22d70b93-c109-11e7-82f2-6c92bf3668a4
             Master_Info_File: /data/mysql/3306/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 22d70b93-c109-11e7-82f2-6c92bf3668a4:49023117-49510318
            Executed_Gtid_Set: 22d70b93-c109-11e7-82f2-6c92bf3668a4:1-49510318,
437083b7-c109-11e7-82f3-6c92bf477039:1-148778030
                Auto_Position: 1
```

至此，主从搭建测试完成。


### 云主库搭建主从到自建数据库的注意事项

- 1、备份主库时候确保空间充足。

- 2、在备份主库数据的时候，三个系统库是不需要备份的（mysql、information_schema、performance_schema），因为阿里云RDS for MySQL对用户权限控制是很严格的，经过二次开发，系统库中很多表是没有读权限，因此也无法备份出来，如下：

```sql
mysql> select count(0) from mysql.db;
ERROR 1142 (42000):  command denied to user 'wwb'@'10.10.10.10' for table 'db'
```

- 3、从库配置文件需添加必要的参数

```
server-id = 885433306  

binlog_format = row
log-bin = /data/mysql/3306/logs/mysql-bin
log-bin-index = mysql-bin.index
        
gtid_mode = on
enforce_gtid_consistency = on                    
log-slave-updates = 1

innodb_file_per_table = on
relay-log  = relay-log
relay_log_index  = relay-log.index

skip_slave_start = 1
relay_log_purge = 1
relay_log_recovery = 1

replicate_wild_ignore_table = mysql.%
replicate_wild_ignore_table = information_schema.%
replicate_wild_ignore_table = performance_schema.%
```

其中，server\-id 必须与主库不同。mysql、information_schema、performance_schema这三个库中所有的表不进行复制，这里建议设置replicate_wild_ignore_table参数来代替replicate_ignore_db参数。

- 4、由于主从的主机资源可能有差异，建议从库的参数按照主库进行配置，如：

主库：

```sql
mysql> show variables like 'max_binlog_cache_size';
+-----------------------+----------------------+
| Variable_name         | Value                |
+-----------------------+----------------------+
| max_binlog_cache_size | 18446744073709547520 |
+-----------------------+----------------------+
```

从库：

```sql
mysql> show variables like 'max_binlog_cache_size';
+-----------------------+---------+
| Variable_name         | Value   |
+-----------------------+---------+
| max_binlog_cache_size | 4194304 |
+-----------------------+---------+
```

如果上面参数不同，在复制的时候可能会报Error_code: 1197 错误。建议改为一致：

从库执行：

```sql
mysql> SET GLOBAL max_binlog_cache_size =18446744073709547520;
```



![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com


