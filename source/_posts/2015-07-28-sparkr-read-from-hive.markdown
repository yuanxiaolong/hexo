---
layout: post
title: "sparkR read from hive"
date: 2015-07-28 17:41:26 +0800
comments: true
categories: ['spark']
tags: spark
description: sparkR 读取 hive 表里的数据
toc: true
---

spark 1.4.0 开始支持 自带 R 的接口，可以利用R语法分析数据

<!--more-->

本文示例 用 sparkR on yarn 来分析 hive 表里的数据

---

## 环境准备

* scala
* hadoop 2.3.0
* spark 1.4.0
* hive (实验环境 0.12.0)

从官网下载二进制包，spark-1.4.0-bin-hadoop2.3 ([<font color="#1c56b8">我的百度云共享</font>](http://pan.baidu.com/s/1c0sEXhu)), ` tar xvf spark-1.4.0-bin-hadoop2.3.tar ` 解压。

---

## spark on yarn

设置环境变量，其中`YARN_CONF_DIR`是 spark on yarn 需要用到的环境变量

``` bash
#scala
export SCALA_HOME=/opt/app/scala-2.10.5
export PATH=$PATH:$SCALA_HOME/bin

#spark
export SPARK_HOME=/opt/app/spark-1.4.0-bin-hadoop2.3
export YARN_CONF_DIR=/etc/hadoop/conf/
export PATH=$SPARK_HOME/bin:$PATH
```

sparkR 启动有2种方式，

1.交互模式 , 其中 `--num-executors` 代表执行机器的个数 , ` --master yarn-client ` 固定写死 （一般用 yarn-client 模式执行，yarn-cluster模式一直没成功）

``` bash
/opt/app/spark-1.4.0-bin-hadoop2.3/bin/sparkR --num-executors 5 --master yarn-client
```

2.提交并执行R文件 ，类似下面命令

``` bash
/opt/app/spark-1.4.0-bin-hadoop2.3/bin/spark-submit --master yarn-client --num-executors 5 /opt/app/spark-1.4.0-bin-hadoop2.3/examples/src/main/r/dataframe.R
```

由于官网的demo dataframe.R 中，依赖了一个json文件，所以需要预先上传到 hdfs 上，路径根据自己的情况而定

``` bash
[yuanxiaolong@hadoop1 ~]$ hadoop fs -ls /opt/app/spark-1.4.0-bin-hadoop2.3/examples/src/main/resources/people.json
Found 1 items
-rw-r--r--   3 yuanxiaolong supergroup         73 2015-07-27 12:03 /opt/app/spark-1.4.0-bin-hadoop2.3/examples/src/main/resources/people.json
```

---

## read data from hive

1.先把hive的配置，让spark能感知到，即spark知道去连哪个hive metastore , 可以利用创建软连接的方式

``` bash
ln -s /etc/hive/conf/hive-site.xml /opt/app/spark-1.4.0-bin-hadoop2.3/conf/hive-site.xml
```

2.启动sparkR 交互模式，以便验证。

``` bash
/opt/app/spark-1.4.0-bin-hadoop2.3/bin/sparkR
## 省略
Welcome to SparkR!
 Spark context is available as sc, SQL context is available as sqlContext

> hiveContext <- sparkRHive.init(sc)
15/07/28 17:28:29 INFO hive.HiveContext: Initializing execution hive, version 0.13.1
15/07/28 17:28:29 INFO hive.metastore: Trying to connect to metastore with URI thrift://hadoop4:9083
15/07/28 17:28:30 INFO hive.metastore: Connected to metastore.
15/07/28 17:28:30 INFO session.SessionState: No Tez session required at this point. hive.execution.engine=mr.

## 省略
> showDF(sql(hiveContext, "USE data_row"))
## 省略
+------+
|result|
+------+
+------+

> showDF(sql(hiveContext, "select vid from pcp_vod_temp limit 10"))
## 省略
+----+
| vid|
+----+
|6296|
|5542|
|1962|
|4799|
|6006|
|6296|
|5542|
|1962|
|4799|
|6006|
+----+
## 省略
> rs <- sql(hiveContext, "select vid from pcp_vod_temp limit 10")
> first(rs)
## 省略
vid
1 6296

## 退出
> q()
Save workspace image? [y/n/c]: n

```

---

## RStudio 和 SparkR 结合

前置条件把 spark 1.4.0 的包，放置到 R中，可以通过软连接的方式

``` bash
ln -s /opt/app/spark-1.4.0-bin-hadoop2.3/R/lib/SparkR /usr/lib64/R/library/
```

进入RStudio 然后执行 sparkR ，执行（可以自己设置 executor的数量）

[https://github.com/apache/spark/tree/master/R](https://github.com/apache/spark/tree/master/R)

``` bash
# Set this to where Spark is installed
Sys.setenv(SPARK_HOME="/Users/shivaram/spark")
Sys.setenv(YARN_CONF_DIR="/etc/hadoop/conf")
Sys.setenv(SCALA_HOME="/opt/app/scala-2.10.5")
# This line loads SparkR from the installed directory
.libPaths(c(file.path(Sys.getenv("SPARK_HOME"), "R", "lib"), .libPaths()))
library(SparkR)
sc <- sparkR.init(master="yarn-client")
```
