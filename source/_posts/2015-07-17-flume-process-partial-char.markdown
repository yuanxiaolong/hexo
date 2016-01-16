---
layout: post
title: "flume process partial char"
date: 2015-07-17 11:50:06 +0800
comments: true
categories:  flume
tags: flume
description: flume bug 修复
toc: true
---

修复了flume的一个BUG

<!--more-->

当flume读取到 特殊字符时，会中断对整个文件的读取，1.5.0 。1.6.0 这里的代码稍微不一样，但是也直接返回 -1 了

---

## 现象

flume source 为 spooldir 方式，默认会以utf8读取log文件，但是如果写入的数据夹杂了非utf8编码的记录，则flume读取到此非法字符时，就会中止整个文件的读取。

例如log文件中有1000行，500行时有这样一条记录 ，写入时以非utf8写

```
{"user_account":"💮毛腿腿💮","user_id”:"11111111","ip”:”127.0.0.1","time":"1430755546","hour":"2015050500"}
```

则flume 只会读取前500行，并传递给source，其中第500行

```
{"user_account":"
```

---

## 分析定位

为什么会产生这样的现象？先一起来看一下 spooldir 读取日志的流程

![](/images/flume/20150717/1.png)


flume会把日志里每一行，转换为一个Event 来处理，Event是最小单位。但flume操作的时候，以文件为逻辑单位，所以当遇到特殊字符时，jdk utf8 decoder 解析不出来，flume则认为到了文件末尾，因此就中止了整个文件的读取。

---

## 解决

知道了原因，那么我们就可以修改flume源码了，修改ResettableFileInputStream.java

``` java
// there may be a partial character in the decoder buffer
    } else {
      incrPosition(delta, false);
      return -1;
    }

```

修改为，我们把特殊字符替换成空格（ASCII 32）flume 1.5.0

原理：当解析不出字符时，走到else分支，如果已到文件末尾，则自增全局文件指针 delta （值为1）个，并返回 -1 代表文件结束。如果未到文件末尾，则自增全局指针1下，跳过 “脏字符”，清空缓冲区，再填充，并处理

``` java
 // there may be a partial character in the decoder buffer
    } else {
      if(isEndOfInput) {
          incrPosition(delta, false);
          logger.info("End of File.");
          return -1;//end of file
      } else{
          incrPosition(1, false);
          buf.clear();
          buf.flip();
          refillBuf();
          logger.warn("May have special characters.");
          return 32;//a partial character 空格的ASCII
      }
    }
```

flume 1.6.0 [ResettableFileInputStream.java](https://github.com/apache/flume/blob/release-1.6.0/flume-ng-core/src/main/java/org/apache/flume/serialization/ResettableFileInputStream.java)

---

## 构建flume

### 环境准备

* 选择一个flume版本，fork到自己的仓库
* maven


1.修改parent pom.xml，修改<hadoop.version>为自己的版本

2.注释掉下面子pom.xml的单元测试

* flume-ng-sinks/flume-hdfs-sink/pom.xml
* flume-tools/pom.xml

``` xml
<dependency>
        <groupId>org.apache.hbase</groupId>
        <artifactId>hbase</artifactId>
        <version>${hbase.version}</version>
        <classifier>tests</classifier>
        <scope>test</scope>
</dependency>

<dependency>
         <groupId>org.apache.hadoop</groupId>
         <artifactId>hadoop-test</artifactId>
         <version>${hadoop.version}</version>
</dependency>
```

3.删除 flume-ng-sinks/flume-ng-hbase-sink/src/test 下的单元测试（我这里编译不过，因此删了）

4.获取ua-parser-1.3.0.jar ，并复制到本地仓库，[<font color="#3573c6">我的百度云共享</font>](http://pan.baidu.com/s/1kT69toN)

5.执行 `mvn install -Phadoop-2 -DskipTests -Dtar`  （如果你是hadoop2的话指定为 -Phadoop-2，否则不用添加）

6.flume-ng-dist/target  下就是构建完成的东西，apache-flume-1.5.2-bin.tar.gz 就可以直接用了

---

## debug flume源码

### 配置

启动flume命令，其中 --name realtime 是配置文件中自定义的，其中 http://<ip>:34545/metrics 是flume的监控统计信息

``` bash
./flume-ng agent --conf ../conf --conf-file ../conf/flume-conf.properties --name realtime -Dflume.monitoring.type=http -Dflume.monitoring.port=34545
```

修改 ` flume-conf.properties ` 其中 ` realtime.sources.fortest.ignorePattern=^.*(?<!\\d{4}\\.log)$ ` 表示只匹配监控 0001.log 、7743.log 这样的文件

``` java
# globel
realtime.sources=fortest
realtime.channels=fortest
realtime.sinks=fortest

#source
realtime.sources.fortest.type=spooldir
realtime.sources.fortest.spoolDir=/Users/xiaolongyuan/Downloads/apache-flume-1.5.0.1-bin/data/send
realtime.sources.fortest.channels=fortest
realtime.sources.fortest.batchSize=1000
realtime.sources.fortest.deserializer=LINE
realtime.sources.fortest.deserializer.maxLineLength=20000000
realtime.sources.fortest.bufferMaxLineLength＝2000000
realtime.sources.fortest.decodeErrorPolicy=IGNORE
realtime.sources.fortest.ignorePattern=^.*(?<!\\d{4}\\.log)$

#channel
realtime.channels.fortest.type = memory
realtime.channels.fortest.capacity = 10000
realtime.channels.fortest.transactionCapacity = 10000
realtime.channels.fortest.byteCapacityBufferPercentage = 20
realtime.channels.fortest.byteCapacity = 800000

#sink
realtime.sinks.fortest.type=file_roll
realtime.sinks.fortest.sink.directory=/Users/xiaolongyuan/Downloads/apache-flume-1.5.0.1-bin/data/received
realtime.sinks.fortest.channel = fortest

```

本地读取file，输出到本地file，方便对比2个file的行数

### DEBUG

源码导入到eclipse 或 idea，并修改 ` flume-env.sh `

``` bash
JAVA_OPTS="-Xdebug -Xrunjdwp:transport=dt_socket,address=8001,server=y,suspend=y”
```
在eclipse或idea里新建 remote debug，ip本地，端口8001，连接、打断点、测试。
