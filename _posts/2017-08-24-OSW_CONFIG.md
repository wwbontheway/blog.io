---
layout: post
title: OSW配置
date: 2017-08-24
categories: Linux
tags: [Linux]
description: 从连不上数据库到更换磁盘。
---

简单介绍一下OSW的配置，这个工具很好用，可以后期分析故障的时候，获得主机在故障时段的信息。这个工具是官方强烈建议安装配置的，在Oracle Exadata上面，叫做exawatcher，用法一样，而且是自带的。可见Oracle官方对这个软件及其捕获的信息的关注的重要程度。在某些时候，给Oracle售后提SR的时候，也会被索要，小w建议尽量都配置，OSW本身消耗的资源微乎其微，真是小成本大收益，嘿嘿~

那么下面简述一下配置方式


### 下载oswatcher，上传到服务器

```shell

[root@ora11g opt]# ls
ORCLfmap  oswbb801.tar
[root@ora11g opt]# tar xvf oswbb801.tar 

--解压输出略

[root@ora11g opt]# ls -l
total 4772
drwxr-xr-x 3 root   root        4096 Jun 13 20:33 ORCLfmap
drwx------ 4 oracle oinstall    4096 May 17 22:25 oswbb
-rw-r--r-- 1 root   root     4864000 Aug 24 10:27 oswbb801.tar
[root@ora11g opt]# id oracle
uid=502(oracle) gid=502(dba) groups=502(dba)
[root@ora11g opt]# chown -R oracle:dba oswbb
[root@ora11g opt]# chmod 775 oswbb

```

配置好了之后，可以加入到crontab中，也可以写进/etc/rc.local中开机自动运行

### 配置示例

每30s采样一次，采样持续7天（24\*7=168），不进行压缩，文件存储在/opt/oswbb/archive下

```shell

su - oracle -c "cd /opt/oswbb && nohup ./startOSWbb.sh 30 168 NONE /opt/oswbb/archive &"

```

### 分析

首先先后台运行osw：

```shell

# nohup ./startOSWbb.sh 10 168 NONE /opt/oswbb/archive &

```

查看进程

```shell

[root@ora11g oswbb]# ps -ef | grep -i osw
root      8102     1  0 15:44 pts/2    00:00:00 /bin/sh ./OSWatcher.sh 10 168 NONE /opt/oswbb/archive
root      8171  8102  0 15:44 pts/2    00:00:00 /bin/sh ./OSWatcherFM.sh 168 /opt/oswbb/archive
root      8751  7981  0 15:45 pts/2    00:00:00 grep -i osw

```

先出现OSWatcher.sh这个进程，随后它会调用OSWatcherFM.sh


查看nohup.out输出

```shell

[root@ora11g oswbb]# vi nohup.out 


Testing for discovery of OS Utilities...
VMSTAT found on your system.
IOSTAT found on your system.
MPSTAT found on your system.
IFCONFIG found on your system.
NETSTAT found on your system.
TOP found on your system.
TRACEROUTE found on your system.

Discovery of CPU CORE COUNT
CPU CORE COUNT will be used by oswbba to automatically look for cpu problems


Warning... CPU CORE COUNT not found on your system.


Defaulting to CPU CORE COUNT = 1
To correctly specify CPU CORE COUNT
1. Correct the error listed above for your unix platform or
2. Manually set core_count on OSWatcher.sh line 16 or
3. Do nothing and accept default value = 1

Discovery completed.

Starting OSWatcher v8.0.1  on Thu Aug 24 15:44:36 CST 2017
With SnapshotInterval = 10
With ArchiveInterval = 168

OSWatcher - Written by Carl Davis, Center of Expertise,
Oracle Corporation
For questions on install/usage please go to MOS (Note:301137.1)
If you need further assistance or have comments or enhancement
requests you can email me Carl.Davis@Oracle.com

Data is stored in directory: /opt/oswbb/archive

Starting Data Collection...

oswbb heartbeat:Thu Aug 24 15:44:41 CST 2017
oswbb heartbeat:Thu Aug 24 15:44:51 CST 2017
oswbb heartbeat:Thu Aug 24 15:45:01 CST 2017

```

查看对应目录，已经生成记录文件：

```shell

[oracle@ora11g oswbb]$ cd archive/
[oracle@ora11g archive]$ ls -lh
total 40K
drwxr-xr-x 2 root root 4.0K Aug 24 17:00 oswifconfig
drwxr-xr-x 2 root root 4.0K Aug 24 17:00 oswiostat
drwxr-xr-x 2 root root 4.0K Aug 24 17:00 oswmeminfo
drwxr-xr-x 2 root root 4.0K Aug 24 17:00 oswmpstat
drwxr-xr-x 2 root root 4.0K Aug 24 17:00 oswnetstat
drwxr-xr-x 2 root root 4.0K Aug 24 15:41 oswprvtnet
drwxr-xr-x 2 root root 4.0K Aug 24 17:00 oswps
drwxr-xr-x 2 root root 4.0K Aug 24 17:00 oswslabinfo
drwxr-xr-x 2 root root 4.0K Aug 24 17:00 oswtop
drwxr-xr-x 2 root root 4.0K Aug 24 17:00 oswvmstat
[oracle@ora11g archive]$ tree .
.
|-- oswifconfig
|   |-- ora11g_ifconfig_17.08.24.1500.dat
|   |-- ora11g_ifconfig_17.08.24.1600.dat
|   `-- ora11g_ifconfig_17.08.24.1700.dat
|-- oswiostat
|   |-- ora11g_iostat_17.08.24.1500.dat
|   |-- ora11g_iostat_17.08.24.1600.dat
|   `-- ora11g_iostat_17.08.24.1700.dat
|-- oswmeminfo
|   |-- ora11g_meminfo_17.08.24.1500.dat
|   |-- ora11g_meminfo_17.08.24.1600.dat
|   `-- ora11g_meminfo_17.08.24.1700.dat
|-- oswmpstat
|   |-- ora11g_mpstat_17.08.24.1500.dat
|   |-- ora11g_mpstat_17.08.24.1600.dat
|   `-- ora11g_mpstat_17.08.24.1700.dat
|-- oswnetstat
|   |-- ora11g_netstat_17.08.24.1500.dat
|   |-- ora11g_netstat_17.08.24.1600.dat
|   `-- ora11g_netstat_17.08.24.1700.dat
|-- oswprvtnet
|-- oswps
|   |-- ora11g_ps_17.08.24.1500.dat
|   |-- ora11g_ps_17.08.24.1600.dat
|   `-- ora11g_ps_17.08.24.1700.dat
|-- oswslabinfo
|   |-- ora11g_slabinfo_17.08.24.1500.dat
|   |-- ora11g_slabinfo_17.08.24.1600.dat
|   `-- ora11g_slabinfo_17.08.24.1700.dat
|-- oswtop
|   |-- ora11g_top_17.08.24.1500.dat
|   |-- ora11g_top_17.08.24.1600.dat
|   `-- ora11g_top_17.08.24.1700.dat
`-- oswvmstat
    |-- ora11g_vmstat_17.08.24.1500.dat
    |-- ora11g_vmstat_17.08.24.1600.dat
    `-- ora11g_vmstat_17.08.24.1700.dat

10 directories, 27 files

```

### 生成分析图表

如果用oracle用户进行分析，需要注意archive目录下的文件权限。需要先加java环境变量到oracle的bash_profile中

```shell

[oracle@ora11g archive]$ cat ~/.bash_profile
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs

PATH=$PATH:$HOME/bin

export PATH
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/dbhome_1
export ORACLE_SID=ora11g
export PATH=$ORACLE_HOME/bin:$ORACLE_HOME/jdk/bin:$PATH:$HOME/bin
export NLS_LANG=AMERICAN_AMERICA.ZHS16GBK
export LD_LIBRARY_PATH=$ORACLE_HOME/lib32:$ORACLE_HOME/lib:$LD_LIBRARY_PATH

```

查看java

```shell

[oracle@ora11g archive]$ java -version
java version "1.5.0_30"
Java(TM) 2 Runtime Environment, Standard Edition (build 1.5.0_30-b03)
Java HotSpot(TM) Client VM (build 1.5.0_30-b03, mixed mode)

```

调用oswbba.jar进行分析
这里使用root进行分析

```shell

java -jar -Xmx128M /opt/oswbb/oswbba.jar -i /opt/oswbb/archive -b  Aug 24 15:45:00 2017 -e  Aug 24 15:50:00 2017
[root@ora11g oswbb]# java -jar -Xmx128M /opt/oswbb/oswbba.jar -i /opt/oswbb/archive -b  Aug 24 15:55:00 2017 -e  Aug 24 16:05:00 2017

Validating times in the archive...


Starting OSW Analyzer V8.0.0
OSWatcher Analyzer Written by Oracle Center of Expertise
Copyright (c)  2017 by Oracle Corporation

Parsing Data. Please Wait...

Scanning file headers for version and platform info...

Parsing file ora11g_iostat_17.08.24.1500.dat ...
Parsing file ora11g_iostat_17.08.24.1600.dat ...
This directory already exists. Rewriting...

Parsing file ora11g_vmstat_17.08.24.1500.dat ...
Parsing file ora11g_vmstat_17.08.24.1600.dat ...


Parsing file ora11g_netstat_17.08.24.1500.dat ...
Thu Aug 24 15:41:55 CST 2017 rejected

--中间部分略

Thu Aug 24 16:59:51 CST 2017 rejected

Parsing file ora11g_top_17.08.24.1500.dat ...
Parsing file ora11g_top_17.08.24.1600.dat ...

Parsing file ora11g_ps_17.08.24.1500.dat ...
Parsing file ora11g_ps_17.08.24.1600.dat ...


Parsing Completed.


Enter 1 to Display CPU Process Queue Graphs
Enter 2 to Display CPU Utilization Graphs
Enter 3 to Display CPU Other Graphs
Enter 4 to Display Memory Graphs
Enter 5 to Display Disk IO Graphs

Enter GC to Generate All CPU Gif Files
Enter GM to Generate All Memory Gif Files
Enter GD to Generate All Disk Gif Files
Enter GN to Generate All Network Gif Files

Enter A to Analyze Data
Enter S to Analyze Subset of Data(Changes analysis dataset including graph time scale)
Enter D to Generate DashBoard

Enter L to Specify Alternate Location of Gif Directory
Enter T to Alter Graph Time Scale Only (Does not change analysis dataset)
Enter B to Return to Default Baseline Graph Time Scale
Enter R to Remove Currently Displayed Graphs
Enter X to Export Parsed Data to File
Enter Q to Quit Program

Please Select an Option:

--输入对应选项，这里选择D生产图表

Please Select an Option:D
Enter a unique analysis/dashBoard name or enter <CR> to accept default name:

A new analysis file analysis/ora11g_Aug24155455_1503567002/analysis.txt has been created.

Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Run_Queue.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Run_Adjusted_Queue.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Block_Queue.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_HB.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_PS_Processes.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Cpu_Idle.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Cpu_System.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Cpu_User.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Cpu_Wa.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Cpu_Interrupts.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Context_Switches.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Memory_Swap.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Memory_Free.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Memory_Page_In_Rate.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Memory_Page_Out_Rate.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Cpu_Wa.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_Block_Queue.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_IO_ST.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_IO_AW.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_IO_PB.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_IO_RPS.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_IO_WPS.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_OS_IO_TPS.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_lo_rx_ok.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_lo_rx_err.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_lo_rx_drp.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_lo_rx_ovr.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_lo_tx_ok.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_lo_tx_err.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_lo_tx_drp.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_lo_tx_ovr.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_eth0_rx_ok.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_eth0_rx_err.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_eth0_rx_drp.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_eth0_rx_ovr.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_eth0_tx_ok.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_eth0_tx_err.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_eth0_tx_drp.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_link_eth0_tx_ovr.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_ip_requests_sent_out.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_ip_total_packets_received.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_ip_bad_header_checksum.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_ip_fragments_dropped.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_ip_fragments_created.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_ip_fragments_received.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_ip_fragments_dropped_after.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_ip_fragments_warn1.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_ip_fragments_warn2.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_udp_datagrams_in.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_udp_datagrams_out.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_udp_dropped.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_udp_broadcast_dropped.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_udp_socket_overflows.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_udp_bad_header_checksums.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_tcp_in_segs.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_tcp_out_segs.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_tcp_retrans_segs.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_tcp_conn_resets_received_segs.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_tcp_resets_sent.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_tcp_failed_conn_attempts.jpg
Generating file analysis/ora11g_Aug24155455_1503567002/dashboard/generated_files/OSWg_tcp_retran_error_rate.jpg

Enter 1 to Display CPU Process Queue Graphs
Enter 2 to Display CPU Utilization Graphs
Enter 3 to Display CPU Other Graphs
Enter 4 to Display Memory Graphs
Enter 5 to Display Disk IO Graphs

Enter GC to Generate All CPU Gif Files
Enter GM to Generate All Memory Gif Files
Enter GD to Generate All Disk Gif Files
Enter GN to Generate All Network Gif Files

Enter A to Analyze Data
Enter S to Analyze Subset of Data(Changes analysis dataset including graph time scale)
Enter D to Generate DashBoard

Enter L to Specify Alternate Location of Gif Directory
Enter T to Alter Graph Time Scale Only (Does not change analysis dataset)
Enter B to Return to Default Baseline Graph Time Scale
Enter R to Remove Currently Displayed Graphs
Enter X to Export Parsed Data to File
Enter Q to Quit Program

Please Select an Option:q

```

已经生成成功，打成tar包导出来查看

```shell

[root@ora11g oswbb]# cd analysis/
[root@ora11g analysis]# ll
total 4
drwxr-xr-x 3 root root 4096 Aug 24 17:42 ora11g_Aug24155455_1503567769
[root@ora11g analysis]# 
[root@ora11g analysis]# zip -r ora11.zip ora11g_Aug24155455_1503567769
  
--输出过程略
  
[root@ora11g analysis]# 
[root@ora11g analysis]# pwd
/opt/oswbb/analysis

```

传到windows下解压，运行index.html，即可查看各种指标~




![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com



