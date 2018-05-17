---
layout: post
title: Linux下python3的安装
date: 2016-11-13
categories: Python
tags: [Python]
description: python3安装。
---

随笔记录一下~

默认RHEL 上的python 版本是2.x，如果想安装python 3，需要单独下载。

下载地址：https://www.python.org/ftp/python/3.5.0/Python-3.5.0b4.tgz

```shell

[root@wmysql Python-3.5.0b4]# tar zxvf Python-3.5.0b4.tgz
[root@wmysql python_work]# ls -lthr
total 20M
drwxrwxr-x. 16 1000 1000 4.0K Jul 26  2015 Python-3.5.0b4
-rw-r--r--.  1 root root  20M Oct 22 18:48 Python-3.5.0b4.tgz
[root@wmysql python_work]# cd Python-3.5.0b4
[root@wmysql Python-3.5.0b4]# ls -ltrh
total 988K
-rw-r--r--.  1 1000 1000  13K Jul 26  2015 LICENSE
drwxrwxr-x.  2 1000 1000 4.0K Jul 26  2015 Include
drwxrwxr-x.  2 1000 1000 4.0K Jul 26  2015 Grammar
drwxrwxr-x.  2 1000 1000 4.0K Jul 26  2015 Misc
-rw-r--r--.  1 1000 1000  56K Jul 26  2015 Makefile.pre.in
drwxrwxr-x.  8 1000 1000 4.0K Jul 26  2015 Mac
drwxrwxr-x. 46 1000 1000  12K Jul 26  2015 Lib
drwxrwxr-x. 22 1000 1000 4.0K Jul 26  2015 Tools
-rw-r--r--.  1 1000 1000  96K Jul 26  2015 setup.py
-rw-r--r--.  1 1000 1000 6.7K Jul 26  2015 README
drwxrwxr-x.  3 1000 1000 4.0K Jul 26  2015 Python
-rw-r--r--.  1 1000 1000  41K Jul 26  2015 pyconfig.h.in
drwxrwxr-x.  2 1000 1000 4.0K Jul 26  2015 Programs
drwxrwxr-x.  2 1000 1000 4.0K Jul 26  2015 PCbuild
drwxrwxr-x.  6 1000 1000 4.0K Jul 26  2015 PC
drwxrwxr-x.  2 1000 1000 4.0K Jul 26  2015 Parser
drwxrwxr-x.  4 1000 1000 4.0K Jul 26  2015 Objects
drwxrwxr-x. 11 1000 1000 4.0K Jul 26  2015 Modules
-rwxr-xr-x.  1 1000 1000 7.0K Jul 26  2015 install-sh
-rw-r--r--.  1 1000 1000 148K Jul 26  2015 configure.ac
-rwxr-xr-x.  1 1000 1000 455K Jul 26  2015 configure
-rwxr-xr-x.  1 1000 1000  35K Jul 26  2015 config.sub
-rwxr-xr-x.  1 1000 1000  42K Jul 26  2015 config.guess
-rw-r--r--.  1 1000 1000 8.3K Jul 26  2015 aclocal.m4
drwxrwxr-x. 18 1000 1000 4.0K Jul 26  2015 Doc
[root@wmysql Python-3.5.0b4]# ./configure 
checking build system type... x86_64-unknown-linux-gnu
checking host system type... x86_64-unknown-linux-gnu
checking for --enable-universalsdk... no
checking for --with-universal-archs... no
checking MACHDEP... linux
checking for --without-gcc... no
checking for gcc... no
checking for cc... no
checking for cl.exe... no
configure: error: in `/python_work/Python-3.5.0b4':
configure: error: no acceptable C compiler found in $PATH
See `config.log' for more details

```

上面报错说明没有gcc包，由于最小化安装，没有安装gcc。将gcc包安装上

```shell

yum install -y gcc

```

重新安装：

```shell

./configure
make
make install

```

等待超长时间之后。。。

```shell

[root@wmysql Python-3.5.0b4]# python3 --version
Python 3.5.0b4

```


安装完毕


![小w](https://wx2.sinaimg.cn/mw1024/891ecf4fly1fr361nvrcnj207w07sad7.jpg)

###### 博文内容为小w原创，部分内容引用自官方资料，如转载请注明出处。^_^

###### 感谢关注w的小站，本小站将不定期(完全看心情～)更新分享技术知识，欢迎阅读，网址：www.itwwb.com



