---
layout: post
title: "install flume on local"
date: 2014-08-17 20:46:58 +0800
comments: true
categories: flume
tags: flume
share: true
description: 单机安装flume用于学习
toc: true
---
介绍一下单机安装flume，并简单测试

<!--more-->

flume是一个数据采集的系统，可以实时获取产生的数据，并支持“推”和“拉”2种方式进行采集，且数据源也是多变的。

---

## 环境准备

*  下载flume [<font color="#a97d43">官网</font>](http://flume.apache.org/)
*  如果要采集数据源到hdfs，需要安装hadoop

---

## 配置

1.生成配置

``` bash
cd conf/

cp flume-conf.properties.template flume-conf.properties
flume-env.sh.template flume-env.sh
```

2.修改 <font color="green">flume-env.sh</font> 、<font color="green">flume-conf.properties</font>

``` bash flume-env.sh
JAVA_HOME=`/System/Library/Frameworks/JavaVM.framework/Versions/Current/Commands/java_home`
```

``` bash flume-conf.properties
# The configuration file needs to define the sources,
# the channels and the sinks.
# Sources, channels and sinks are defined per agent,
# in this case called 'agent'

localagent.sources = localSource
localagent.channels = localChannel
localagent.sinks = localSink

# For each one of the sources, the type is defined
localagent.sources.localSource.type = netcat
localagent.sources.localSource.bind = localhost
localagent.sources.localSource.port = 44444

# The channel can be defined as follows.
localagent.sources.localSource.channels = localChannel

# Each sink's type must be defined
localagent.sinks.localSink.type = logger

#Specify the channel the sink should use
localagent.sinks.localSink.channel = localChannel

# Each channel's type is defined.
localagent.channels.localChannel.type = memory

# Other config values specific to each type of channel(sink or source)
# can be defined as well
# In this case, it specifies the capacity of the memory channel
localagent.channels.localChannel.capacity = 100

```

其中如何配置，更多信息需要参考[<font color="#a545aa">user guide</font>](http://flume.apache.org/FlumeUserGuide.html)

---

## 运行

1.在flume的bin文件夹，运行下面命令，可以看到已经成功在44444端口启动服务

``` bash
xiaolongyuan@xiaolongdeMacBook-Air bin$ flume-ng agent --conf ../conf --conf-file ../conf/flume-conf.properties --name localagent -Dflume.root.logger=INFO,console
Info: Sourcing environment configuration script /Users/xiaolongyuan/Documents/flume-1.5.0/conf/flume-env.sh
Info: Including Hadoop libraries found via (/Users/xiaolongyuan/Documents/hadoop-1.2.1/bin/hadoop) for HDFS access
Warning: $HADOOP_HOME is deprecated.

……省略

2014-08-17 21:08:08,900 (lifecycleSupervisor-1-0) [INFO - org.apache.flume.source.NetcatSource.start(NetcatSource.java:150)] Source starting
2014-08-17 21:08:08,914 (lifecycleSupervisor-1-0) [INFO - org.apache.flume.source.NetcatSource.start(NetcatSource.java:164)] Created serverSocket:sun.nio.ch.ServerSocketChannelImpl[/127.0.0.1:44444]

```

2.利用telnet命令，输入任意字符

``` bash
telnet 127.0.0.1 44444

```

3.可以看到flume已经“感知”到了有数据发送过来了。

``` bash
2014-08-17 21:09:26,936 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:70)] Event: { headers:{} body: 74 68 69 73 20 69 73 20 6D 79 20 66 69 72 73 74 this is my first }
```

flume还可以将数据源的信息，直接写入到hdfs等等，需要进一步查看官网文档
