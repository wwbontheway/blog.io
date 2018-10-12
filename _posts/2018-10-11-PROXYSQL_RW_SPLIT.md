---
layout: post
title: ProxySQL实现MySQL主从的读写分离
date: 2018-10-11
categories: ProxySQL
tags: [ProxySQL]
description: ProxySQL实现MySQL主从的读写分离。
---

测试了一下ProxySQL实现MySQL主从的读写分离，这里面说明一下，我的主从是云到本地的环境。另外个人看完ProxySQL之后，感觉这个中间件功能确实很好用，相对于mycat那种xml的规则配置，这种类似数据库的操作的规则配置令我这个做了多年数据库的人感到很舒服，嗯嗯~

ProxySQL下载地址：
```
https://github.com/sysown/proxysql/releases
https://www.percona.com/downloads/proxysql
```

注：文中涉及信息,约定如下：

|约定条目|约定信息|
|:--|:--|
|主库地址|masterhost.mysql.rds.aliyuncs.com:3306|
|从库地址|172.16.10.10:3309|
|ProxySQL地址|proxysqlhost|
|密码信息|Password!123|


### 下载安装ProxySQL

注：主从搭建步骤略，可以参考之前的博文，简单介绍一下环境，
主节点：阿里云rds平台，MySQL 5.6.16
从节点：CentOS 7，MySQL 5.6.16，主从已同步。

```sql
[root@proxysqlhost ~]# wget -c 'https://www.percona.com/downloads/proxysql/proxysql-1.4.10/binary/redhat/7/x86_64/proxysql-1.4.10-1.1.el7.x86_64.rpm' -O proxysql-1.4.10-1.1.el7.x86_64.rpm &

[root@proxysqlhost ~]# rpm -xvh proxysql-1.4.10-1.1.el7.x86_64.rpm

[root@proxysqlhost ~]# service proxysql start

[root@proxysqlhost ~]# proxysql --version
ProxySQL version 1.4.10-percona-1.1, codename Truls

```

### 配置ProxySQL

#### 登陆ProxySQL

注意：ProxySQL的管理端口是6032，服务端口是6033。如果感觉不好记的话，想想MySQL的默认端口号是多少，发现规律了没，嘿嘿~

```sql
[root@proxysqlhost ~]# mysql -u admin -padmin -h 127.0.0.1 -P6032

MySQL [(none)]> \R ProxysqlAdmin>
PROMPT set to 'ProxysqlAdmin>'
ProxysqlAdmin> show databases;
+-----+---------------+-------------------------------------+
| seq | name          | file                                |
+-----+---------------+-------------------------------------+
| 0   | main          |                                     |
| 2   | disk          | /var/lib/proxysql/proxysql.db       |
| 3   | stats         |                                     |
| 4   | monitor       |                                     |
| 5   | stats_history | /var/lib/proxysql/proxysql_stats.db |
+-----+---------------+-------------------------------------+
5 rows in set (0.00 sec)
```

需要保证目前没有配置，即mysql_servers，mysql_replication_hostgroups，mysql_query_rules这三张表是空的：
```sql
ProxysqlAdmin>select * from mysql_servers;
Empty set (0.00 sec)

ProxysqlAdmin>select * from mysql_replication_hostgroups;
Empty set (0.00 sec)

ProxysqlAdmin>select * from mysql_query_rules;
Empty set (0.00 sec)
```

#### 配置MySQL

注意这里hostgroup_id值0为写，1为读（可以随意自定义的，虽然可以在后面通过配置ProxySQL来检测read_only值来自动分组，不过建议还是直接手动规划的好）：

```sql
ProxysqlAdmin>INSERT INTO mysql_servers(hostgroup_id,hostname,port,comment) VALUES (0,'masterhost.mysql.rds.aliyuncs.com',3306,'writehost');
ProxysqlAdmin>INSERT INTO mysql_servers(hostgroup_id,hostname,port,comment) VALUES (1,'172.16.10.10',3309,'readhost');

```

上面配置相当于在memory层进行了改变，要是即刻生效，需要将配置加载到runtime层，想将配置持久化保存，需要保存到disk层。具体的ProxySQL的三层结构，可以看看官方文档的介绍，这里面不在赘述:

```sql
ProxysqlAdmin>LOAD MYSQL SERVERS TO RUNTIME;
ProxysqlAdmin>SAVE MYSQL SERVERS TO DISK;
```

#### 配置用户
在后端MySQL服务器中创建监控用户(psqlmon)和应用用户(produser)：
注意：由于使用阿里云作为主库搭建的主从复制，mysql数据库是不复制的。所以需要在主从分别建立相应用户。

```sql
grant select,process on *.* to psqlmon@'proxysqlhost' identified by 'Password!123';
GRANT select,insert,update,delete ON *.* TO  produser@'proxysqlhost' identified by 'Password!123';
flush privileges;
```

在ProxySQL中添加监控用户：

```sql
ProxysqlAdmin>UPDATE global_variables SET variable_value='psqlmon' WHERE variable_name='mysql-monitor_username';
ProxysqlAdmin>UPDATE global_variables SET variable_value='Password!123' WHERE variable_name='mysql-monitor_password';
```
也可以：
```sql
ProxysqlAdmin>SET mysql-monitor_username='psqlmon';
ProxysqlAdmin>SET mysql-monitor_password='Password!123';
```

#### 配置其他全局变量

```sql
ProxysqlAdmin>set mysql-default_charset='utf';
ProxysqlAdmin>set mysql-server_version='5.6.16';
```

即时生效配置并持久化：
```sql
ProxysqlAdmin>LOAD MYSQL VARIABLES TO RUN;
ProxysqlAdmin>SAVE MYSQL VARIABLES TO DISK;
```

#### ProxySQL中配置读写组

0为写组，1为读组，号码不唯一，可以随意定制：

```sql
ProxysqlAdmin>INSERT INTO mysql_replication_hostgroups VALUES (0,1,'mysql_rw_split');
```

在ProxySQL中添加MySQL应用用户：

注：这里面默认组设定为0组

```sql
ProxysqlAdmin>INSERT INTO  mysql_users(username,password,active,default_hostgroup,transaction_persistent) VALUES('produser','Password!123',1,0,1);
```

#### 配置路由规则

首先确认规则表是空的：
```sql
ProxysqlAdmin>select * from mysql_query_rules;
Empty set (0.00 sec)
```
添加查询规则，除了”select...for update“之外的select语句，都路由到1组。

小技巧：rule_id是按照顺序生效的，建议不要紧挨着号码，这样未来有需要提到前面的规则，可以直接插到中间

```sql
ProxysqlAdmin>INSERT INTO mysql_query_rules (rule_id,active,match_digest,destination_hostgroup,apply)
VALUES (11,1,'^SELECT.*FOR UPDATE$',0,1),(21,1,'^SELECT',1,1);

```

即时生效配置并持久化：
```sql
ProxysqlAdmin>LOAD MYSQL QUERY RULES TO RUN;
ProxysqlAdmin>LOAD MYSQL SERVERS TO RUN;
ProxysqlAdmin>LOAD MYSQL USERS TO RUN;
ProxysqlAdmin>LOAD MYSQL VARIABLES TO RUN;
ProxysqlAdmin>SAVE MYSQL QUERY RULES TO DISK;
ProxysqlAdmin>SAVE MYSQL SERVERS TO DISK;
ProxysqlAdmin>SAVE MYSQL USERS TO DISK;
ProxysqlAdmin>SAVE MYSQL VARIABLES TO DISK; 
```



#### 应用连接数据库
```sql
[root@proxysqlhost ~]# mysql -uproduser -p -hproxysqlhost -P6033
```
连接之后随便执行一些SQL语句，测试一下。

### 监控

ProxySQL中监控服务器组的状态：
登陆管理账户，查看表：runtime_mysql_servers

```sql
ProxysqlAdmin>select  hostgroup,schemaname,username,digest_text,count_star from stats_mysql_query_digest order by last_seen desc;
+-----------+--------------------+----------+-----------------------------------+------------+
| hostgroup | schemaname         | username | digest_text                       | count_star |
+-----------+--------------------+----------+-----------------------------------+------------+
| 0         | information_schema | produser | select @@version_comment limit ?  | 3          |
| 1         | test               | produser | select * from testw01 limit ?     | 1          |
| 1         | test               | produser | select * from testw01             | 2          |
| 0         | test               | produser | select * from testw01 for update  | 1          |
| 0         | test               | produser | insert into testw01 values(?)     | 1          |
| 0         | test               | produser | show create table testw01         | 1          |
| 0         | test               | produser | create table wtab1(a int)         | 1          |
| 0         | test               | produser | show tables                       | 2          |
| 1         | information_schema | produser | SELECT DATABASE()                 | 1          |
| 0         | test               | produser | show databases                    | 1          |
| 1         | information_schema | produser | select * from wdb1.testw03 limit ?| 1          |
| 0         | information_schema | produser | show grants                       | 2          |
| 0         | information_schema | produser | show databases                    | 3          |
| 1         | information_schema | produser | select ?                          | 2          |
+-----------+--------------------+----------+-----------------------------------+------------+
```


可见，除了普通的select被路由到1组，其余的操作都路由到0组。
至此，ProxySQL实现了MySQL读写分离。文中涉及的表、变量等，都可以参考ProxySQL官方文档进行详细查询。





![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com


