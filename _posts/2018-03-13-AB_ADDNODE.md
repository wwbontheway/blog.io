---
layout: post
title: 不寻常的rac添加节点
date: 2018-03-13
categories: Oracle
tags: [Oracle]
description: 不寻常的rac添加节点
---


由于一个客户的不寻常的需求，促成了这次不寻常的添加节点。为何说不寻常呢，因为正常的rac添加节点，一般要求是新加入的节点的os版本要和原rac种的节点的os版本一致，官方文档也是这么要求的。而这次，加入的节点版本与原rac中节点的版本是不一致的。那么会成功么，下面就是见证奇迹的时刻，嘿嘿~


## 基础环境

原rac中两个节点的os版本

```sql

cat /etc/redhat-release
Red Hat Enterprise Linux Server release 6.5 (Tikanga)

```

新加入的节点

```sql

Red Hat Enterprise Linux Server release 5.5 (Tikanga)

```

安装rac3节点上oracle软件的依赖包，创建与rac1、rac2相同id的用户，目录结构，环境变量等。网络配置，hosts文件，ntp等
调整/etc/sysctl.conf和/etc/security/limits.conf中文件和rac1、rac2相同。


更改三个节点的hosts文件

```sql

[oracle@rac3 dbs]$ cat /etc/hosts
# Do not remove the following line, or various programs
# that require network functionality will fail.
127.0.0.1       localhost localhost.localdomain localhost4 localhost4.localdomain4
::1     localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.56.161 rac1
192.168.56.162 rac2

192.168.56.163 rac1vip
192.168.56.164 rac2vip

10.10.10.3 rac1priv
10.10.10.4 rac2priv

192.168.56.165 racscanip

#standby ip
192.168.56.123 orasingle

192.168.56.168 rac3
192.168.56.169 rac3vip
10.10.10.5 rac3priv

```

配置三个节点间的互信
oracle和grid的都做

```sql

[oracle@rac3 ~]$ mkdir -m 755 .ssh
 cd .ssh
 ssh-keygen -t rsa
 ssh-keygen -t dsa
 scp rac1:/home/oracle/.ssh/authorized_keys ./
 cat *.pub >>authorized_keys
 scp authorized_keys rac1:/home/oracle/.ssh/
 scp authorized_keys rac2:/home/oracle/.ssh/

```
 
grid用户类似操作

之后两个用户都测试一下

```sql

ssh rac1 date;ssh rac2 date;ssh rac1priv date;ssh rac2priv date

```

将asm的盘在新机器上识别到
这里需要注意是什么方式的，实验中采用udev的方式

```sql

cd /etc/udev/rules.d/

[oracle@rac3 rules.d]$ cat 99-oracleasm.rules      
KERNEL=="sdb", OWNER="grid", GROUP="asmadmin", MODE="660"
KERNEL=="sdc", OWNER="grid", GROUP="asmadmin", MODE="660"

```

和两外两个节点一样的配置
之后start_udev

检查一下

```sql

[grid@rac3 ~]$ ls -ltr /dev/sd*
brw-r----- 1 root disk     8,  2 Mar  9 16:00 /dev/sda2
brw-r----- 1 root disk     8,  0 Mar  9 16:00 /dev/sda
brw-r----- 1 root disk     8,  1 Mar  9 16:00 /dev/sda1
brw-rw---- 1 grid asmadmin 8, 32 Mar  9 19:15 /dev/sdc
brw-rw---- 1 grid asmadmin 8, 16 Mar  9 19:15 /dev/sdb

```


## 添加开始

#### 1. 添加节点:添加cluster软件到rac3

只需在一个正常节点上执行

```sql

[grid@rac1 ~]$ cluvfy stage -post hwos -n rac1,rac2,rac3 -verbose  

```

只需在一个正常节点上执行，这个会检查所有节点的软件包安装情况等信息

```sql

[grid@rac1 ~]$ cluvfy stage -pre crsinst -ndbp,dbs,dbi -verbose 

```

根据上面预检查脚本调节对应参数


添加节点的脚本在原集群中的一个节点执行即可，这里是在rac1上执行的，在$GRID_HOME/oui/bin

```sql

[grid@rac1 bin]$cd $ORACLE_HOME/oui/bin
./addNode.sh "CLUSTER_NEW_NODES={rac3}" "CLUSTER_NEW_VIRTUAL_HOSTNAMES={rac3vip}"

```

如果预检查总卡在/etc/resolv.conf过不去并且这个文件在三个节点中都没有内容，可以用下面参数跳过预检查：

```sql

export IGNORE_PREADDNODE_CHECKS=Y 

```

之后重新运行

```sql

./addNode.sh "CLUSTER_NEW_NODES={rac3}" "CLUSTER_NEW_VIRTUAL_HOSTNAMES={rac3vip}"

```

之后按照要求在新添加的rac3节点运行2个root脚本

```sql

[root@rac3 ~]# /oracle/app/oraInventory/orainstRoot.sh
Creating the Oracle inventory pointer file (/etc/oraInst.loc)

Changing permissions of /oracle/app/oraInventory.
Adding read,write permissions for group.
Removing read,write,execute permissions for world.

Changing groupname of /oracle/app/oraInventory to oinstall.
The execution of the script is complete.
[root@rac3 ~]# 
[root@rac3 ~]# 
[root@rac3 ~]# /oracle/11.2.0/grid/crs/root.sh 
Performing root user operation for Oracle 11g 

The following environment variables are set as:
    ORACLE_OWNER= grid
    ORACLE_HOME=  /oracle/11.2.0/grid/crs

Enter the full pathname of the local bin directory: [/usr/local/bin]: 
   Copying dbhome to /usr/local/bin ...
   Copying oraenv to /usr/local/bin ...
   Copying coraenv to /usr/local/bin ...


Creating /etc/oratab file...
Entries will be added to the /etc/oratab file as needed by
Database Configuration Assistant when a database is created
Finished running generic part of root script.
Now product-specific root actions will be performed.


Using configuration parameter file: /oracle/11.2.0/grid/crs/crs/install/crsconfig_params
Creating trace directory
User ignored Prerequisites during installation
Installing Trace File Analyzer
OLR initialization - successful
Adding Clusterware entries to inittab
CRS-4402: The CSS daemon was started in exclusive mode but found an active CSS daemon on node rac1, number 1, and is terminating
An active cluster was found during exclusive startup, restarting to join the cluster
clscfg: EXISTING configuration version 5 detected.
clscfg: version 5 is 11g Release 2.
Successfully accumulated necessary OCR keys.
Creating OCR keys for user 'root', privgrp 'root'..
Operation successful.
Preparing packages for installation...
cvuqdisk-1.0.9-1
Configure Oracle Grid Infrastructure for a Cluster ... succeeded

```


#### 2. 添加节点:添加数据库软件到rac3

这个脚本是在$ORACLE_HOME/oui/bin下面，在原集群中一个节点上运行即可，这里是rac1

```sql

[oracle@rac1 ~]$ cd /oracle/app/oracle/product/11.2.0/db_1/oui/bin
[oracle@rac1 ~]$ ./addNode.sh -silent "CLUSTER_NEW_NODES={rac3}"

```

之后按照提示在rac3运行root脚本：

```sql

[root@rac3 ~]# /oracle/app/oracle/product/11.2.0/db_1/root.sh
Performing root user operation for Oracle 11g 

The following environment variables are set as:
    ORACLE_OWNER= oracle
    ORACLE_HOME=  /oracle/app/oracle/product/11.2.0/db_1

Enter the full pathname of the local bin directory: [/usr/local/bin]: 
The contents of "dbhome" have not changed. No need to overwrite.
The contents of "oraenv" have not changed. No need to overwrite.
The contents of "coraenv" have not changed. No need to overwrite.

Entries will be added to the /etc/oratab file as needed by
Database Configuration Assistant when a database is created
Finished running generic part of root script.
Now product-specific root actions will be performed.
Finished product-specific root actions.

```

之后更改rac3的密码文件，参数文件指向等。


#### 3. 添加数据库实例

需要先注意一下监听状态：

```sql

[oracle@rac1 ~]$ dbca -h

[oracle@rac1 ~]$ lsnrctl status

LSNRCTL for Linux: Version 11.2.0.4.0 - Production on 09-MAR-2018 18:23:00

Copyright (c) 1991, 2013, Oracle.  All rights reserved.

Connecting to (ADDRESS=(PROTOCOL=tcp)(HOST=)(PORT=1521))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 11.2.0.4.0 - Production
Start Date                09-MAR-2018 14:27:28
Uptime                    0 days 3 hr. 55 min. 32 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /oracle/11.2.0/grid/crs/network/admin/listener.ora
Listener Log File         /oracle/app/11.2.0/diag/tnslsnr/rac1/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=LISTENER)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=192.168.56.161)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=192.168.56.163)(PORT=1521)))
Services Summary...
Service "+ASM" has 1 instance(s).
  Instance "+ASM1", status READY, has 1 handler(s) for this service...
Service "o11204" has 1 instance(s).
  Instance "o112041", status READY, has 1 handler(s) for this service...
Service "o11204XDB" has 1 instance(s).
  Instance "o112041", status READY, has 1 handler(s) for this service...
The command completed successfully
[oracle@rac1 ~]$ dbca -silent -addInstance -nodeList rac3 -gdbName o11204 -instanceName o112043 -sysDBAUserName sys -sysDBAPassword oracle
Adding instance
1% complete
2% complete
6% complete
13% complete
20% complete
26% complete
33% complete
40% complete
46% complete
53% complete
66% complete
Completing instance management.
76% complete
100% complete
Look at the log file "/oracle/app/oracle/cfgtoollogs/dbca/o11204/o11204.log" for further details.

```


## 添加完成

查看集群和数据库状态

```sql

[oracle@rac3 ~]$sqlplus / as sysdba

SQL> select inst_id,status from gv$instance;

   INST_ID STATUS
---------- ------------
         2 OPEN
         1 OPEN
         3 OPEN

SQL> show parameter name

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
cell_offloadgroup_name               string
db_file_name_convert                 string
db_name                              string      o11204
db_unique_name                       string      o11204
global_names                         boolean     FALSE
instance_name                        string      o112043
lock_name_space                      string
log_file_name_convert                string
processor_group_name                 string
service_names                        string      o11204
SQL> !cat /etc/redhat-release
Red Hat Enterprise Linux Server release 5.5 (Tikanga)

[grid@rac3 ~]$ olsnodes -n
rac1    1
rac2    2
rac3    3

[root@rac3 ~]# cat /etc/oratab
+ASM3:/oracle/11.2.0/grid/crs:N         # line added by Agent
o11204:/oracle/app/oracle/product/11.2.0/db_1:N         # line added by Agent

```

### 参考文档

Clusterware Administration and Deployment Guide 11g Release 2 (11.2)--4 Adding and Deleting Cluster Nodes
Real Application Clusters Administration and Deployment Guide 11g Release 2 (11.2)--10 Adding and Deleting Oracle RAC from Nodes on Linux and UNIX Systems




![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com




