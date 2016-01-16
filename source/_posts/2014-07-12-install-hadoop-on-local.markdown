---
layout: post
title: "install hadoop  on local"
date: 2014-07-12 20:51:26 +0800
comments: true
category: hadoop
tags: hadoop
share: true
description: 单机伪分布式hadoop
toc: true
---
介绍如何在本机搭建一个单机伪分布式的hadoop 1.2.1 ，用于学习

<!--more-->

---

## 环境准备

1.一台有linux系统的电脑，ubuntu、centos均可。在物理机windows上装虚拟机linux也行至少分配1G以上内存，便于启动、调试。

2.安装java(版本1.6以上)，下载[hadoop-1.2.1-bin.tar.gz](http://apache.fayea.com/apache-mirror/hadoop/common/hadoop-1.2.1/)

``` bash
xiaolongyuan@xiaolongdeMacBook-Air .ssh$ java -version
java version "1.7.0_51"
Java(TM) SE Runtime Environment (build 1.7.0_51-b13)
Java HotSpot(TM) 64-Bit Server VM (build 24.51-b03, mixed mode)
```

3.配置ssh（可以参考[这里](/blog/2014/07/13/install-ssh-and-config/)）

## 配置

1.解压hadoop-1.2.1-bin.tar.gz到文件夹中,并在此文件夹下,新建 tmp 文件夹，然后修改几个文件


``` xml core-site.xml
<property>
        <name>fs.default.name</name>
        <value>hdfs://localhost:9000</value>
</property>
```

``` xml hdfs-site.xml
<property>
        <name>dfs.replication</name>
        <value>1</value>
</property>
<property>
        <name>hadoop.tmp.dir</name>
        <value>/Users/xiaolongyuan/Documents/hadoop-1.2.1/tmp</value>
</property>
```

``` xml mapred-site.xml
<property>
        <name>mapred.job.tracker</name>
        <value>localhost:9001</value>
</property>
```

(注:由于master文件和slave文件都是 localhost，因此不用修改)

2.格式化namenode的hdfs

``` bash
cd hadoop-1.2.1/bin
./hadoop namenode -format
```

3.修改 `conf/hadoop-env.sh`，向里面添加`JAVA_HOME`

``` bash
# The java implementation to use.  Required.
export JAVA_HOME='本机java的安装路径'
```

4.启动hadoop，jps检查hadoop进程是否启动

![](/images/hadoop/hadoop-install-on-local-1.png)

5.再访问一下web界面，看是否可以正常访问
* http://localhost:50030/jobtracker.jsp
* http://localhost:50070/dfshealth.jsp
