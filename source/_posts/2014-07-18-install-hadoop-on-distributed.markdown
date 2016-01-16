---
layout: post
title: "install hadoop on distributed"
date: 2014-07-18 23:53:44 +0800
comments: true
category: hadoop
tags: hadoop
share: true
description: 完全分布式hadoop
toc: true
---
以3个节点为例，简单介绍一下hadoop完全分布式安装。

<!--more-->

---

## 准备工作
1.  已经阅读过 [<font color="#5a72ca">如何配置ssh互信</font>](http://blog.yuanxiaolong.cn/blog/2014/07/13/install-ssh-and-config/)
2.  已经阅读过 [<font color="#5a72ca">hadoop本地伪分布式安装</font>](http://blog.yuanxiaolong.cn/blog/2014/07/12/install-hadoop-on-local/)
3.  有3台机器（可以是1个物理机上用vbox做出的3个虚拟机）

---

## 开始安装
1.假定已经安装了java和下载过hadoop-1.2.1，并配置过ssh各个节点之间的互信

2.修改/etc/hosts 将3台机器的ip 分别加入到 **每一个** 节点的hosts文件内

3.假定已经完成上面2步，则3台机器IP分别为: <font color="#999698" size="3">我们需要选取1个namenode、2个datanode（通俗讲就是1个领导，2个员工）</font>

* <font color="#558c96" >192.168.1.1  (namenode) </font>
* <font color="#555796" >192.168.1.2  (datanode) </font>
* <font color="#555796" >192.168.1.3  (datanode) </font>

然后，开始修改下面几个文件

`core-site.xml` 把伪分布式安装中的 `fs.default.name` 值 hdfs://localhost:9000 修改成namenode的ip
`hdfs-site.xml`把伪分布式安装中的 `dfs.replication` 值设置为2，即冗余2份，分别在2个datanode中
`mapred-site.xml`同core-site.xml

4.修改slave文件，向里面添加datanode的ip，一行一个。（可以不设置masters文件,因为此文件是配置SecondaryNameNode的ip，默认跟namenode在同一个机器上）

5.利用scp命令，将已经配置好的一个hadoop文件夹拷贝到其他节点上。

6.格式化namenode节点的hdfs，并在namenode上启动hadoop集群，用jps查看一下进程是否存在,如果正常启动，则web界面可以访问。

* namenode下有NameNode、JobTracker、secondaryNameNode 进程
* datanode下有DataNode 、TaskTracker进程
