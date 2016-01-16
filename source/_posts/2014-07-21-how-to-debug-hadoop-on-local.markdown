---
layout: post
title: "how to debug hadoop on local"
date: 2014-07-21 20:59:42 +0800
comments: true
categories: hadoop
tags: hadoop
share: true
description: 伪分布式debug hadoop源码
toc: true
---

介绍一下如何在伪分布式hadoop上，配置hadoop源码debug环境，以便学习

<!--more-->

本文分2部分来介绍

1.  配置源码显示，以便跟踪
2.  创建远程debug hadoop的jvm

---

## 环境准备

1.  本地部署伪分布式hadoop（可以参考 [<font color="#6868b4">这里</font>](http://blog.yuanxiaolong.cn/blog/2014/07/12/install-hadoop-on-local/)）
2.  下载hadoop源码，2种方式。
    *  svn [<font color="#0c95a1"> http://svn.apache.org/repos/asf/hadoop/common/tags/release-1.2.1/ </font>](http://svn.apache.org/repos/asf/hadoop/common/tags/release-1.2.1/)
    *  git [<font color="#0c95a1"> github 地址 </font>](https://github.com/apache/hadoop-common/tree/release-1.2.1)
3.  eclipse 配置已好，标准的就可以，可以在 [<font color="#6868b4">这里</font>](http://www.eclipse.org/downloads/) 下载

---

## 配置eclipse hadoop源码工程

<font color="#7c837f" size="3"> 为了方便查看debug时所在的行号、变量、类什么的，我们需要先配置hadoop源码工程到eclipse里。</font>

1.解压源码包，将源码包里的这3个目录下的文件copy出来，放到一个新的org/apache/hadoop文件夹内，准备导入到eclipse里。

* src/core/org/apache/hadoop
* src/hdfs/org/apache/hadoop
* src/mapred/org/apache/hadoop

2.eclipse里新建一个java工程，导入刚才合并后的 org/apache/hadoop 内的文件 <font color="#7c837f">（此时刷新工程，会有很多红叉）</font>

3.将hadoop源码里的lib文件夹下的所有jar包 copy出来 <font color="#7c837f"> (同时网上找一个 </font>[<font color="#6868b4">ant.jar</font>](http://pan.baidu.com/s/1bnlI8EV) <font color="#7c837f">添加进去)</font>，放到刚才我们新建的工程里，并配置eclipse build path 依赖这些jar

4.此时，还有1个报错，是因为eclipse默认禁止使用sun的jar包，需要在buildpath中添加一个规则，右击工程 <font color="#d66d58">属性->buildpath</font>，如下：
![](/images/hadoop/eclipse-hadoop-src-sun-jar-error.png)

---

## 开始debug

1.修改 hadoop-env.sh，在8001端口上，开启对namenode 的jvm调试。<font color="#7c837f">（如果学过j2ee，跟tomcat远程调试一样的道理）</font>

``` bash hadoop-env.sh
# Command specific options appended to HADOOP_OPTS when specified
#export HADOOP_NAMENODE_OPTS="-Dcom.sun.management.jmxremote $HADOOP_NAMENODE_OPTS"
export HADOOP_NAMENODE_OPTS="-Xdebug -Xrunjdwp:transport=dt_socket,address=8001,server=y,suspend=y"
export HADOOP_SECONDARYNAMENODE_OPTS="-Dcom.sun.management.jmxremote $HADOOP_SECONDARYNAMENODE_OPTS"
export HADOOP_DATANODE_OPTS="-Dcom.sun.management.jmxremote $HADOOP_DATANODE_OPTS"
export HADOOP_BALANCER_OPTS="-Dcom.sun.management.jmxremote $HADOOP_BALANCER_OPTS"
export HADOOP_JOBTRACKER_OPTS="-Dcom.sun.management.jmxremote $HADOOP_JOBTRACKER_OPTS"
```

2.在我们eclipse hadoop源码工程中，打开 <font color="#a6752f"> NameNode.java </font> ，在280行打上一个断点。

3.在eclipse里新建远程调试<font color="#7c837f">（先别点debug按钮，因为此时hadoop还没运行）</font>
![](/images/hadoop/hadoop-eclipse-debugconf.png)

4.运行hadoop，可以看到在8001开启了端口用于debug
![](/images/hadoop/hadoop-start-debug-port.png)

5.此时，再点击hadoop工程的debug远程调试按钮，可以看到，断点已经生效
![](/images/hadoop/hadoop-breakpoint.png)

---

## debug shell command

1.修改 bin/hadoop 脚本。

``` bash
elif [ "$COMMAND" = "fs" ] ; then
CLASS='org.apache.hadoop.fs.FsShell'
HADOOP_OPTS="$HADOOP_OPTS $HADOOP_CLIENT_OPTS"
```

在下面多加一句，然后重新启动Hadoop集群，这样在命令行的所有操作都可以在FsShell里断点了。

``` bash
HADOOP_OPTS="$HADOOP_OPTS $HADOOP_CLIENT_OPTS -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=8008"
```
