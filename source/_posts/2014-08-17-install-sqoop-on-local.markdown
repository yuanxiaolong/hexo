---
layout: post
title: "install sqoop on local"
date: 2014-08-17 17:15:58 +0800
comments: true
categories: sqoop
tags: sqoop
share: true
description: 单机安装sqoop用于学习
toc: true
---

介绍一下单机安装sqoop的过程

<!--more-->

sqoop（sql to hadoop），用于数据交换。即可以从关系型DB如mysql，导入到hdfs、hive、hbase。当然也有导出功能。（本文只示例导入hdfs、hbase，导出等只是参数不同而已）


## 环境准备

*  hadoop（1.2.1）
*  hive （0.13.1）
*  hbase （0.98.4-hadoop1）
*  mysql （5.6.16）

<font color="#7c837f">注：括号里的是我本机的实验环境，并非必要条件</font>

---

## 安装过程

1.下载sqoop，可以在[<font color="#6868b4">这里下载</font>](http://sqoop.apache.org/)

2.解压sqoop，并修改配置。

先执行如下命令，创建配置文件

```
xiaolongyuan@xiaolongdeMacBook-Air conf$ pwd
/Users/xiaolongyuan/Documents/sqoop-1.4.4/conf
xiaolongyuan@xiaolongdeMacBook-Air conf$ cp sqoop-env-template.sh   sqoop-env.sh
```

再编辑配置文件<font color="#7c837f">(最下面zookeeper配置，我们选择hbase的默认自带管理)</font>

``` bash sqoop-env.sh
#Set path to where bin/hadoop is available
export HADOOP_COMMON_HOME=/Users/xiaolongyuan/Documents/hadoop-1.2.1

#Set path to where hadoop-*-core.jar is available
export HADOOP_MAPRED_HOME=/Users/xiaolongyuan/Documents/hadoop-1.2.1

#set the path to where bin/hbase is available
export HBASE_HOME=/Users/xiaolongyuan/Documents/hbase-0.98.4-hadoop1

#Set the path to where bin/hive is available
export HIVE_HOME=/Users/xiaolongyuan/Documents/hive-0.13.1

#Set the path for where zookeper config dir is
export ZOOCFGDIR=/Users/xiaolongyuan/Documents/hbase-0.98.4-hadoop1/lib
```

修改配置 <font color="green"> bin/configure-sqoop </font>，注释掉有关 HCAT_HOME 的shell脚本，如果安装了HCatalog，则不需要注释

如下：

``` bash configure-sqoop
#if [ -z "${HCAT_HOME}" ]; then
#  HCAT_HOME=/usr/lib/hcatalog
#fi

## Moved to be a runtime check in sqoop.
#if [ ! -d "${HCAT_HOME}" ]; then
#  echo "Warning: $HCAT_HOME does not exist! HCatalog jobs will fail."
#  echo 'Please set $HCAT_HOME to the root of your HCatalog installation.'
#fi

# Add HCatalog to dependency list
#if [ -e "${HCAT_HOME}/bin/hcat" ]; then
#  TMP_SQOOP_CLASSPATH=${SQOOP_CLASSPATH}:`${HCAT_HOME}/bin/hcat -classpath`
#  if [ -z "${HIVE_CONF_DIR}" ]; then
#    TMP_SQOOP_CLASSPATH=${TMP_SQOOP_CLASSPATH}:${HIVE_CONF_DIR}
#  fi
#  SQOOP_CLASSPATH=${TMP_SQOOP_CLASSPATH}
#fi
```

3.添加需要数据交换的数据源驱动，例如数据源是mysql的话，需要添加 [<font color="#6868b4">mysql-connector-java-5.1.25-bin.jar</font>](http://yun.baidu.com/share/link?shareid=903445453&uk=3826203270)
到sqoop的lib下

---

## 运行

1.先启动hadoop

2.如果需要将mysql的数据导入到hive、hbase里的话，则需要启动hive hbase

``` bash
xiaolongyuan@localhost$ sqoop
Warning: $HADOOP_HOME is deprecated.

Try 'sqoop help' for usage.
```

3.在mysql里对需要使用的用户授权，使其可以从任意主机上访问mysql

``` bash
mysql> grant all privileges on *.* to 'hadoop'@'%' identified by 'hadoop' with grant option;
Query OK, 0 rows affected (0.00 sec)
```

4.mysql上此时有数据如下：
![](/images/sqoop/20140817/mysql.png)

---

### <font color="#8a345d" size="5"> mysql --> sqoop --> hdfs </font>

1.运行如下命令

-m 1 表示 限制map的数量是1个，由于我们是单机且数量少，所以用此。

``` bash
sqoop import --connect jdbc:mysql://127.0.0.1:3306/test --username hadoop --password hadoop --table student -m 1

```

2.在hdfs里查看

``` bash
xiaolongyuan@xiaolongdeMacBook-Air bin$ hadoop dfs -ls
Warning: $HADOOP_HOME is deprecated.
Found 3 items
drwxr-xr-x   - xiaolongyuan supergroup          0 2014-06-29 23:02 /user/xiaolongyuan/in
drwxr-xr-x   - xiaolongyuan supergroup          0 2014-06-08 14:47 /user/xiaolongyuan/out
drwxr-xr-x   - xiaolongyuan supergroup          0 2014-08-17 12:19 /user/xiaolongyuan/student

xiaolongyuan@xiaolongdeMacBook-Air bin$ hadoop dfs -ls student/
Warning: $HADOOP_HOME is deprecated.
Found 3 items
-rw-r--r--   1 xiaolongyuan supergroup          0 2014-08-17 12:19 /user/xiaolongyuan/student/_SUCCESS
drwxr-xr-x   - xiaolongyuan supergroup          0 2014-08-17 12:19 /user/xiaolongyuan/student/_logs
-rw-r--r--   1 xiaolongyuan supergroup         34 2014-08-17 12:19 /user/xiaolongyuan/student/part-m-00000

xiaolongyuan@xiaolongdeMacBook-Air bin$ hadoop dfs -cat student/part-m-00000
Warning: $HADOOP_HOME is deprecated.
1,tom,12,class1
2,kitty,16,class2
```

---

### <font color="#8a345d" size="5"> mysql --> sqoop --> hive </font>

1.先创建hive数据库

``` bash
xiaolongyuan@xiaolongdeMacBook-Air flume-1.5.0$ hive

Logging initialized using configuration in jar:file:/Users/xiaolongyuan/Documents/hive-0.13.1/lib/hive-common-0.13.1.jar!/hive-log4j.properties
hive (default)> create database test;
OK
Time taken: 0.765 seconds
```

2.执行 <font color="#7e986c">（如果hdfs下有之前导入过的student目录，需要先删除，否则会报文件夹已存在的错误）</font>

``` bash
sqoop import --connect jdbc:mysql://127.0.0.1:3306/test --username hadoop --password hadoop --table student --hive-import --hive-database test --hive-delims-replacement '\t' -m 1
```

3.查看hive里的数据<font color="#7c837f"> (我配置了hive显示表头，database_name和tab_name都是表头，不是数据) </font>

``` bash
xiaolongyuan@xiaolongdeMacBook-Air hive-0.13.1$ hive

Logging initialized using configuration in jar:file:/Users/xiaolongyuan/Documents/hive-0.13.1/lib/hive-common-0.13.1.jar!/hive-log4j.properties

hive (default)> show databases;
OK
database_name
default
test
Time taken: 0.736 seconds, Fetched: 2 row(s)

hive (default)> use test;
OK
Time taken: 0.037 seconds

hive (test)> show tables;
OK
tab_name
student
Time taken: 0.039 seconds, Fetched: 1 row(s)

hive (test)> select * from student;
OK
student.s_no	student.s_name	student.s_age	student.s_class
1	tom	12	class1
2	kitty	16	class2
Time taken: 0.508 seconds, Fetched: 2 row(s)
```

---

### <font color="#8a345d" size="5"> mysql --> sqoop --> hbase </font>

1.执行命令，导入数据到hbase

``` bash
sqoop import --connect jdbc:mysql://127.0.0.1:3306/test --username hadoop --password hadoop --table student --hbase-create-table --hbase-table student --column-family s_info --hbase-row-key s_no -m 1
```

2.查看数据

``` bash
xiaolongyuan@xiaolongdeMacBook-Air bin$ start-hbase.sh
localhost: starting zookeeper, logging to /Users/xiaolongyuan/Documents/hbase-0.98.4-hadoop1/bin/../logs/hbase-xiaolongyuan-zookeeper-xiaolongdeMacBook-Air.local.out
starting master, logging to /Users/xiaolongyuan/Documents/hbase-0.98.4-hadoop1//logs/hbase-xiaolongyuan-master-xiaolongdeMacBook-Air.local.out
localhost: starting regionserver, logging to /Users/xiaolongyuan/Documents/hbase-0.98.4-hadoop1/bin/../logs/hbase-xiaolongyuan-regionserver-xiaolongdeMacBook-Air.local.out

xiaolongyuan@xiaolongdeMacBook-Air bin$ jps
81678 Jps
79845 TaskTracker
79673 SecondaryNameNode
81534 HMaster
79746 JobTracker
79573 DataNode
81488 HQuorumPeer
79474 NameNode
81645 HRegionServer

xiaolongyuan@xiaolongdeMacBook-Air bin$ hbase shell
HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 0.98.4-hadoop1, r890e852ce1c51b71ad180f626b71a2a1009246da, Mon Jul 14 18:54:31 PDT 2014

hbase(main):001:0> scan 'student'
2014-08-17 20:21:36.166 java[81679:1903] Unable to load realm info from SCDynamicStore
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/Users/xiaolongyuan/Documents/hbase-0.98.4-hadoop1/lib/slf4j-log4j12-1.6.4.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/Users/xiaolongyuan/Documents/hadoop-1.2.1/lib/slf4j-log4j12-1.4.3.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
ROW                                                  COLUMN+CELL
 1                                                   column=s_info:s_age, timestamp=1408250793626, value=12
 1                                                   column=s_info:s_class, timestamp=1408250793626, value=class1
 1                                                   column=s_info:s_name, timestamp=1408250793626, value=tom
 2                                                   column=s_info:s_age, timestamp=1408250793626, value=16
 2                                                   column=s_info:s_class, timestamp=1408250793626, value=class2
 2                                                   column=s_info:s_name, timestamp=1408250793626, value=kitty
2 row(s) in 0.3350 seconds

hbase(main):002:0>
```
