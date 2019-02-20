---
layout: post
title: MongoDB复制集同步问题处理-修改oplog大小
date: 2019-02-20
categories: MongoDB
tags: [MongoDB]
description: MongoDB复制集同步问题处理。
---


比较尴尬的事情总会发生，新搭建的一套分片+复制集因为oom被os干掉了-。-
被干掉的是一个分片中的复制集的一个成员，另一台secondary节点自动升级为primary。
先将其中被干掉的分片启动：
```
[mongo@mongodb3 mongoconf]$ mongod -f /home/mongo/mongoconf/shard1_3.cnf
[mongo@mongodb3 mongoconf]$ ps -ef | grep mongoc
mongo     4523     1  6 Jan11 ?        2-19:10:48 mongod -f /home/mongo/mongoconf/shard1_3.cnf
mongo     4725     1  0 Jan11 ?        04:20:21 mongod -f /home/mongo/mongoconf/configsvr3.cnf
mongo     4838     1  0 Jan11 ?        01:20:28 mongos -f /home/mongo/mongoconf/mongos.cnf
mongo    24850     1 26 Feb18 ?        14:27:37 mongod -f /home/mongo/mongoconf/shard2_3.cnf
mongo    27181 23019  0 17:52 pts/0    00:00:00 grep --color=auto mongoc

```
查看进程已经启动了，进入到这个节点看一下状态：


```
mongoshard1:RECOVERING> 
```
发现是recovering状态，查看rs.status()和这个片成员的log发现，由于时间比较久，已经找不到可以同步的成员
```
2019-02-20T10:35:32.394+0800 I REPL     [replication-0] We are too stale to use 10.88.190.30:28017 as a sync source. Blacklisting this sync source because our last fetched timestamp: Timestamp(1550576222, 254) is before their earliest timestamp: Timestamp(1550629961, 1148) for 1min until: 2019-02-20T10:36:32.394+0800
2019-02-20T10:35:32.394+0800 I REPL     [replication-0] sync source candidate: 10.88.190.20:28017
2019-02-20T10:35:32.394+0800 I ASIO     [RS] Connecting to 10.88.190.20:28017
2019-02-20T10:35:32.395+0800 I REPL     [replication-0] We are too stale to use 10.88.190.20:28017 as a sync source. Blacklisting this sync source because our last fetched timestamp: Timestamp(1550576222, 254) is before their earliest timestamp: Timestamp(1550629971, 3358) for 1min until: 2019-02-20T10:36:32.395+0800
2019-02-20T10:35:32.395+0800 I REPL     [replication-0] could not find member to sync from

```

由于数据量比较小，将出现问题的分片成员的数据目录清空，重新启动进行数据初始化同步。查看状态看到是STARTUP2，说明正在同步数据。通过观察数据目录的增长也可以看到

```
mongoshard1:STARTUP2> rs.status()
{
	"set" : "mongoshard1",
	"date" : ISODate("2019-02-20T09:57:39.359Z"),
	"myState" : 5,
	"term" : NumberLong(2),
	"syncingTo" : "10.88.190.30:28017",
	"syncSourceHost" : "10.88.190.30:28017",
	"syncSourceId" : 2,
	"heartbeatIntervalMillis" : NumberLong(2000),
	"optimes" : {
		"lastCommittedOpTime" : {
			"ts" : Timestamp(1550656659, 391),
			"t" : NumberLong(2)
		},
		"appliedOpTime" : {
			"ts" : Timestamp(0, 0),
			"t" : NumberLong(-1)
		},
		"durableOpTime" : {
			"ts" : Timestamp(0, 0),
			"t" : NumberLong(-1)
		}
	},
	"lastStableCheckpointTimestamp" : Timestamp(0, 0),
	"members" : [
		{
			"_id" : 0,
			"name" : "10.88.190.10:28017",
			"health" : 1,
			"state" : 5,
			"stateStr" : "STARTUP2",
			"uptime" : 15007,
			"optime" : {
				"ts" : Timestamp(0, 0),
				"t" : NumberLong(-1)
			},
			"optimeDate" : ISODate("1970-01-01T00:00:00Z"),
			"syncingTo" : "10.88.190.30:28017",
			"syncSourceHost" : "10.88.190.30:28017",
			"syncSourceId" : 2,
			"infoMessage" : "",
			"configVersion" : 1,
			"self" : true,
			"lastHeartbeatMessage" : ""
		},
		{
			"_id" : 1,
			"name" : "10.88.190.20:28017",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 15006,
			"optime" : {
				"ts" : Timestamp(1550656659, 391),
				"t" : NumberLong(2)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1550656659, 391),
				"t" : NumberLong(2)
			},
			"optimeDate" : ISODate("2019-02-20T09:57:39Z"),
			"optimeDurableDate" : ISODate("2019-02-20T09:57:39Z"),
			"lastHeartbeat" : ISODate("2019-02-20T09:57:39.160Z"),
			"lastHeartbeatRecv" : ISODate("2019-02-20T09:57:37.936Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncingTo" : "10.88.190.30:28017",
			"syncSourceHost" : "10.88.190.30:28017",
			"syncSourceId" : 2,
			"infoMessage" : "",
			"configVersion" : 1
		},
		{
			"_id" : 2,
			"name" : "10.88.190.30:28017",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 15006,
			"optime" : {
				"ts" : Timestamp(1550656659, 391),
				"t" : NumberLong(2)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1550656659, 391),
				"t" : NumberLong(2)
			},
			"optimeDate" : ISODate("2019-02-20T09:57:39Z"),
			"optimeDurableDate" : ISODate("2019-02-20T09:57:39Z"),
			"lastHeartbeat" : ISODate("2019-02-20T09:57:39.169Z"),
			"lastHeartbeatRecv" : ISODate("2019-02-20T09:57:37.362Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncingTo" : "",
			"syncSourceHost" : "",
			"syncSourceId" : -1,
			"infoMessage" : "",
			"electionTime" : Timestamp(1550576233, 1007),
			"electionDate" : ISODate("2019-02-19T11:37:13Z"),
			"configVersion" : 1
		}
	],
	"ok" : 1,
	"operationTime" : Timestamp(0, 0),
	"$gleStats" : {
		"lastOpTime" : Timestamp(0, 0),
		"electionId" : ObjectId("000000000000000000000000")
	},
	"lastCommittedOpTime" : Timestamp(1550656659, 391),
	"$configServerState" : {
		"opTime" : {
			"ts" : Timestamp(1550656629, 256),
			"t" : NumberLong(1)
		}
	},
	"$clusterTime" : {
		"clusterTime" : Timestamp(1550656659, 474),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	}
}

```

在即将完成的时候依然出现recovering。检查一下发现是之前配置的oplog已经不足以支撑当前的数据更新速度，简单的说就是备库接收到了sync复制信息，但是因为断档时间太久了，导致sync失败。需要将oplog增大并再次人工同步。
查看当前的oplog大小：

```
mongoshard1:PRIMARY> rs.printReplicationInfo()
configured oplog size:   256MB
log length start to end: 204secs (0.06hrs)
oplog first event time:  Wed Feb 20 2019 16:25:04 GMT+0800 (CST)
oplog last event time:   Wed Feb 20 2019 16:28:28 GMT+0800 (CST)
now:                     Wed Feb 20 2019 16:28:28 GMT+0800 (CST)

```
在之前是没有问题的，但是最近迁移过来的数据多了，很多项目都连接MongoDB查询数据，所以压力上来了，现在复制集再出现问题，oplog的大小显得有些小了。需要进行加大。
不过还好我的MongoDB版本是4.0.4，可以在线更改oplog的大小。命令如下：
```
mongoshard1:PRIMARY> db.adminCommand({replSetResizeOplog: 1, size: 2000})
{
	"ok" : 1,
	"operationTime" : Timestamp(1550646300, 1575),
	"$gleStats" : {
		"lastOpTime" : Timestamp(0, 0),
		"electionId" : ObjectId("7fffffff0000000000000003")
	},
	"lastCommittedOpTime" : Timestamp(1550646300, 1499),
	"$configServerState" : {
		"opTime" : {
			"ts" : Timestamp(1550646300, 1592),
			"t" : NumberLong(1)
		}
	},
	"$clusterTime" : {
		"clusterTime" : Timestamp(1550646300, 1598),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	}
}
mongoshard1:PRIMARY> rs.printReplicationInfo()
configured oplog size:   2000MB
log length start to end: 154secs (0.04hrs)
oplog first event time:  Wed Feb 20 2019 15:02:29 GMT+0800 (CST)
oplog last event time:   Wed Feb 20 2019 15:05:03 GMT+0800 (CST)
now:                     Wed Feb 20 2019 15:05:03 GMT+0800 (CST)

```
这里我将oplog的大小调整为2g，需要注意的是这个命令的单位是M，还有一点就是对于复制集来说，要先执行secondary，然后最后在执行primary。更多关于这个命令的信息MongoDB的官方文档有，这里附上连接：
https://docs.mongodb.com/manual/tutorial/change-oplog-size/#oplog

更改之后重新同步即可。






![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com
