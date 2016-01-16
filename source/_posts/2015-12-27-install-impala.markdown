---
layout: post
title: "install impala"
date: 2015-12-27 18:13:36 +0800
comments: true
categories: impala
tags: impala
description: impala安装配置
toc: true
---
介绍impala的 rpm包安装形式，利用impala 2.2.0

<!--more-->

impala 是一个数据查询引擎，类似hive，但跟hive解决的问题也有不太一样的地方，优点速度快，缺点吃内存，不适合大数据批处理

---

## 环境准备

注：本文的 hadoop hive hbase 都是通过tar 包独立安装的，并未通过rpm来安装，所以删除后面的软连接

1.下载rpm包 [<font color="#2798a2">http://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/5.4.0/RPMS/x86_64/</font>](http://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/5.4.0/RPMS/x86_64/)

共享我的[<font color="#2798a2">百度云</font>](http://pan.baidu.com/s/1i4nBSzF)

``` bash
bigtop-utils-0.7.0+cdh5.4.0+0-1.cdh5.4.0.p0.47.el6.noarch.rpm
impala-2.2.0+cdh5.4.0+0-1.cdh5.4.0.p0.75.el6.x86_64.rpm
impala-state-store-2.2.0+cdh5.4.0+0-1.cdh5.4.0.p0.75.el6.x86_64.rpm
impala-shell-2.2.0+cdh5.4.0+0-1.cdh5.4.0.p0.75.el6.x86_64.rpm
impala-catalog-2.2.0+cdh5.4.0+0-1.cdh5.4.0.p0.75.el6.x86_64.rpm
impala-server-2.2.0+cdh5.4.0+0-1.cdh5.4.0.p0.75.el6.x86_64.rpm
impala-udf-devel-2.2.0+cdh5.4.0+0-1.cdh5.4.0.p0.75.el6.x86_64.rpm
impala-debuginfo-2.2.0+cdh5.4.0+0-1.cdh5.4.0.p0.75.el6.x86_64.rpm (可不装)
```

2.安装，建议上面的顺序

``` bash
sudo rpm -ivh impala-2.2.0+cdh5.4.0+0-1.cdh5.4.0.p0.75.el6.x86_64.rpm  --nodeps --force
```

由于我的hadoop是通过tar包安装的，所以需要强制安装此rpm，其余正常安装

3.查看一下哪些有impala目录

``` bash
[xiaolong@node007011 impala-rpm]$ sudo find / -name impala
/var/log/impala
/var/run/impala
/var/lib/impala
/var/lib/alternatives/impala
/usr/lib/impala
/etc/default/impala
/etc/impala
/etc/alternatives/impala

```

impala默认的安装目录为 `/usr/lib/impala`，jar包地址为`/usr/lib/impala/lib/`

4.删除默认的不存在软连接，并添加新连接。

``` bash
sudo rm -rf /usr/lib/impala/lib/avro*.jar
sudo rm -rf /usr/lib/impala/lib/hadoop-*.jar
sudo rm -rf /usr/lib/impala/lib/hive-*.jar
sudo rm -rf /usr/lib/impala/lib/hbase-*.jar
sudo rm -rf /usr/lib/impala/lib/parquet-hadoop-bundle.jar
sudo rm -rf /usr/lib/impala/lib/sentry-*.jar
sudo rm -rf /usr/lib/impala/lib/zookeeper.jar
sudo rm -rf /usr/lib/impala/lib/libhadoop.so
sudo rm -rf /usr/lib/impala/lib/libhadoop.so.1.0.0
sudo rm -rf /usr/lib/impala/lib/libhdfs.so
sudo rm -rf /usr/lib/impala/lib/libhdfs.so.0.0.0
```

5.建立新连接，建议利用脚本运行，自己设一下各个home即可，软连后的名字不能改。

``` bash
sudo ln -s $HBASE_HOME/lib/avro-1.7.6-cdh5.4.0.jar /usr/lib/impala/lib/avro.jar
sudo ln -s $HADOOP_HOME/share/hadoop/common/lib/hadoop-annotations-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-annotations.jar
sudo ln -s $HADOOP_HOME/share/hadoop/common/lib/hadoop-auth-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-auth.jar
sudo ln -s $HADOOP_HOME/share/hadoop/tools/lib/hadoop-aws-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-aws.jar
sudo ln -s $HADOOP_HOME/share/hadoop/common/hadoop-common-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-common.jar
sudo ln -s $HADOOP_HOME/share/hadoop/hdfs/hadoop-hdfs-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-hdfs.jar
sudo ln -s $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-client-common-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-mapreduce-client-common.jar
sudo ln -s $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-client-core-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-mapreduce-client-core.jar
sudo ln -s $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-mapreduce-client-jobclient.jar
sudo ln -s $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-client-shuffle-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-mapreduce-client-shuffle.jar
sudo ln -s $HADOOP_HOME/share/hadoop/yarn/hadoop-yarn-api-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-yarn-api.jar
sudo ln -s $HADOOP_HOME/share/hadoop/yarn/hadoop-yarn-client-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-yarn-client.jar
sudo ln -s $HADOOP_HOME/share/hadoop/yarn/hadoop-yarn-common-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-yarn-common.jar
sudo ln -s $HADOOP_HOME/share/hadoop/yarn/hadoop-yarn-server-applicationhistoryservice-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-yarn-server-applicationhistoryservice.jar
sudo ln -s $HADOOP_HOME/share/hadoop/yarn/hadoop-yarn-server-common-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-yarn-server-common.jar
sudo ln -s $HADOOP_HOME/share/hadoop/yarn/hadoop-yarn-server-nodemanager-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-yarn-server-nodemanager.jar
sudo ln -s $HADOOP_HOME/share/hadoop/yarn/hadoop-yarn-server-resourcemanager-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-yarn-server-resourcemanager.jar
sudo ln -s $HADOOP_HOME/share/hadoop/yarn/hadoop-yarn-server-web-proxy-2.6.0-cdh5.4.0.jar /usr/lib/impala/lib/hadoop-yarn-server-web-proxy.jar

sudo ln -s $HBASE_HOME/lib/hbase-annotations-1.0.0-cdh5.4.0.jar /usr/lib/impala/lib/hbase-annotations.jar
sudo ln -s $HBASE_HOME/lib/hbase-client-1.0.0-cdh5.4.0.jar /usr/lib/impala/lib/hbase-client.jar
sudo ln -s $HBASE_HOME/lib/hbase-common-1.0.0-cdh5.4.0.jar /usr/lib/impala/lib/hbase-common.jar
sudo ln -s $HBASE_HOME/lib/hbase-protocol-1.0.0-cdh5.4.0.jar /usr/lib/impala/lib/hbase-protocol.jar

sudo ln -s $HIVE_HOME/lib/hive-ant-1.1.0-cdh5.4.0.jar /usr/lib/impala/lib/hive-ant.jar
sudo ln -s $HIVE_HOME/lib/hive-beeline-1.1.0-cdh5.4.0.jar /usr/lib/impala/lib/hive-beeline.jar
sudo ln -s $HIVE_HOME/lib/hive-common-1.1.0-cdh5.4.0.jar /usr/lib/impala/lib/hive-common.jar
sudo ln -s $HIVE_HOME/lib/hive-exec-1.1.0-cdh5.4.0.jar /usr/lib/impala/lib/hive-exec.jar
sudo ln -s $HIVE_HOME/lib/hive-hbase-handler-1.1.0-cdh5.4.0.jar /usr/lib/impala/lib/hive-hbase-handler.jar
sudo ln -s $HIVE_HOME/lib/hive-metastore-1.1.0-cdh5.4.0.jar /usr/lib/impala/lib/hive-metastore.jar
sudo ln -s $HIVE_HOME/lib/hive-serde-1.1.0-cdh5.4.0.jar /usr/lib/impala/lib/hive-serde.jar
sudo ln -s $HIVE_HOME/lib/hive-service-1.1.0-cdh5.4.0.jar /usr/lib/impala/lib/hive-service.jar
sudo ln -s $HIVE_HOME/lib/hive-shims-common-1.1.0-cdh5.4.0.jar /usr/lib/impala/lib/hive-shims-common.jar
sudo ln -s $HIVE_HOME/lib/hive-shims-1.1.0-cdh5.4.0.jar /usr/lib/impala/lib/hive-shims.jar
sudo ln -s $HIVE_HOME/lib/hive-shims-scheduler-1.1.0-cdh5.4.0.jar /usr/lib/impala/lib/hive-shims-scheduler.jar

sudo ln -s $HADOOP_HOME/lib/native/libhadoop.so /usr/lib/impala/lib/libhadoop.so
sudo ln -s $HADOOP_HOME/lib/native/libhadoop.so.1.0.0 /usr/lib/impala/lib/libhadoop.so.1.0.0
sudo ln -s $HADOOP_HOME/lib/native/libhdfs.so /usr/lib/impala/lib/libhdfs.so
sudo ln -s $HADOOP_HOME/lib/native/libhdfs.so.0.0.0 /usr/lib/impala/lib/libhdfs.so.0.0.0

sudo ln -s $HIVE_HOME/lib/parquet-hadoop-bundle-1.5.0-cdh5.4.0.jar /usr/lib/impala/lib/parquet-hadoop-bundle.jar

sudo ln -s $SENTRY_HOME/lib/sentry-binding-hive-1.4.0-cdh5.4.0.jar /usr/lib/impala/lib/sentry-binding-hive.jar
sudo ln -s $SENTRY_HOME/lib/sentry-core-common-1.4.0-cdh5.4.0.jar /usr/lib/impala/lib/sentry-core-common.jar
sudo ln -s $SENTRY_HOME/lib/sentry-core-model-db-1.4.0-cdh5.4.0.jar /usr/lib/impala/lib/sentry-core-model-db.jar
sudo ln -s $SENTRY_HOME/lib/sentry-policy-common-1.4.0-cdh5.4.0.jar /usr/lib/impala/lib/sentry-policy-common.jar
sudo ln -s $SENTRY_HOME/lib/sentry-policy-db-1.4.0-cdh5.4.0.jar /usr/lib/impala/lib/sentry-policy-db.jar
sudo ln -s $SENTRY_HOME/lib/sentry-provider-cache-1.4.0-cdh5.4.0.jar /usr/lib/impala/lib/sentry-provider-cache.jar
sudo ln -s $SENTRY_HOME/lib/sentry-provider-common-1.4.0-cdh5.4.0.jar /usr/lib/impala/lib/sentry-provider-common.jar
sudo ln -s $SENTRY_HOME/lib/sentry-provider-db-1.4.0-cdh5.4.0.jar /usr/lib/impala/lib/sentry-provider-db.jar
sudo ln -s $SENTRY_HOME/lib/sentry-provider-file-1.4.0-cdh5.4.0.jar /usr/lib/impala/lib/sentry-provider-file.jar

sudo ln -s $ZOOKEEPER_HOME/zookeeper-3.4.5-cdh5.3.3.jar /usr/lib/impala/lib/zookeeper.jar
```

---

## 配置

1.修改`/etc/default/bigtop-utils`

``` bash
export JAVA_HOME=/usr/local/datacenter/jdk1.7.0_79
```

2.修改 `/etc/default/impala` 默认文件，除了修改环境，还需要修改启动参数，显示传递给 statestored 和 catalogd 它们各自需要的ip和port ，不然都默认 localhost ，分开部署会有问题的。

``` bash
IMPALA_CATALOG_SERVICE_HOST=192.168.7.11
IMPALA_STATE_STORE_HOST=192.168.7.11

IMPALA_CATALOG_ARGS=" -log_dir=${IMPALA_LOG_DIR} -catalog_service_host=${IMPALA_CATALOG_SERVICE_HOST} -state_store_host=${IMPALA_STATE_STORE_HOST}"
IMPALA_STATE_STORE_ARGS=" -log_dir=${IMPALA_LOG_DIR} -state_store_host=${IMPALA_STATE_STORE_HOST} -state_store_port=${IMPALA_STATE_STORE_PORT}  -catalog_service_host=${IMPALA_CATALOG_SERVICE_HOST} -catalog_service_port=${CATALOG_SERVICE_PORT} "

MYSQL_CONNECTOR_JAR=/usr/local/datacenter/hive/lib/mysql-connector-java-5.1.34.jar
```

3.根据实际环境修改impala相关脚本文件 `/etc/init.d/impala-state-store`、`/etc/init.d/impala-server`、`/etc/init.d/impala-catalog`，修改其中两处跟用户相关的地方

``` bash
###编者注：这里默认是impala
SVC_USER="hadoop"
###编者注：这里默认是impala
install -d -m 0755 -o hadoop -g hadoop /var/run/impala 1>/dev/null 2>&1 || :
```

为保险起见 手工建立日志目录并修改权限 `mkdir /var/log/impala && chmod -R 777 /var/log/impala`

4.修改 hdfs-site.xml 这一处就够了

``` xml hdfs-site.xml
<property>
    <name>dfs.client.read.shortcircuit</name>
    <value>true</value>
</property>
<property>
    <name>dfs.domain.socket.path</name>
    <value>/var/run/hadoop-hdfs/dn</value>
</property>
<property>
  <name>dfs.datanode.hdfs-blocks-metadata.enabled</name>
  <value>true</value>
</property>
<property>
   <name>dfs.client.use.legacy.blockreader.local</name>
   <value>false</value> ###编者注：这里需要为false ，官网文档错了
</property>
<property>
   <name>dfs.datanode.data.dir.perm</name>
   <value>750</value>
</property>
<property>
   <name>dfs.block.local-path-access.user</name>
   <value>hadoop</value>
</property>
<property>
   <name>dfs.client.file-block-storage-locations.timeout</name>
   <value>3000</value>
</property>
```

官网的文档有一处错误，`dfs.client.use.legacy.blockreader.local` 属性应该为 false，文档中是 true

[http://www.cloudera.com/content/www/en-us/documentation/archive/impala/2-x/2-1-x/topics/impala_config_performance.html?scroll=config_performance ](http://www.cloudera.com/content/www/en-us/documentation/archive/impala/2-x/2-1-x/topics/impala_config_performance.html?scroll=config_performance)

注意每个节点创建 `hdfs-site.xml` 里指定的 `/var/run/hdfs-sockets/dn` 权限

5.impalad的配置文件路径由环境变量IMPALA_CONF_DIR指定，默认为 `/etc/impala/conf`，拷贝配置或建立软连接 的`hive-site.xml`、`core-site.xml`、`hdfs-site.xml`、`hbase-site.xml`文件至`/etc/impala/conf`目录下。

``` bash
sudo ln -s /usr/local/datacenter/hadoop/etc/hadoop/core-site.xml /etc/impala/conf/
sudo ln -s /usr/local/datacenter/hadoop/etc/hadoop/hdfs-site.xml /etc/impala/conf/
sudo ln -s /usr/local/datacenter/hive/conf/hive-site.xml /etc/impala/conf/
sudo ln -s /usr/local/datacenter/hbase/conf/hbase-site.xml /etc/impala/conf/
```

---

## 启动

启动是最诡异的地方，由于官网建议statestored和catalogd在同一台机器上，所以ip一样，但端口不一样，然后利用service启动就只能启动一个，可以通过 pid file 看到是同一个进程.

1.在 `statestored` 和 `catalogd` 机器上启动「impala主节点」， 建议启动时，一个利用service 启动一个利用手工启动

``` bash
cd /usr/lib/impala
nohup /usr/lib/impala/sbin/statestored -log_dir=/var/log/impala -state_store_host=192.168.7.11 -state_store_port=24000 -catalog_service_host=192.168.7.11:26000 -catalog_service_port=26000 &

sudo service impala-catalog start
```

碰见问题 ,缺少jdk so文件，缺少classpath impala ，所以修改环境变量

``` bash
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib:/usr/local/datacenter/jdk1.7.0_79/jre/lib/amd64:/usr/local/datacenter/jdk1.7.0_79/jre/lib/amd64/server:/usr/lib/impala/lib/

export CLASSPATH=$CLASSPATH:$JAVA_HOME/lib:$JAVA_HOME/jre/lib:/usr/lib/impala/lib
```

2.在「impala从节点」上，启动 `serverd`

``` bash
sudo service  impala-server start
```

3.访问主节点 25010 端口，得到statestore web ui ，其中比较有参考意义的，logs 和 varz 里面有环境变量的信息
![](/images/impala/20151227/1.png)


4.访问从节点的 25000 ，得到 serverd 的 web ui
![](/images/impala/20151227/2.png)

---

## 执行

1.进入`impala-shell`，即连接的本机的 impala-server

``` bash
[root@node007012 ~]# impala-shell
Starting Impala Shell without Kerberos authentication
Connected to node007012:21000
Server version: impalad version 2.2.0-cdh5 RELEASE (build 2ffd73a4255cefd521362ffe1cfb37463f67f75c)

```

2.同步hive元信息，以便感知已存在的hive库表

``` bash
[node007012:21000] > invalidate metadata;
Query: invalidate metadata

Fetched 0 row(s) in 3.98s
```

3.执行一下ddl语句，看是否正常

``` bash
[node007012:21000] > SHOW DATABASES;
Query: show DATABASES
+------------------+
| name             |
+------------------+
| _impala_builtins |
| default          |
| test             |
+------------------+
Fetched 3 row(s) in 0.01s
[node007012:21000] > USE test;
Query: use test
[node007012:21000] > show tables;
Query: show tables
+--------------+
| name         |
+--------------+
| g_info       |
| g_info_pa    |
| g_info_sqoop |
| pokes        |
| pokes_pa     |
| sales_order  |
| t1           |
| t2           |
| user_sqoop   |
+--------------+
Fetched 9 row(s) in 0.01s
```

4.验证查询语句，其中g_info表是textfile，g_info_pa 是parquetfile 的，可见效率差异性

``` bash
[node007012:21000] > select count(*) from g_info;
Query: select count(*) from g_info
+----------+
| count(*) |
+----------+
| 1222023  |
+----------+
WARNINGS: Unknown disk id.  This will negatively affect performance. Check your hdfs settings to enable block location metadata. (1 of 8 similar)

Fetched 1 row(s) in 44.13s
[node007012:21000] > select count(*) from g_info_pa;
Query: select count(*) from g_info_pa
+----------+
| count(*) |
+----------+
| 1192760  |
+----------+
WARNINGS: Unknown disk id.  This will negatively affect performance. Check your hdfs settings to enable block location metadata. (1 of 4 similar)

Fetched 1 row(s) in 4.73s
```

这个警告日志里报错的是，待查

``` bash
W1226 19:14:35.052875 30015 DomainSocketFactory.java:167] error creating DomainSocket
Java exception follows:
java.net.ConnectException: connect(2) error: Connection refused when trying to connect to '/var/run/hdfs-sockets/dn'
```

参考：

* [http://blog.csdn.net/zhong_han_jun/article/details/45563505](http://blog.csdn.net/zhong_han_jun/article/details/45563505)

* [http://www.cnblogs.com/chenz/articles/3629698.html](http://www.cnblogs.com/chenz/articles/3629698.html)

* [http://www.cloudera.com/content/cloudera/zh-CN/documentation/core/v5-3-x/topics/impala_config_options.html](http://www.cloudera.com/content/cloudera/zh-CN/documentation/core/v5-3-x/topics/impala_config_options.html)

* [http://stackoverflow.com/questions/23469758/aws-emr-impala-daemon-issue](http://stackoverflow.com/questions/23469758/aws-emr-impala-daemon-issue)
