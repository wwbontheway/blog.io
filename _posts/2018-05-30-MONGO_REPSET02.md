---
layout: post
title: MongoDB复制集搭建（下）
date: 2018-05-30
categories: MongoDB
tags: [MongoDB]
description: MongoDB复制集搭建
---

接上篇，本部分主要介绍MongoDB复制集的配置以及一些简单的测试～

## 环境信息

操作系统|主机名| IP|端口号| MongoDB角色 
:-----|:--------|:--------|:-----|:---------
RHEL6.5|wmaster|192.168.56.123|27017 |Primary
RHEL6.5|wmaster|192.168.56.123 |27020|Arbiter
RHEL6.5|wslave|192.168.56.128 |27017|Scendary


## 初始化复制集

登入任意一台机器的mongodb执行，因为是全新的复制集，所以可以任意进入一台执行。如果是一台有数据，则需要在有数据上执行。如果多台有数据则不能初始化。

```sql
[mongo@wmaster ~]$ mongo -host 192.168.56.123 -port 27017
MongoDB shell version v3.4.9
connecting to: mongodb://192.168.56.123:27017/
MongoDB server version: 3.4.9
...##省略部分显示输出
2018-05-29T12:54:12.960+0800 I CONTROL  [initandlisten]
> config={_id:'rep1',members:[{_id:0,host:'192.168.56.123:27017'},{_id:1,host:'192.168.56.128:27017'}, {_id:2,host:'192.168.56.123:27020',arbiterOnly:true}]}
{
	"_id" : "rep1",
	"members" : [
		{
			"_id" : 0,
			"host" : "192.168.56.123:27017"
		},
		{
			"_id" : 1,
			"host" : "192.168.56.128:27017"
		},
		{
			"_id" : 2,
			"host" : "192.168.56.123:27020",
			"arbiterOnly" : true
		}
	]
}
> rs.initiate(config)
{ "ok" : 1 }
rep1:PRIMARY> rs.status()
{
	"set" : "rep1",
	"date" : ISODate("2018-05-29T05:17:13.334Z"),
	"myState" : 1,
	"term" : NumberLong(1),
	"heartbeatIntervalMillis" : NumberLong(2000),
	"optimes" : {
		"lastCommittedOpTime" : {
			"ts" : Timestamp(1527542233, 1),
			"t" : NumberLong(1)
		},
		"appliedOpTime" : {
			"ts" : Timestamp(1527542233, 1),
			"t" : NumberLong(1)
		},
		"durableOpTime" : {
			"ts" : Timestamp(1527542233, 1),
			"t" : NumberLong(1)
		}
	},
	"members" : [
		{
			"_id" : 0,
			"name" : "192.168.56.123:27017",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 1381,
			"optime" : {
				"ts" : Timestamp(1527541999, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2018-05-29T05:17:09Z"),
			"electionTime" : Timestamp(1527541999, 1),
			"electionDate" : ISODate("2018-05-29T05:13:47Z"),
			"configVersion" : 1,
			"self" : true
		},
		{
			"_id" : 1,
			"name" : "192.168.56.128:27017",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 216,
			"optime" : {
				"ts" : Timestamp(1527542239, 1),
				"t" : NumberLong(1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1527542239, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2018-05-29T05:17:09Z"),
			"optimeDurableDate" : ISODate("2018-05-29T05:17:09Z"),
			"lastHeartbeat" : ISODate("2018-05-29T05:17:11.585Z"),
			"lastHeartbeatRecv" : ISODate("2018-05-29T05:17:12.506Z"),
			"pingMs" : NumberLong(0),
			"syncingTo" : "192.168.56.123:27017",
			"configVersion" : 1
		},
		{
			"_id" : 2,
			"name" : "192.168.56.123:27020",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
			"uptime" : 216,
			"lastHeartbeat" : ISODate("2018-05-29T05:17:11.502Z"),
			"lastHeartbeatRecv" : ISODate("2018-05-29T05:17:09.206Z"),
			"pingMs" : NumberLong(0),
			"configVersion" : 1
		}
	],
	"ok" : 1
}
```

这里面先定义了一个名为config的JSON集合
```
config={_id:'rep1',members:[{_id:0,host:'192.168.56.123:27017'},{_id:1,host:'192.168.56.128:27017'}, {_id:2,host:'192.168.56.123:27020',arbiterOnly:true}]}
```
然后使用集合中的元素进行初始化：rs.initiate(config)
初始化成功之后，\>号就变成了rep1:PRIMARY\> ，表示登录的这个实例为Primary节点。

登录Secondary节点，验证一下：

```sql
[mongo@wmaster ~]$ mongo -host 192.168.56.128 -port 27017
MongoDB shell version v3.4.9
connecting to: mongodb://192.168.56.128:27017/
MongoDB server version: 3.4.9
...##省略部分显示输出
2018-05-29T13:06:06.374+0800 I CONTROL  [initandlisten]
rep1:SECONDARY>
```

如上，192.168.56.128:27017显示为“rep1:SECONDARY>”，即为Secondary节点。然后在从库上执行rs.slaveOk()命令确认从库（先演示一下不执行rs.slaveOk，则在查询的时候会报错，这是因为当前secondary是不允许读写的，看errmsg部分）：

```sql
rep1:SECONDARY> show dbs
2018-05-29T13:57:22.786+0800 E QUERY    [thread1] Error: listDatabases failed:{
	"ok" : 0,
	"errmsg" : "not master and slaveOk=false",
	"code" : 13435,
	"codeName" : "NotMasterNoSlaveOk"
} :
_getErrorWithCode@src/mongo/shell/utils.js:25:13
Mongo.prototype.getDBs@src/mongo/shell/mongo.js:62:1
shellHelper.show@src/mongo/shell/utils.js:769:19
shellHelper@src/mongo/shell/utils.js:659:15
@(shellhelp2):1:1
rep1:SECONDARY> rs.slaveOk()
```
当然，也可以最开始只初始化一个主节点，然后在使用rs.add()添加复制节点（例如：rs.add('192.168.168.128:27017')），使用rs.addArb添加仲裁节点（例如：rs.addArb('192.168.168.123:27020')），这里不再做详细演示。



## 验证复制集Secondary节点的复制功能

#### 在Primary节点上插入数据

```sql
[mongo@wmaster ~]$ mongo -host wmaster -port 27017
MongoDB shell version v3.4.9
connecting to: mongodb://wmaster:27017/
MongoDB server version: 3.4.9
...
rep1:PRIMARY> show dbs
admin  0.000GB
local  0.000GB
test   0.000GB
rep1:PRIMARY> use test
switched to db test
rep1:PRIMARY> show collections
rep1:PRIMARY> db.users.insert({id:1,name:'wwb'})
WriteResult({ "nInserted" : 1 })
rep1:PRIMARY> db.users.find()
{ "_id" : ObjectId("5b177a87c05416c97321808e"), "id" : 1, "name" : "wwb" }
```

#### 在Secondary节点查询

```sql
rep1:SECONDARY> show dbs
admin  0.000GB
local  0.000GB
test   0.000GB
rep1:SECONDARY> use test
switched to db test
rep1:SECONDARY> show collections
users
rep1:SECONDARY> db.users.find()
{ "_id" : ObjectId("5b177a10c05416c97321808d"), "id" : 1, "name" : "wwb" }
```

#### 验证一下仲裁节点能否看到数据

```sql
[mongo@wmaster ~]$ mongo -host wmaster -port 27020
MongoDB shell version v3.4.9
connecting to: mongodb://wmaster:27020/
MongoDB server version: 3.4.9
...
rep1:ARBITER> db
test
rep1:ARBITER> rs.slaveOk()
rep1:ARBITER> show collections
rep1:ARBITER> db.users.find()
Error: error: {
	"ok" : 0,
	"errmsg" : "node is not in primary or recovering state",
	"code" : 13436,
	"codeName" : "NotMasterOrSecondary"
}
rep1:ARBITER>
```

由此可见，仲裁节点并没有同步数据，因为它仅是用来投票的。

## 测试故障转移

#### 将Primary节点关闭

```sql
[mongo@wmaster ~]$ mongo
MongoDB shell version v3.4.9
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.4.9
...
rep1:PRIMARY>
rep1:PRIMARY> use admin
switched to db admin
rep1:PRIMARY> db.shutdownServer()
server should be down...
2018-05-29T14:19:29.359+0800 I NETWORK  [thread1] trying reconnect to 127.0.0.1:27017 (127.0.0.1) failed
2018-05-29T14:19:29.438+0800 I NETWORK  [thread1] Socket recv() Connection reset by peer 127.0.0.1:27017
2018-05-29T14:19:29.438+0800 I NETWORK  [thread1] SocketException: remote: (NONE):0 error: 9001 socket exception [RECV_ERROR] server [127.0.0.1:27017]
2018-05-29T14:19:29.438+0800 I NETWORK  [thread1] reconnect 127.0.0.1:27017 (127.0.0.1) failed failed
2018-05-29T14:19:29.442+0800 I NETWORK  [thread1] trying reconnect to 127.0.0.1:27017 (127.0.0.1) failed
2018-05-29T14:19:29.442+0800 W NETWORK  [thread1] Failed to connect to 127.0.0.1:27017, in(checking socket for error after poll), reason: Connection refused
2018-05-29T14:19:29.442+0800 I NETWORK  [thread1] reconnect 127.0.0.1:27017 (127.0.0.1) failed failed
>
```

#### 查看其他两个节点的状态：

```sql
rep1:SECONDARY>rs.status()
{
	"set" : "rep1",
	"date" : ISODate("2018-05-29T06:21:45.738Z"),
	"myState" : 1,
	"term" : NumberLong(2),
	"heartbeatIntervalMillis" : NumberLong(2000),
	"optimes" : {
		"lastCommittedOpTime" : {
			"ts" : Timestamp(1527574904, 1),
			"t" : NumberLong(1)
		},
		"appliedOpTime" : {
			"ts" : Timestamp(1527574904, 1),
			"t" : NumberLong(2)
		},
		"durableOpTime" : {
			"ts" : Timestamp(1527574904, 1),
			"t" : NumberLong(2)
		}
	},
	"members" : [
		{
			"_id" : 0,
			"name" : "192.168.56.123:27017",
			"health" : 0,
			"state" : 8,
			"stateStr" : "(not reachable/healthy)",
			"uptime" : 0,
			"optime" : {
				"ts" : Timestamp(0, 0),
				"t" : NumberLong(-1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(0, 0),
				"t" : NumberLong(-1)
			},
			"optimeDate" : ISODate("1970-01-01T00:00:00Z"),
			"optimeDurableDate" : ISODate("1970-01-01T00:00:00Z"),
			"lastHeartbeat" : ISODate("2018-05-29T06:21:45.043Z"),
			"lastHeartbeatRecv" : ISODate("2018-05-29T06:19:03.338Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "Connection refused",
			"configVersion" : -1
		},
		{
			"_id" : 1,
			"name" : "192.168.56.128:27017",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 4539,
			"optime" : {
				"ts" : Timestamp(1527574904, 1),
				"t" : NumberLong(2)
			},
			"optimeDate" : ISODate("2018-05-29T06:21:44Z"),
			"electionTime" : Timestamp(1527574904, 2),
			"electionDate" : ISODate("2018-05-29T06:19:19Z"),
			"configVersion" : 1,
			"self" : true
		},
		{
			"_id" : 2,
			"name" : "192.168.56.123:27020",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
			"uptime" : 4111,
			"lastHeartbeat" : ISODate("2018-05-29T06:21:44.972Z"),
			"lastHeartbeatRecv" : ISODate("2018-05-29T06:21:45.200Z"),
			"pingMs" : NumberLong(0),
			"configVersion" : 1
		}
	],
	"ok" : 1
}
rep1:PRIMARY>
```

发现原Secondary节点（192.168.56.128:27017）已经变成Primary节点了。查询rs.status()发现，原Primary节点（192.168.56.123:27017）状态显示为“not reachable/healthy”。说明故障转移成功。

这时候再把192.168.56.123:27017启动，观察一下复制集状态：

```shell
[mongo@wmaster ~]$ ps -ef | grep mongod
mongo     2749     1  0 12:58 ?        00:00:30 mongod -f /usr/local/mongodb/conf/arbiter1.conf
mongo     3730  3185  0 14:27 pts/3    00:00:00 grep mongod
[mongo@wmaster ~]$ mongod -f /usr/local/mongodb/conf/primary27017.cnf
about to fork child process, waiting until server is ready for connections.
forked process: 3741
child process started successfully, parent exiting
[mongo@wmaster ~]$ ps -ef | grep mongod
mongo     2749     1  0 12:58 ?        00:00:30 mongod -f /usr/local/mongodb/conf/arbiter1.conf
mongo     3741     1  3 14:27 ?        00:00:00 mongod -f /usr/local/mongodb/conf/primary27017.cnf
mongo     3819  3185  0 14:28 pts/3    00:00:00 grep mongod
[mongo@wmaster ~]$ mongo -host 192.168.56.123 -port 27017
MongoDB shell version v3.4.9
connecting to: mongodb://192.168.56.123:27017/
MongoDB server version: 3.4.9
...
rep1:SECONDARY> rs.status()
{
	"set" : "rep1",
	"date" : ISODate("2018-05-29T06:29:34.653Z"),
	"myState" : 2,
	"term" : NumberLong(2),
	"syncingTo" : "192.168.56.128:27017",
	"heartbeatIntervalMillis" : NumberLong(2000),
	"optimes" : {
		"lastCommittedOpTime" : {
			"ts" : Timestamp(1527575384, 1),
			"t" : NumberLong(2)
		},
		"appliedOpTime" : {
			"ts" : Timestamp(1527575384, 1),
			"t" : NumberLong(2)
		},
		"durableOpTime" : {
			"ts" : Timestamp(1527575384, 1),
			"t" : NumberLong(2)
		}
	},
	"members" : [
		{
			"_id" : 0,
			"name" : "192.168.56.123:27017",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 100,
			"optime" : {
				"ts" : Timestamp(1527575384, 1),
				"t" : NumberLong(2)
			},
			"optimeDate" : ISODate("2018-05-29T06:29:04Z"),
			"syncingTo" : "192.168.56.128:27017",
			"configVersion" : 1,
			"self" : true
		},
		{
			"_id" : 1,
			"name" : "192.168.56.128:27017",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 100,
			"optime" : {
				"ts" : Timestamp(1527575384, 1),
				"t" : NumberLong(2)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1527575384, 1),
				"t" : NumberLong(2)
			},
			"optimeDate" : ISODate("2018-05-29T06:29:04Z"),
			"optimeDurableDate" : ISODate("2018-05-29T06:29:04Z"),
			"lastHeartbeat" : ISODate("2018-05-29T06:29:34.598Z"),
			"lastHeartbeatRecv" : ISODate("2018-05-29T06:29:32.831Z"),
			"pingMs" : NumberLong(0),
			"electionTime" : Timestamp(1528265959, 2),
			"electionDate" : ISODate("2018-05-29T06:19:19Z"),
			"configVersion" : 1
		},
		{
			"_id" : 2,
			"name" : "192.168.56.123:27020",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
			"uptime" : 100,
			"lastHeartbeat" : ISODate("2018-05-29T06:29:34.570Z"),
			"lastHeartbeatRecv" : ISODate("2018-05-29T06:29:30.094Z"),
			"pingMs" : NumberLong(0),
			"configVersion" : 1
		}
	],
	"ok" : 1
}
rep1:SECONDARY>
```

如上，192.168.56.123:27017重新回到了复制集中，并且状态已经变为Secondary。



![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com




