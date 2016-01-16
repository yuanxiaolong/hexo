---
layout: post
title: "tez on yarn"
date: 2015-12-27 17:47:23 +0800
comments: true
categories: tez
tags: tez
description: tez on yarn
toc: true
---
介绍 tez 框架运行在 yarn上

<!--more-->

tez是一个底层框架，跟mr不同之处，是在于 tez 将mr处理进行了优化，然后再跑优化后的 mr DAG，因此效率会快一些

---

## 环境准备

1.下载源码，建议0.6.0 以上，因为Tez以0.6.0版本为分隔点，很多功能例如web ui 都仅支持 0.6.0 以上的版本

[<font color="#2798a2">官网网址 </font>](http://tez.apache.org/releases/)   （tez 没有打包好的二进制可运行包，需要自己通过源码编译）

[<font color="#2798a2">我共享的百度云地址 </font>](http://pan.baidu.com/s/1nukfT49)

2.jdk1.7 和 maven3 , 也共享一下jdk1.7 的[<font color="#2798a2">地址</font>](http://pan.baidu.com/s/1jGPTsu6)

3.protobuf 2.5.0 编译安装（如果已有则跳过）, protobuf 的代码在 googlecode 上我共享一下 [<font color="#2798a2">我的百度云地址</font>](http://pan.baidu.com/s/1eRisu9G)

``` bash
sudo ./configure && sudo make && sudo make install
```

4.安装 `npm  install phantomjs -g` 以便构建 tez-ui.war (npm是 nodejs的包管理器，如果没有npm命令，安装一个较新版本的nodejs就可以了)

---

## 编译

1.编译源码，进入 tez-src 目录，根据自己的hadoop版本，修改顶层pom.xml 里 `hadoop.version` 然后打包

``` bash
mvn clean package -DskipTests=true -Dmaven.javadoc.skip=true
```

2.打包好后，把tez-dist/target 下的 tez-0.7.0.tar.gz 上传到 hdfs 上的一个自定义目录里

3.下载tomcat 7.0.67 或以上，并解压，把 `tez-dist/target/tez-0.7.0/tez-ui-0.7.0.war` 解压到 `$TOMCAT_HOME/webapp` 下，形成 `$TOMCAT_HOME/webapp/tez-ui` 这样的结构（可以删除tomcat自带的webapp下的项目，也可以通过 `$TOMCAT_HOME/conf/server.xml` 指定CONTEXT 配到指定 tez-ui-0.7.0.war解压后的目录）

4.编辑`$TOMCAT_HOME/conf/web.xml`，并添加以下内容，不然访问context会报404

``` xml
 <filter>
   <filter-name>CorsFilter</filter-name>
   <filter-class>org.apache.catalina.filters.CorsFilter</filter-class>
 </filter>
 <filter-mapping>
   <filter-name>CorsFilter</filter-name>
   <url-pattern>/*</url-pattern>
 </filter-mapping>
```

5.把编译好的 tez-0.7.0.tar.gz 分发到所有节点上，并解压到路径一致的目录里，以便设置 TEZ_HOME 环境变量

---

## 配置

1.在 `$HADOOP_HOME/etc/hadoop/` 目录下创建 `tez-site.xml` 文件并编辑 `tez-site.xml` 文件内容如下

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
    <property>
      <name>tez.lib.uris</name>
      <value>${fs.defaultFS}/share/tez-0.7.0.tar.gz</value>
    </property>
    <property>
      <name>tez.history.logging.service.class</name>
      <value>org.apache.tez.dag.history.logging.ats.ATSHistoryLoggingService</value>
    </property>
    <property>
       <description>URL for where the Tez UI is hosted</description>
       <name>tez.tez-ui.history-url.base</name>
       <value>http://127.0.0.1:8080/tez-ui/</value>
    </property>
</configuration>
```

注：这里的127.0.0.1 是tomcat的ip地址，同时8080是tomcat的端口（也可以通过tomcat指定为其他端口），tez-ui是tomcat的 context 。而 `tez.lib.uris` 是刚才上传到hdfs的文件

2.编辑 hadoop-env.sh

``` bash
export TEZ_HOME=/usr/local/datacenter/tez-0.7.0
for jar in `ls $TEZ_HOME |grep jar`; do
    export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:$TEZ_HOME/$jar
done
for jar in `ls $TEZ_HOME/lib`; do
    export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:$TEZ_HOME/lib/$jar
done
```

3.编辑 `mapred-site.xml` ，修改为 tez 类型

``` xml
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn-tez</value>
</property>
```

4.修改 `yarn-site.xml` 添加如下

``` xml
<property>
  <description>Indicate to clients whether Timeline service is enabled or not. If enabled, the TimelineClient library used by end-users will post entities and events to the Timeline server.</description>
      <name>yarn.timeline-service.enabled</name>
      <value>true</value>
</property>
<property>
      <description>The hostname of the Timeline service web application.</description>
      <name>yarn.timeline-service.hostname</name>
      <value>127.0.0.1</value>
</property>
<property>
      <description>Enables cross-origin support (CORS) for web services where cross-origin web response headers are needed. For example, javascript making a web services request to the timeline server.</description>
      <name>yarn.timeline-service.http-cross-origin.enabled</name>
      <value>true</value>
</property>
<property>
      <description>Publish YARN information to Timeline Server</description>
      <name>yarn.resourcemanager.system-metrics-publisher.enabled</name>
      <value>true</value>
</property>

  <property>
    <description>Address for the Timeline server to start the RPC server.</description>
    <name>yarn.timeline-service.address</name>
    <value>${yarn.timeline-service.hostname}:10200</value>
</property>

<property>
    <description>The http address of the Timeline service web application.</description>
    <name>yarn.timeline-service.webapp.address</name>
    <value>${yarn.timeline-service.hostname}:8188</value>
</property>

<property>
    <description>The https address of the Timeline service web application.</description>
    <name>yarn.timeline-service.webapp.https.address</name>
    <value>${yarn.timeline-service.hostname}:8190</value>
</property>

<property>
    <description>Handler thread count to serve the client RPC requests.</description>
    <name>yarn.timeline-service.handler-thread-count</name>
    <value>10</value>
</property>

<property>
    <description>Enables cross-origin support (CORS) for web services where
    cross-origin web response headers are needed. For example, javascript making
    a web services request to the timeline server.</description>
    <name>yarn.timeline-service.http-cross-origin.enabled</name>
    <value>true</value>
</property>

<property>
    <description>Comma separated list of origins that are allowed for web
    services needing cross-origin (CORS) support. Wildcards (*) and patterns
    allowed</description>
    <name>yarn.timeline-service.http-cross-origin.allowed-origins</name>
    <value>*</value>
</property>

<property>
    <description>Comma separated list of methods that are allowed for web
    services needing cross-origin (CORS) support.</description>
    <name>yarn.timeline-service.http-cross-origin.allowed-methods</name>
    <value>GET,POST,HEAD,OPTIONS</value>
</property>

<property>
    <description>Comma separated list of headers that are allowed for web
    services needing cross-origin (CORS) support.</description>
    <name>yarn.timeline-service.http-cross-origin.allowed-headers</name>
    <value>X-Requested-With,Content-Type,Accept,Origin,Access-Control-Allow-Origin</value>
</property>

<property>
    <description>The number of seconds a pre-flighted request can be cached
    for web services needing cross-origin (CORS) support.</description>
    <name>yarn.timeline-service.http-cross-origin.max-age</name>
    <value>1800</value>
</property>
```

> 注：跟官网给出的文档有3处不同
>
> yarn.timeline-service.http-cross-origin.enabled 默认为false，我修改为true让其支持跨域访问
>
> yarn.timeline-service.http-cross-origin.allowed-headers 里新增了Access-Control-Allow-Origin 这种 header
>
> yarn.timeline-service.http-cross-origin.allowed-methods 里新增了 OPTIONS 方法

以下内容我没有添加，没有试验，不过在 yarn-default.xml 里可以看到，即使没有显示写 ttl 配置，也有默认值

``` xml
<property>
  <description>Indicate to ResourceManager as well as clients whether
  history-service is enabled or not. If enabled, ResourceManager starts
  recording historical data that Timelien service can consume. Similarly,
  clients can redirect to the history service when applications
  finish if this is enabled.</description>
  <name>yarn.timeline-service.generic-application-history.enabled</name>
  <value>true</value>
</property>

<property>
  <description>Store class name for history store, defaulting to file system
  store</description>
  <name>yarn.timeline-service.generic-application-history.store-class</name>
  <value>org.apache.hadoop.yarn.server.applicationhistoryservice.FileSystemApplicationHistoryStore</value>
</property>

<property>
  <description>Store class name for timeline store.</description>
  <name>yarn.timeline-service.store-class</name>
  <value>org.apache.hadoop.yarn.server.timeline.LeveldbTimelineStore</value>
</property>

<property>
  <description>Enable age off of timeline store data.</description>
  <name>yarn.timeline-service.ttl-enable</name>
  <value>true</value>
</property>

<property>
  <description>Time to live for timeline store data in milliseconds.</description>
  <name>yarn.timeline-service.ttl-ms</name>
  <value>6048000000</value>
</property>
```

![](/images/tez/20151227/1.png)

---

## 运行

1.重启 yarn 及 jobhistory

* stop-yarn.sh 和 start-yarn.sh

* yarn-daemon.sh start timelineserver 和 yarn-daemon.sh stop timelineserver

2.验证 tez 计算框架

先上传一个文本文件

``` bash
hadoop dfs -mkdir -p /test/out
hadoop dfs -put LICENSE.txt /test/LICENSE.txt
```

进入  `tez-dist/target/tez-0.7.0/` ，执行 ` hadoop jar ./tez-examples-0.7.0.jar orderedwordcount /test/LICENSE.txt /test/out `

![](/images/tez/20151227/2.png)

我碰见第一次未执行成功，是因为TEZ去找 `/bin/java` 了，可以把当前jdk做个软连接直接连接到 `/bin/java` 就可以了

启动tomcat 访问 http://127.0.0.1:8080/tez-ui

![](/images/tez/20151227/3.png)
![](/images/tez/20151227/4.png)
![](/images/tez/20151227/5.png)
![](/images/tez/20151227/6.png)


---

## tez on hive

在hive里使用tez十分方便，有2种方式

1.进入hive，然后执行 ` set hive.execution.engine=tez;` ，然后就可以正常使用

2.编写hive-site.xml，修改下面属性，这样就全局影响了

``` xml
<property>
    <name>hive.execution.engine</name>
    <value>tez</value>
    <description>
      Expects one of [mr, tez, spark].
      Chooses execution engine. Options are: mr (Map reduce, default), tez (hadoop 2 only), spark
    </description>
  </property>
```

---

## 已碰见问题

1.Failed to execute goal com.github.eirslett:frontend-maven-plugin ~ A required class was missing

``` bash
[ERROR] Failed to execute goal com.github.eirslett:frontend-maven-plugin:0.0.16:install-node-and-npm
(install node and npm) on project tez-ui: Execution install node and npm of goal
com.github.eirslett:frontend-maven-plugin:0.0.16:install-node-and-npm failed:
A required class was missing while executing
com.github.eirslett:frontend-maven-plugin:0.0.16:install-node-and-npm:org/slf4j/helpers/MarkerIgnoringBase
```

> Reason: Eirslett frontend-maven-plugin version(0.0.22 in above case) is not compatible with current maven version
> Solution: Force the expected plugging version while building tez-ui: mvn clean package -Dfrontend-maven-plugin.version=0.0.XX
                       For maven version < 3.1 the frontend-maven-plugin version has to be <= 0.0.22
                       For maven version >=3.1 the frontend-maven-plugin version has to be >= 0.0.23

maven 版本 和 插件不兼容, 修改 顶层 pom.xml

``` xml
<plugin>
      <groupId>com.github.eirslett</groupId>
      <artifactId>frontend-maven-plugin</artifactId>
      <version>0.0.23</version>
 </plugin>
```

[https://cwiki.apache.org/confluence/display/TEZ/Build+errors+and+solutions](https://cwiki.apache.org/confluence/display/TEZ/Build+errors+and+solutions)

2.网络不好，下载不下来包，重新编译几次就可以了

3.Classnotfound MRVersion 的问题，解决不掉，重新换一个hive就行了，估计是环境问题

### 后记 MRVersion的问题解决掉了

* https://issues.apache.org/jira/browse/TEZ-3030
* https://issues.apache.org/jira/browse/TEZ-3031

在 `hive-env.sh` 里添加

``` bash
export HIVE_AUX_JARS_PATH=hadoop-core-2.6.0-mr1-cdh5.4.0.jar
export HIVE_AUX_JARS_PATH=/usr/local/datacenter/phoenix/lib/hadoop-core-2.6.0-mr1-cdh5.4.0.jar:/usr/local/datacenter/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-client-core-2.6.0-cdh5.4.0.jar
```

如果通过 hue -> beeswax 查询，需要把这2个jar 放到 `HIVE_CLASSPATH` 环境变量里，并重启hiveserver2，即可

tez指定队列，需要在`tez-site.xml`里

``` xml
<property>
  <name>tez.queue.name</name>
  <value>tez</value>
</property>
```
