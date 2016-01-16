---
layout: post
title: "data exchange"
date: 2015-12-15 23:54:41 +0800
comments: true
categories: ['sqoop']
tags: sqoop
description: 数据交换 常见场景及方法
toc: true
---

本文将介绍一下常见的几种数据交换的场景，及解决方法

<!--more-->

1.不同集群之间 hdfs 文件 或 hive表 迁移

2.利用sqoop实现按日增量导入，及状态表的全量覆盖

---

## 环境准备

- hadoop2
- sqoop-1.4.6.bin__hadoop-2.0.4-alpha
- hive

---

## hive表集群间迁移

利用hadoop distcp命令，例如把hive 的 test库下的g_info表导入到数据目录下，如果仅仅是文件拷贝而不涉及hive表，则这一条命令就可以了

``` bash
hadoop distcp hdfs://192.168.1.169:9000/user/hive/warehouse/test.db/t1 hdfs://192.168.7.11:9000/user/hive/warehouse/test.db/t1_cp
```

### <font color="#d67b0f"> hive表是非分区表 </font>

1.先创建「外部表」

``` sql
CREATE external TABLE t1_cp(
  a bigint,
  b int,
  c int)
COMMENT 'Created by xiaolong.yuanxl 20151211'
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY '\t'
  LINES TERMINATED BY '\n'
STORED AS TEXTFILE location 'hdfs://192.168.7.11:9000/user/hive/warehouse/test.db/t1_cp';
```

2.通过load data的方式加载数据

``` sql
LOAD DATA INPATH 'hdfs://192.168.7.11:9000/user/hive/warehouse/test.db/t1_cp/' INTO TABLE t1_cp;
```

### <font color="#d67b0f"> hive是分区表 </font>

1.如果hive是分区表，则迁移复制过来的目录结构应该带「分区文件夹」的，所以不能直接一次性导入，因此需要先创建「外部表」且添加「分区信息」，去掉「数据目录」

``` sql
CREATE external TABLE t1_cp(
  a bigint,
  b int,
  c int)
COMMENT 'Created by xiaolong.yuanxl 20151211'
PARTITIONED BY (
  `pdt` string)
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY '\t'
  LINES TERMINATED BY '\n'
STORED AS TEXTFILE;
```

2.利用 `hadoop distcp` 命令把数据复制到 hive `show create table t1_cp` 下的 location 数据目录下

3.由于一个表的分区信息可能很多，甚至几千个，所以需要用脚本自动化来实现「添加分区」及「导入数据」的过程 （注意：根据自己的实际情况，修改路径及 awk截取字符串的位置）

``` bash
#!/bin/sh
ret=`hadoop fs -ls hdfs://192.168.7.168:9000/user/hive/warehouse/user.db/channel_user_reg_t | awk '{print substr($8,78)}'`

for pt in ${ret[@]}
do
    echo "hive -e \"use user;alter table channel_user_reg_t add partition (pdt="\"${pt}\"") location \"/user/hive/warehouse/user.db/channel_user_reg_t/pdt=${pt}\";\""
    hive -e "use user;alter table channel_user_reg_t add partition (pdt="\"${pt}\"") location \"/user/hive/warehouse/user.db/channel_user_reg_t/pdt=${pt}\";"
done
```

4.执行脚本 ` nohup sh your.sh & `


---

## sqoop 导入数据

一般我们的导入，有「全量覆盖」或「按日新增」 两种方式，而这2种方式，都要先做两件事情

1.创建内部表，如果是「按日增量」则需要添加 partition 语句

``` sql
CREATE TABLE t1_cp(
  a bigint,
  b int,
  c int)
COMMENT 'Created by xiaolong.yuanxl 20151211'
PARTITIONED BY (
  `pdt` string)
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY '\t'
  LINES TERMINATED BY '\n'
STORED AS TEXTFILE;
```

2.建立分区

``` sql
alter table t1_cp ADD PARTITION (pdt='2015-12-12');
```

3.执行sqoop导入命令

### <font color="#d67b0f"> 无分区 </font>

<font color="#b65b25" size="3">注:下面是一行命令</font>

``` bash
sqoop import --connect "jdbc:mysql://127.0.0.1:3306/test" --username sqoop --password sqoop \
--query "select id,name,age,created from test.school where created >= '2015-12-10 00:00:00' and created < '2015-12-11 00:00:00' and \$CONDITIONS " \
--target-dir "/user/hive/warehouse/test.db/t2" \
--delete-target-dir \
--fields-terminated-by '\t' \
--hive-database "test" --hive-table "t2" \
--verbose -m 1

```

重要参数说明
  * `--query` 代表需要执行的sql语句，其中需要硬性添加一个 `$CONDITIONS` 变量，注意shell环境$符号需要转义
  * `--target-dir` 代表sqoop执行mr完后的输出目录，跟query成对出现
  * `--delete-target-dir` 代表每次删除目录，然后重新覆盖导入
  * `--fields-terminated-by` 字段间分隔符
  * `--m` 代表mapreduce中 map 个数

### <font color="#d67b0f"> 分区表 </font>

<font color="#b65b25" size="3">注:下面是一行命令</font>

``` bash
sqoop import --connect "jdbc:mysql://127.0.0.1:3306/test" --username sqoop --password sqoop \
--query "select id,name,age,created from test.school where created >= '2015-12-10 00:00:00' and created < '2015-12-11 00:00:00' and \$CONDITIONS " \
--target-dir "/user/hive/warehouse/test.db/t2” \
--append --fields-terminated-by '\t' \
--hive-database "test" --hive-table "t2" \
--verbose -m 1
```

重要参数说明
  * `--append` 追加模式，不会增量，仅仅增加数据文件。因此需要用sql控制增量
  * 不要添加 `--delete-target-dir`

---

## 其他

1.有些数据库，会有中文的问题，需要在sql中添加转义，例如

``` sql
convert(binary convert(title using latin1) using gbk) as title
```

2.sqoop会用 `hadoop-env.sh` 里配置的 `$JAVA_HOME` 指向的 jdk，而不会关心系统环境变量

3.如果需要自动化，可以利用shell获取时间，然后每日crontab导入



### 后记补充

#### 问题一: 最近 使用hive表的字段时，发现错位了

原因是字段内容本身有 \t 字符，然后删除表，设定sqoop脚本为 \001 字符，然后创建hive表分割字符为 \001  换行符也是一样（hive只支持\n）

可以添加 `--hive-drop-import-delims` （Drops \n, \r, and \01 from string fields when importing to Hive.）参数

#### 问题二: 导出数据会自动把 数字0 和非零 mysql 数据，转换到hive 为 true false

引用自：http://www.cnblogs.com/cenyuhai/archive/2013/09/06/3306073.html

jdbc会把 tinyint（1）认为是 java.sql.Types.BIT ,然后sqoop就会转为Boolean了

* 解决方法1：
  * 在连接上加上一句话 tinyInt1isBit=false 例如  jdbc:mysql://localhost/test?tinyInt1isBit=false

* 解决方法2：
  * hive使用 --map-column-hive foo=tinyint
  * 非hive使用--map-column-java foo=integer
