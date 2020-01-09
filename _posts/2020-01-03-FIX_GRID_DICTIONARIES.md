---
layout: post
title: Oracle集群和软件目录被误改权限之后的修复方法
date: 2020-01-03
categories: Oracle
tags: [Oracle]
description: Oracle集群和软件目录被误改权限之后的修复方法。
---

权限大了也不一定是好事，不懂的人稍微谨慎些的就不会乱搞，但是最怕似懂非懂的人，想当然的更改，往往是致命的。。
我们这就出现了这样的一个问题，公司模拟测试环境的oracle集群突然间其中一个节点连不上，平时这套环境用的人很少，而且业务是连接的scanip，所以好几天就挂了但是没人发现。直到我们环境运维童鞋登陆的时候才发现报错oracle不可用（因为运维童鞋连接的是其中一个节点的public ip），然后他尝试启动发现起不来，便来求助我。
遇到这个问题我感觉也很迷，ps查了一下进程发现oracle相关进程都没了，尝试启动失败报读不到文件，查看了一下，asm相关进程都在，使用grid用户使用sysasm进入准备看一下dg有没有问题，发现报TNS错误，查看了一下监听状态，发现出问题的节点的监听没有注册任何服务，另一个节点则是正常的，最直观感觉好像是监听的问题，看了一下集群日志和监听日志，发现grid在三天前的某一时刻就报找不到ocrdg，但是第一反应就是肯定有人动了什么，使用asmcmd命令发现居然报权限不足，查看了一下grid集群安装目录，好嘛，全变成了oracle属主的了，这肯定有人改动了。。。我追问了下负责环境运维的童鞋，在那天做什么操作了，他们说因为要做什么操作没权限，直接改了权限，而且最可怕的就是使用了-R选项。。。好吧，问题定位了，虽然低级失误很无语，但是问题已经出了，先修复吧。。。论dba的心酸-。-
对于这种情况，直接-R都改回grid用户肯定是不行的，熟悉数据库的童鞋都知道，oracle11g集群软件的权限是比较乱的，有些是root权限，有些是grid权限。不过还好，我们不是还有一个正常的节点么，那么参照另一个正常的节点将权限更改回来不就可以了么？
那么问题来了，怎么参照正常节点把有问题的节点的权限刷回来呢？

###方法一：使用集群验证工具cluvfy
```
cluvfy -help
USAGE:
cluvfy [ -help ]
cluvfy stage { -list | -help }
cluvfy stage {-pre|-post}  [-verbose]
cluvfy comp  { -list | -help }
cluvfy comp    [-verbose]
```
这里主要用到comp这个组件：
```
cluvfy comp -list
USAGE:
cluvfy comp    [-verbose]
有效组件为:
        nodereach : 检查各节点间的可访问性
        nodecon   : 检查节点的连接性
        cfs       : 检查 CFS 完整性
        ssa       : 检查共享存储的可访问性
        space     : 检查空间可用性
        sys       : 检查最小系统要求
        clu       : 检查集群完整性
        clumgr    : 检查集群管理器完整性
        ocr       : 检查 OCR 完整性
        olr       : 检查 OLR 完整性
        ha        : 检查 HA 完整性
        crs       : 检查 CRS 完整性
        nodeapp   : 检查节点应用程序是否存在
        admprv    : 检查管理权限
        peer      : 与对等端比较属性
        software  : 检查软件分发
        asm       : 检查 ASM 完整性
        acfs      : 检查 ACFS 完整性
        gpnp      : 检查 GPnP 完整性
        gns       : 检查 GNS 完整性
        scan      : 检查 SCAN 配置
        ohasd     : 检查 OHASD 完整性
        clocksync : 检查时钟同步
        vdisk      : 检查表决磁盘 Udev 设置
        
USAGE:
cluvfy comp software [-n <node_list>] [-d <oracle_home>] [-r {10gR1|10gR2|11gR1|11gR2}] [-verbose]

<node_list> is the comma separated list of non-domain qualified nodenames, on which the test should be conducted. If "all" is specified, then all the nodes in the cluster will be used for verification.
<oracle_home> is the location of the oracle home.
<10gR1|10gR2|11gR1|11gR2> is the release number of the product.

DESCRIPTION:
Checks the attributes of the files installed during the installation of Oracle Software. If no '-n' option is provided, the check is performed on the local node. By default files under Clusterware home are verified, if oracle home is specified by the option -d, files for the database home are verified instead.
```
cluvfy comp software -n all –verbose
这个命令如果一切正常的话会返回如下结果：
```
Verifying Software
Check: Software
  1178 files verified
Software check passed
Verification of software was successful.
```
如果失败的话会告诉你那些文件的权限不对，不过对于这种-R全改了，挨个全改回来会疯掉了。。。

###方法二：写一个shell脚本循环改
其实这种方法我当时没用，不过也是一个思路，现在简单写一下
```
#!/bin/bash
echo -n “Input Cluster install dictionary which was modified:”
read dir 
for i in $(find $dir)
do
owner=$(ls -ld |awk '{printf $3":"$4}')
echo "chown $owner $i">>chmod.sh
priv=$(stat “$i”|grep Uid|cut -c 10-13)
echo “chmod $priv $i”>>chmod.sh
done
```
在完好的节点上运行之后，再在出问题的节点上运行（但是两个节点并不是完全一致，这需自己vi批量替换下hostname、asm sid、oracle sid等，比如rac2为rac1、ASM2为ASM1、orcl2为orcl1）。
###方法三：使用getfacl、setfacl命令
这两个命令可以获得文件的acl，具体用法可以看man手册
首先，递归的获取完好节点上面文件和目录的权限
```
# getfacl -pR $GRID_BASE > priv_bk.txt
```
然后scp到出问题的节点上
```
# scp priv_bk.txt rac1:/home/grid
```
之后和上一步一样，用vi批量替换一下，替换完使用setfacl命令直接回复，因为两面的文件可能不会完全一样，有些文件可能没有的，不过没关系，没有更改报错会回显出来，可以单独处理一下就好，主体的目录结构的权限应该都更改回来了。
```
# setfacl –restore=/home/grid/priv_bk.txt
```
之后直接启动asm实例和数据库就好了，恢复正常～
最后可以在检查验证一下有没有没改过来的目录。

我当时就是使用第三种方式处理的，不过这三种方式都是只有在剩下一个节点是完好的情况下才能使用，都改了，估计就很麻烦了。所以说啊，平时一定要注意权限管控，还有就是别想当然的操作，方便一时，之后修起来还有点麻烦。。



![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com

