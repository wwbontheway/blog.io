---
layout: post
title: LVM扩容
date: 2017-06-06
categories: Linux
tags: [Linux,Diagnosis]
description: LVM扩容
---

一个客户打电话说做全备的备份失败，远程登陆发现原来是备份的目录空间不够了，不足以放下一个完整的全备，原来之前他用来备份的目录的磁盘出现问题，这个是临时加进来的一块。由于客户的磁盘是LVM管理的，这里面就用LVM命令进行扩容


远程登陆上来，查看一下空间，之前备份集大概需要500G左右，而新添加的磁盘挂载的目录为/bk03，只有不到400G，这肯定是不够的啊~

```shell

[root@otmdb2 ~]# df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/mpathap6   61G  7.9G   50G  14% /
tmpfs                  95G  361M   95G   1% /dev/shm
/dev/mapper/mpathap1  194M   34M  151M  18% /boot
/dev/mapper/mpathap5  9.7G  152M  9.0G   2% /opt
/dev/mapper/mpathap3   77G   20G   54G  27% /oracle
/dev/mapper/orabak-baklv
                      394G   80G  295G  22% /bk03
					 
```

在扩容之前要先将这个目录umount掉才可以，查看这个目录里面都被那些进程占用，和客户确认没有问题之后，手动kill掉进程。

```shell
					 
[root@otmdb2 ~]# lsof /bk03
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF    NODE NAME
bash     7784 oracle  cwd    DIR 253,15     4096 8126465 /bk03/backupset
rman     9484 oracle  cwd    DIR 253,15     4096 8126465 /bk03/backupset
bash     9798 oracle  cwd    DIR 253,15     4096       2 /bk03
tail    25557 oracle  cwd    DIR 253,15     4096       2 /bk03
tail    25557 oracle    3r   REG 253,15    13985      27 /bk03/rman_restore.log
[root@otmdb2 ~]# kill -9 7784
[root@otmdb2 ~]# kill -9 9484
[root@otmdb2 ~]# kill -9 9798
[root@otmdb2 ~]# kill -9 25557

````

再次查看，发现没有进程占用目录/bk03，将目录umount

```shell

[root@otmdb2 ~]# lsof /bk03
[root@otmdb2 ~]# umount /bk03

```

给对应的lv扩展400G

```shell

[root@otmdb2 ~]# lvextend -L 400G /dev/mapper/orabak-baklv
  New size (102400 extents) matches existing size (102400 extents)
  Run `lvextend --help' for more information.
[root@otmdb2 ~]# lvextend -L 800G /dev/mapper/orabak-baklv
  Extending logical volume baklv to 800.00 GiB
  Logical volume baklv successfully resized
[root@otmdb2 ~]# resize2fs /dev/mapper/orabak-baklv
resize2fs 1.41.12 (17-May-2010)
Please run 'e2fsck -f /dev/mapper/orabak-baklv' first.

[root@otmdb2 ~]# e2fsck -f /dev/mapper/orabak-baklv
e2fsck 1.41.12 (17-May-2010)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/mapper/orabak-baklv: 51/26214400 files (5.9% non-contiguous), 22422336/104857600 blocks
[root@otmdb2 ~]# resize2fs /dev/mapper/orabak-baklv
resize2fs 1.41.12 (17-May-2010)
Resizing the filesystem on /dev/mapper/orabak-baklv to 209715200 (4k) blocks.
The filesystem on /dev/mapper/orabak-baklv is now 209715200 blocks long.

```

重新挂载目录，查看目录大小

```shell

[root@otmdb2 ~]# mount /dev/mapper/orabak-baklv /bk03
[root@otmdb2 ~]# df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/mpathap6   61G  7.9G   50G  14% /
tmpfs                  95G  361M   95G   1% /dev/shm
/dev/mapper/mpathap1  194M   34M  151M  18% /boot
/dev/mapper/mpathap5  9.7G  152M  9.0G   2% /opt
/dev/mapper/mpathap3   77G   20G   54G  27% /oracle
/dev/mapper/orabak-baklv
                      788G   80G  669G  11% /bk03
[root@otmdb2 ~]# cd /bk03

```

如上，/bk03已经扩展到788G


![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com




