---
layout: post
title: "operating hbase through thrift with nodejs"
date: 2014-08-21 23:32:05 +0800
comments: true
categories: thrift
tags: thrift
share: true
description: 利用thrift，nodejs操作hbase
toc: true
---

用thrift做中间件，nodejs做客户端，操作hbase

<!--more-->

thrift是一种中间件，百度上有很多介绍，跟protobuf有些像。

![](/images/thrift/20140821/nodehbase.png)


## 环境准备

*  node安装配置 (百度搜索“nodejs linux 安装”)
*  thrift安装  
*  hadoop安装配置
*  hbase安装配置

注：hadoop和hbase的安装配置，我的blog里有介绍，感兴趣的同学可以翻阅一下。thrift安装就比较麻烦了，这里有个[<font color="#6868b4">官网连接</font>](http://thrift.apache.org/docs/BuildingFromSource)
还有不同操作系统安装之前的[<font color="#6868b4">依赖</font>](http://thrift.apache.org/docs/install/)。

thrift安装好后，可以查看一下版本

``` bash
xiaolongyuan@xiaolongdeMacBook-Air conf$ thrift -version
Thrift version 0.9.1
```

---

## 集成过程

1.先对nodejs进行安装thrift的依赖包,设置<font color="green">NODE_PATH</font>。

``` bash
xiaolongyuan@xiaolongdeMacBook-Air thrift_work$ npm install -g thrift
thrift@0.9.1 /usr/local/lib/node_modules/thrift
├── node-int64@0.3.1
└── nodeunit@0.8.8 (tap@0.4.12)
```

``` bash
#node
export NODE_PATH=/usr/local/lib/node_modules/
```

2.启动hadoop，启动hbase及hbase thrift服务

``` bash hadoop/bin
sh ./start-all.sh
```

``` bash hbase/bin
sh ./start-hbase.sh
hbase-daemon.sh start thrift -f
```

通过jps可以查看到进程<font color="#7c837f">（我用的是hadoop hbase伪分布式）</font>

``` bash
xiaolongyuan@xiaolongdeMacBook-Air bin$ jps
440 JobTracker
238 DataNode
974 ThriftServer
801 HMaster
754 HQuorumPeer
915 HRegionServer
361 SecondaryNameNode
538 TaskTracker
120 NameNode
1031 Jps
```

3.下载hbase源码，当前版本 [<font color="#6868b4">hbase-0.98.5-src.tar.gz</font>](http://mirror.bit.edu.cn/apache/hbase/stable/) ，在路径<font color="#942476"> hbase-thrift/src/main/resources/org/apache/hadoop/hbase/thrift </font>下有个 <font color="#ae8647">Hbase.thrift </font>的文件，建议先新建一个文件夹，以便存放生成的内容，进入文件夹，执行命令

``` bash
thrift -r --gen js:node /Users/xiaolongyuan/Documents/src-code/src-hbase-0.98.5/hbase-thrift/src/main/resources/org/apache/hadoop/hbase/thrift/Hbase.thrift
```

可以看到在此路径下有文件生成，提供给客户端语言使用的API接口。
![node](/images/thrift/20140821/thrift-node.png)

4.在gen-nodejs同级目录下，新建客户端js文件<font color="#7c837f">（如果不想在同级目录，需要修改js里的require路径)</font>

``` js node4hbase.js
var thrift = require('thrift'),
  HBase = require('./gen-nodejs/HBase.js'),
  HBaseTypes = require('./gen-nodejs/HBase_types.js'),
  connection = thrift.createConnection('localhost', 9090, {
    transport: thrift.TFramedTransport,
    protocol: thrift.TBinaryProtocol
  });

connection.on('connect', function() {
  var client = thrift.createClient(HBase,connection);
  client.getTableNames(function(err,data) {
    if (err) {
      console.log('gettablenames error:', err);
    } else {
      console.log('hbase tables:', data);
    }
    connection.end();
  });
});

connection.on('error', function(err){
  console.log('error', err);
});
```

5.node执行js文件，验证。

``` bash
xiaolongyuan@xiaolongdeMacBook-Air thrift_work$ node node4hbase.js
hbase tables: [ 'student' ]
```

查看hbase里的表

``` bash
xiaolongyuan@xiaolongdeMacBook-Air bin$ hbase shell

HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 0.98.4-hadoop1, r890e852ce1c51b71ad180f626b71a2a1009246da, Mon Jul 14 18:54:31 PDT 2014

hbase(main):001:0> list
TABLE
2014-08-21 23:01:08.459 java[1038:1903] Unable to load realm info from SCDynamicStore
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/Users/xiaolongyuan/Documents/hbase-0.98.4-hadoop1/lib/slf4j-log4j12-1.6.4.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/Users/xiaolongyuan/Documents/hadoop-1.2.1/lib/slf4j-log4j12-1.4.3.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
student
1 row(s) in 1.9050 seconds

=> ["student"]
hbase(main):002:0> exit

```

---

## 参考资料

1.  thrift官方教程 [<font color="#6868b4">地址</font>](http://thrift.apache.org/tutorial/)
2.  nodejs操作hbase [<font color="#6868b4">地址</font>](http://dailyjs.com/2013/07/04/hbase/)
