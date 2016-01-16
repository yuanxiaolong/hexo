---
layout: post
title: "hbase data transfer"
date: 2016-01-10 18:32:27 +0800
comments: true
categories: hbase
tags: hbase
description: hbase 数据迁移
toc: true
---

Hbase 迁移数据

<!--more-->

利用Hbase Export Import 导入导出工具，进行数据迁移

---

Hbase 数据迁移的方法有大概4种，最近需要迁移数据，所以研究了一下，发现最灵活合适的 是利用 Export Import 进行导入导出。其他方式可以自己百度查阅。


## 环境准备

* hbase 2个不同集群 clusterA 和 clusterB （忽略网络不通）


## 步骤

前置：先在需要导入的新集群上，创建跟老集群一样的hbase表

1.先把 clusterA 的 Hbase 表导出到 cluster A hdfs 里

例如把 fact_mdd 表导出到 hdfs 路径

```
./hbase org.apache.hadoop.hbase.mapreduce.Export fact_mdd /hbaseOldData/fact_mdd
```

导出的时候，可以添加参数。例如指定 版本、时间戳、列族、压缩等

```
./hbase org.apache.hadoop.hbase.mapreduce.Export \
-D hbase.mapreduce.scan.column.family=1000121000 \
-D mapreduce.output.fileoutputformat.compress=true \
-D mapreduce.output.fileoutputformat.compress.codec=org.apache.hadoop.io.compress.SnappyCodec \
-D mapreduce.output.fileoutputformat.compress.type=BLOCK \
fact_mdd /hbaseOldData/fact_mdd_1000121000
```

2.导入，指定的hdfs路径是老集群的，重复导入没有关系，只会刷新记录的 timestamp

``` bash
./hbase org.apache.hadoop.hbase.mapreduce.Import fact_mdd hdfs://192.168.7.168:9000/hbaseOldData/fact_mdd_1000121000
```


如果导入导出的集群，网段不通，可以先 get hdfs 到本地，然后再 put 到新集群hdfs上。

导入的时候还可以指定 hbase元数据版本（没有测试）

在导出的时候，我碰见一个表比较大，所以 Export 时候会失败，而且会挂掉 hbase 节点，所以可以按列族导出，然后再逐一导入。可以写个脚本，让hbase批量执行

``` bash
./hbase shell yourscript.hbaseshell
```

建议，导入完成后，逐一修改列压缩为 SNAPPY 格式，注意列名，如果指定了一个错误的列名，会新增一个列族，也可以用脚本批量执行

``` bash
disable 'test'
alter 'test', NAME => 'f1', COMPRESSION => 'snappy'
alter 'test', NAME => 'f2', COMPRESSION => 'snappy'
alter 'test', NAME => 'f3', COMPRESSION => 'snappy'
enable 'test'
major_compact 'test'
exit
```

---

## 官网文档

> Export
>
> `$ bin/hbase org.apache.hadoop.hbase.mapreduce.Export <tablename> <outputdir> [<versions> [<starttime> [<endtime>]]]`
>
> By default, the Export tool only exports the newest version of a given cell, regardless of the number of versions stored. To export more than one version, replace <versions> with the desired number of versions.
>
>Note: caching for the input Scan is configured via hbase.client.scanner.caching in the job configuration.

> Import
>
> `$ bin/hbase org.apache.hadoop.hbase.mapreduce.Import <tablename> <inputdir>`
>
> To import 0.94 exported files in a 0.96 cluster or onwards, you need to set system property "hbase.import.version" when running the import command as below:
>
> `$ bin/hbase -Dhbase.import.version=0.94 org.apache.hadoop.hbase.mapreduce.Import <tablename> <inputdir>`
