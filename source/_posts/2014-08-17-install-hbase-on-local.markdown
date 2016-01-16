---
layout: post
title: "install hbase on local"
date: 2014-08-17 16:45:34 +0800
comments: true
categories: hbase
tags: hbase
share: true
description: 单机伪分布式安装
toc: true
---
介绍一下Hbase单机伪分布式安装

<!--more-->

hbase是一个列式数据库，基于hdfs文件的。跟mongodb同属nosql系列，具有海量insert和key-value简单高效查询的作用。

---

## 环境准备

*  hadoop环境，伪分布式或集群环境
*  hbase 适当版本对应hadoop-1.x、hadoop-2.x 的安装包，可以在[<font color="#6868b4">这里下载</font>](http://hbase.apache.org/)

---

## 安装过程（根据自己机器配置适当修改）

1.解压tar包，我用的是hadoop-1.2.1，因此hbase此时选择的是hbase-0.98.4-hadoop1

2.设置环境变量，追加下面内容 <font color="#4e5d5e">（记得用source命令将其生效）</font>

``` bash
#hbase
export HBASE_HOME=/Users/xiaolongyuan/Documents/hbase-0.98.4-hadoop1/
export PATH=$PATH:$HBASE_HOME/bin
```

3.修改 <font color="green"> hbase-env.sh </font>和 <font color="green"> hbase-site.xml </font>

``` bash hbase-env.sh
export JAVA_HOME=`/System/Library/Frameworks/JavaVM.framework/Versions/Current/Commands/java_home`
# Tell HBase whether it should manage it's own instance of Zookeeper or not.
export HBASE_MANAGES_ZK=true
```

``` xml hbase-site.xml
<property>
  <name>hbase.rootdir</name>
  <value>hdfs://127.0.0.1:9000/hbase</value>
</property>
<property>
  <name>hbase.cluster.distributed</name>
  <value>true</value>
</property>
```

---

## 启动过程

由于hbase基于hdfs的，因此

*  启动顺序 hadoop -> hbase
*  关闭顺序 hbase  -> hadoop

示例：

1.启动hadoop、启动hbase后，可以看到进程如下

``` bash
xiaolongyuan@xiaolongdeMacBook-Air bin$ jps
75399 JobTracker
75326 SecondaryNameNode
77145 Jps
76873 HQuorumPeer
77022 HRegionServer
75127 NameNode
76918 HMaster
75498 TaskTracker
75226 DataNode
```

2.执行hbase命令

``` bash
xiaolongyuan@xiaolongdeMacBook-Air bin$ hbase shell
HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 0.98.4-hadoop1, r890e852ce1c51b71ad180f626b71a2a1009246da, Mon Jul 14 18:54:31 PDT 2014

hbase(main):001:0> version
0.98.4-hadoop1, r890e852ce1c51b71ad180f626b71a2a1009246da, Mon Jul 14 18:54:31 PDT 2014

hbase(main):002:0> exit

```

3.发现hdfs下面有hbase的文件

``` bash
xiaolongyuan@xiaolongdeMacBook-Air bin$ hadoop dfs -ls /hbase
Warning: $HADOOP_HOME is deprecated.

Found 6 items
drwxr-xr-x   - xiaolongyuan supergroup          0 2014-08-17 12:42 /hbase/.tmp
drwxr-xr-x   - xiaolongyuan supergroup          0 2014-08-17 12:42 /hbase/WALs
drwxr-xr-x   - xiaolongyuan supergroup          0 2014-08-17 12:42 /hbase/data
-rw-r--r--   1 xiaolongyuan supergroup         42 2014-08-17 12:42 /hbase/hbase.id
-rw-r--r--   1 xiaolongyuan supergroup          7 2014-08-17 12:42 /hbase/hbase.version
drwxr-xr-x   - xiaolongyuan supergroup          0 2014-08-17 12:42 /hbase/oldWALs
```
