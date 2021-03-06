---
layout: post
title: MongoDB复制集搭建（上）
date: 2018-05-29
categories: MongoDB
tags: [MongoDB]
description: MongoDB复制集搭建
---

介绍一下MongoDB的复制集的搭建，这部分主要介绍MongoDB的安装配置～～

## 下载

MongoDB可以通过官方网站 https://www.mongodb.com/download-center 下载

## 环境信息

仲裁节点（Arbiter）不需要存储数据，所以由于是实验环境，就与Primary节点安装在同一台机器，只要端口号不同即可，MongoDB默认端口号是27017，可以根据需要更改

操作系统|主机名| IP|端口号| MongoDB角色 
:-----|:--------|:--------|:-----|:---------
RHEL6.5|wmaster|192.168.56.123|27017 |Primary
RHEL6.5|wmaster|192.168.56.123 |27020|Arbiter
RHEL6.5|wslave|192.168.56.128 |27017|Scendary

## 安装软件

其实解压即用～
创建必要的目录，建议创建mongo用户来管理MongoDB，并添加环境变量

```shell
tar xvf /soft/mongodb-linux-x86_64-rhel70-3.4.9.tgz
mv /soft/mongodb-linux-x86_64-rhel70-3.4.9 /usr/local/mongodb
mkdir -vp /usr/local/mongodb/{conf,logs,data,arbiter1}
useradd mongo
chown -R mongo:mongo /usr/local/mongodb/
echo "export MONGO_HOME=/usr/local/mongodb" >> /home/mongo/.bash_profile
echo "export PATH=$PATH:$MONGO_HOME/bin" >> /home/mongo/.bash_profile
```

## 编辑配置文件

#### Primary的配置文件

```shell
[mongo@wmaster conf]$ cat primary27017.cnf 
systemLog:
  destination: file
  path: /usr/local/mongodb/logs/shard1.log 
  logAppend: true
storage: 
  journal:
    enabled: true
  dbPath: /usr/local/mongodb/data 
  directoryPerDB: true
  engine: wiredTiger 
  wiredTiger:
    engineConfig: 
      cacheSizeGB: 1 
      directoryForIndexes: true
    collectionConfig: 
      blockCompressor: zlib
    indexConfig: 
      prefixCompression: true
net:
  port: 27017
processManagement: 
  fork: true
replication:
  oplogSizeMB: 512
  replSetName: rep1
sharding:
  clusterRole: shardsvr
```

#### 仲裁节点的配置文件

```shell
[mongo@wmaster conf]$ cat arbiter1.conf 
systemLog:
  destination: file
  path: /usr/local/mongodb/logs/arbiter1.log 
  logAppend: true
storage: 
  journal:
    enabled: true
  dbPath: /usr/local/mongodb/arbiter1 
  directoryPerDB: true
  engine: wiredTiger 
  wiredTiger:
    engineConfig: 
      cacheSizeGB: 1 
      directoryForIndexes: true
    collectionConfig: 
      blockCompressor: zlib
    indexConfig: 
      prefixCompression: true
net:
  port: 27020
processManagement: 
  fork: true
replication:
  oplogSizeMB: 512
  replSetName: rep1
sharding:
  clusterRole: shardsvr
```

#### Secondary的配置文件

```shell
[mongo@wslave conf]$ cat secondary27017.cnf 
systemLog:
  destination: file
  path: /usr/local/mongodb/logs/shard1.log 
  logAppend: true
storage: 
  journal:
    enabled: true
  dbPath: /usr/local/mongodb/data 
  directoryPerDB: true
  engine: wiredTiger 
  wiredTiger:
    engineConfig: 
      cacheSizeGB: 1 
      directoryForIndexes: true
    collectionConfig: 
      blockCompressor: zlib
    indexConfig: 
      prefixCompression: true
net:
  port: 27017
processManagement: 
  fork: true
replication:
  oplogSizeMB: 512
  replSetName: rep1
sharding:
  clusterRole: shardsvr
```
## 启动MongoDB

master节点:

```shell
[mongo@wmaster conf]$ mongod -f /usr/local/mongodb/conf/primary27017.cnf 
about to fork child process, waiting until server is ready for connections.
forked process: 2674
child process started successfully, parent exiting
[mongo@wmaster conf]$ mongod -f /usr/local/mongodb/conf/arbiter1.conf 
about to fork child process, waiting until server is ready for connections.
forked process: 2749
child process started successfully, parent exiting
[mongo@wmaster conf]$ ps aux | grep -i mongod
mongo     2674  0.7  4.6 1062196 47852 ?       Sl   12:54   0:02 mongod -f /usr/local/mongodb/conf/primary27017.cnf
mongo     2749  0.5  4.3 1062196 44892 ?       Sl   12:58   0:00 mongod -f /usr/local/mongodb/conf/arbiter1.conf
mongo     2795  0.0  0.0 103244   876 pts/0    S+   13:00   0:00 grep -i mongod
```

slave1节点:

```shell
[mongo@wslave ~]$ mongod -f /usr/local/mongodb/conf/secondary27017.cnf 
about to fork child process, waiting until server is ready for connections.
forked process: 2959
child process started successfully, parent exiting
[mongo@wslave ~]$ ps aux | grep -i mongod
mongo     2959  1.4  4.0 1062120 41040 ?       Sl   13:06   0:00 mongod -f /usr/local/mongodb/conf/secondary27017.cnf
mongo     2985  0.0  0.0 103244   880 pts/0    S+   13:06   0:00 grep -i mongod
```


#### To be continued...

![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com
