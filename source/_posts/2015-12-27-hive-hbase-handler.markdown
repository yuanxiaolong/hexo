---
layout: post
title: "hive hbase handler"
date: 2015-12-27 17:43:20 +0800
comments: true
categories: hive
tags: hive
description: hive 和 Hbase 整合
toc: true
---

介绍 hive 和 hbase 互通

<!--more-->

利用hive 查询 hbase 已有数据，或向hbase导入数据

---

## 介绍

主要是利用 hive 的 「hive-hbase-hander」模块进行处理。我用的是 hive-1.1.0，已经可以直接使用这个模块了，
听说低版本的hive，还需要将这个模块，单独打成jar后，利用jar包来处理（最后我将简略写一下构建此jar的过程）。

---

## hive 中 创建 hbase 表

``` sql
CREATE TABLE hbase_table_1(key int, value string) \
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler' \
WITH SERDEPROPERTIES ("hbase.columns.mapping" =":key,cf1:") \
TBLPROPERTIES("hbase.table.name" = "aaa");
```

这句话的意思，新建一个 hive表 叫 `hbase_table_1` ，新建一个 hbase 表叫 `aaa` ，然后它们的映射关系是，
hive中key -> base中 key ，hive中value -> base中cf1

然后我们可以从一个已存在的hive表中，给我们新建的 hbase_table_1 导入数据，这里虚构一个表叫 `pokes`

``` sql
insert overwrite table hbase_table_1 select * from pokes;
```

导入成功后，我们可以比较一下数据，例如

进入hive

``` sql
select * from hbase_table1;
```

进入hbase

``` sql
scan "aaa"
```

---

##  hive读取已存在的hbase表

这个场景特别常见，hive和hbase分开存储，但查询统一走hive

1.先造一个hbase表并放入数据，创建一个scores表，有2个列族，grade列族只有1列，course列族有2列

``` sql
create 'scores','grade', 'course'

put 'scores','Tom','grade:','5'
put 'scores','Tom','course:math','97'
put 'scores','Tom','course:art','87'
put 'scores','Jim','grade','4'
put 'scores','Jim','course:math','89'
put 'scores','Jim','course:art','80'
```

2.建立hive外部表，映射到hbase上

``` sql
CREATE EXTERNAL TABLE \
hive_scores (rowkey string, grade map<string,string>, course map<string,string>) \
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler' \
WITH SERDEPROPERTIES ("hbase.columns.mapping" = ":key,grade:,course:") \
TBLPROPERTIES("hbase.table.name" = "scores");
```

此时便可以查询hbase表了

---

## 编译 hive-hbase-hander 插件

注：如果你能成功执行以上语句，就没必要编译这个插件了，编译好后，需要放到 `$HIVE_HOME/lib` 下

1.首先导入hive此模块源码，到IDE里，去除原有的工程依赖 jar包，然后在 classpath 下添加如下jar包

``` bash
/YOUR_WORK_DIR/hive-1.1.0/lib/commons-io-2.4.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/commons-lang-2.6.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/commons-logging-1.1.3.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/common/hadoop-common-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/hdfs/hadoop-hdfs-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/hdfs/hadoop-hdfs-nfs-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/mapreduce/hadoop-mapreduce-client-app-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/mapreduce/hadoop-mapreduce-client-common-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/mapreduce/hadoop-mapreduce-client-core-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/mapreduce/hadoop-mapreduce-client-hs-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/mapreduce/hadoop-mapreduce-client-hs-plugins-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/mapreduce/hadoop-mapreduce-client-shuffle-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/common/hadoop-nfs-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/yarn/hadoop-yarn-api-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/yarn/hadoop-yarn-applications-distributedshell-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/yarn/hadoop-yarn-applications-unmanaged-am-launcher-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/yarn/hadoop-yarn-client-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/yarn/hadoop-yarn-common-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/yarn/hadoop-yarn-registry-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/yarn/hadoop-yarn-server-applicationhistoryservice-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/yarn/hadoop-yarn-server-common-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/yarn/hadoop-yarn-server-nodemanager-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/yarn/hadoop-yarn-server-resourcemanager-2.6.0.jar
/YOUR_WORK_DIR/hadoop-2.6.0/share/hadoop/yarn/hadoop-yarn-server-web-proxy-2.6.0.jar
/YOUR_WORK_DIR/hbase-1.0.0-cdh5.4.0/lib/hbase-annotations-1.0.0-cdh5.4.0.jar
/YOUR_WORK_DIR/hbase-1.0.0-cdh5.4.0/lib/hbase-client-1.0.0-cdh5.4.0.jar
/YOUR_WORK_DIR/hbase-1.0.0-cdh5.4.0/lib/hbase-common-1.0.0-cdh5.4.0.jar
/YOUR_WORK_DIR/hbase-1.0.0-cdh5.4.0/lib/hbase-hadoop2-compat-1.0.0-cdh5.4.0.jar
/YOUR_WORK_DIR/hbase-1.0.0-cdh5.4.0/lib/hbase-prefix-tree-1.0.0-cdh5.4.0.jar
/YOUR_WORK_DIR/hbase-1.0.0-cdh5.4.0/lib/hbase-protocol-1.0.0-cdh5.4.0.jar
/YOUR_WORK_DIR/hbase-1.0.0-cdh5.4.0/lib/hbase-rest-1.0.0-cdh5.4.0.jar
/YOUR_WORK_DIR/hbase-1.0.0-cdh5.4.0/lib/hbase-server-1.0.0-cdh5.4.0.jar
/YOUR_WORK_DIR/hbase-1.0.0-cdh5.4.0/lib/hbase-thrift-1.0.0-cdh5.4.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-accumulo-handler-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-ant-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-beeline-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-cli-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-common-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-contrib-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-exec-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-hwi-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-jdbc-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-metastore-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-serde-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-service-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-shims-0.20S-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-shims-0.23-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-shims-common-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-shims-scheduler-1.1.0.jar
/YOUR_WORK_DIR/hive-1.1.0/lib/hive-testutils-1.1.0.jar
/YOUR_WORK_DIR/jsr305-3.0.0.jar
/YOUR_WORK_DIR/zookeeper-3.4.7/zookeeper-3.4.7.jar
```

2.然后再根据自己IDE的不同，百度搜一下如何打成jar

后记：通过hive查询hbase，利用rowkey查询特别快，但通过其他条件查，不一定，甚至很慢，效果不是很理想。

参考：

[http://blog.csdn.net/wulantian/article/details/38111683](http://blog.csdn.net/wulantian/article/details/38111683)

[http://www.aboutyun.com/thread-7817-1-1.html](http://www.aboutyun.com/thread-7817-1-1.html)
