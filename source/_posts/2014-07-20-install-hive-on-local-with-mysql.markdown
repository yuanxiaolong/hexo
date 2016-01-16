---
layout: post
title: "install hive on local with mysql"
date: 2014-07-20 01:40:49 +0800
comments: true
categories: [hive]
tags: hive
share: true
description: 安装hive
toc: true
---
介绍一下本地安装 Hive 0.13.1 ,并元数据存放到本地mysql上

<!--more-->

---

## 准备工作
1.  本地安装了hadoop 、mysql
2.  下载 [<font color="#3756b2">apache-hive-0.13.1-bin.tar.gz</font>](http://mirrors.hust.edu.cn/apache/hive/stable/)

---

## 开始安装
1.解压 apache-hive-0.13.1-bin.tar.gz

``` bash
tar xzf ./apache-hive-0.13.1-bin.tar.gz
```

2.复制几个配置文件

``` bash
cp hive-default.xml.template hive-site.xml
cp hive-env.sh.template hive-env.sh
```

3.用vi修改这2个文件的内容

``` bash hive-env.sh
# Set HADOOP_HOME to point to a specific hadoop install directory
HADOOP_HOME=$HADOOP_HOME

```
<font color="#818a8b" size="3">(注:如果你的PATH下有HADOOP_HOME变量才可以这样设置，否则应该指出hadoop的安装路径)</font>

``` bash hive-site.xml
<property>
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:mysql://localhost:3306/hive?characterEncoding=UTF-8</value>
  <description>JDBC connect string for a JDBC metastore</description>
</property>

<property>
  <name>javax.jdo.option.ConnectionDriverName</name>
  <value>com.mysql.jdbc.Driver</value>
  <description>Driver class name for a JDBC metastore</description>
</property>

<property>
  <name>javax.jdo.option.ConnectionUserName</name>
  <value>hive</value>
  <description>username to use against metastore database</description>
</property>

<property>
  <name>javax.jdo.option.ConnectionPassword</name>
  <value>hive</value>
  <description>password to use against metastore database</description>
</property>
```

<font color="#800b7e">解释一下这4个变量的意思</font>

* 第一个： 说明是让hive将元数据存放到mysql中，且用mysql连接串来连接
* 第二个： 连接mysql的驱动 ([<font color="#3756b2">这里下载</font>](http://pan.baidu.com/s/1dDnaw05) 并把它放到hive的lib目录下)
* 第三个： 连接mysql用户名
* 第四个： 连接mysql密码

因此，我们需要先在mysql上，新建一个用户和密码（我这里都是叫hive），同时<font color="red">用hive这个用户新建一个叫hive的数据库</font>

然后修改mysql hive 这个数据库字符集

``` bash
alter database hive character set latin1;
```

---

## 启动hive

在hive的bin目录下执行

``` bash
sh ./hive
```

whoops~~~~报错了,如下：

``` bash
xiaolongyuan@xiaolongdeMacBook-Air bin$ ./hive
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/hadoop/hive/conf/HiveConf
	at java.lang.Class.forName0(Native Method)
	at java.lang.Class.forName(Class.java:270)
	at org.apache.hadoop.util.RunJar.main(RunJar.java:153)
Caused by: java.lang.ClassNotFoundException: org.apache.hadoop.hive.conf.HiveConf
	at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
	at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
	at java.security.AccessController.doPrivileged(Native Method)
	at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
	... 3 more
```

<font color="#e14a40">解决方法: </font>

修改你的Hadoop配置hadoop-env.sh，在后面加上 *<font color="green">$HADOOP_CLASSHPATH</font>*

``` bash hadoop-env.sh
# Extra Java CLASSPATH elements.  Optional.
export HADOOP_CLASSPATH=.:$HADOOP_CLASSPATH
```

再次运行，whoops~~~又报错了。。。

``` bash
Caused by: MetaException(message:Got exception: java.net.ConnectException Call to
localhost/127.0.0.1:9000 failed on connection exception: java.net.ConnectException:
 Connection refused)
```

这次因为由于Hadoop没有启动,启动hadoop即可。然后再次运行hive，既可以成功。

``` bash
xiaolongyuan@xiaolongdeMacBook-Air bin$ ./hive
14/07/20 01:38:56 WARN conf.HiveConf: DEPRECATED: hive.metastore.ds.retry.* no longer has any effect.  Use hive.hmshandler.retry.* instead

Logging initialized using configuration in jar:file:/Users/xiaolongyuan/Documents/hive-0.13.1/lib/hive-common-0.13.1.jar!/hive-log4j.properties
hive> show databases;
OK
default
Time taken: 0.73 seconds, Fetched: 1 row(s)
hive>
```

我们还可以看到hive的元数据已经存到Mysql里了
![](/images/hadoop/hive-local-mysql.png)
