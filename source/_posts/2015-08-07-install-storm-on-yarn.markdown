---
layout: post
title: "install storm on yarn"
date: 2015-08-07 14:31:27 +0800
comments: true
categories: ['storm']
tags: strom
description: storm on yarn 部署
toc: true
---

storm on yarn 部署

<!--more-->

目前有 yarn ，但 storm 是利用自己资源管理的，如果把 资源统一管理，用 Yarn 或许可以减轻维护成本

---

## 环境准备

* 多台 centos6 x64 机器
 * 10.200.8.43  hadoop1 8G 154G 8CPU
 * 10.200.8.44  hadoop2 16G 154G 8CPU
 * 10.200.8.45  hadoop3 16G 1.5T 16CPU
 * 10.200.8.46  hadoop4 16G 260G 16CPU
 * 10.200.8.47  hadoop5 16G 260G 16CPU
 * 10.200.8.48  hadoop6 16G 800G 8CPU
 * 10.200.8.85  hadoop7 64G 42T 8CPU
 * 10.200.8.88  hadoop8 64G 42T 8CPU
* maven 3.1.1
* jdk7
* git

---

## 步骤

1.下载 yahoo 的 storm-on-yarn 工程 得到源代码 目录 storm-yarn-master

``` bash
git clone git@github.com:yahoo/storm-yarn.git
```

2.解压 `storm-yarn-master/lib/storm-0.9.0-wip21.zip` 到 storm-yarn-master

最终目录是

``` bash
<your dir>/storm-yarn-master
<your dir>/storm-0.9.0-wip21
```

3.配置环境变量

``` bash
export STORM_WORK=<your dir>
export STORM_HOME=$STORM_WORK
export PATH=$STORM_WORK/storm-yarn-master/bin:$STORM_WORK/storm-0.9.0-wip21/bin:$PATH
```

正常来说还要配置hadoop的环境，但我是用cloudera-manager 管理的，所以下面的配置在 Yarn 里是默认的

``` bash
export HADOOP_INSTALL=/opt/hadoop
export HADOOP_HOME=$HADOOP_INSTALL
export PATH=$PATH:$HADOOP_INSTALL/bin
export PATH=$PATH:$HADOOP_INSTALL/sbin
export HADOOP_MAPRED_HOME=$HADOOP_INSTALL
export HADOOP_COMMON_HOME=$HADOOP_INSTALL
export HADOOP_HDFS_HOME=$HADOOP_INSTALL
export YARN_HOME=$HADOOP_INSTALL
```

![](/images/storm/20150807/1.png)

4.进入工程代码，修改 pom.xml 中的hadoop版本，编译打包，打包完后会产生 storm-yarn-1.0-alpha.jar ，这个不用动

``` xml
<properties>
    <storm.version>0.9.0-wip21</storm.version>
    <hadoop.version>2.3.0</hadoop.version>
    <!--hadoop.version>2.1.0.2.0.5.0-67</hadoop.version-->
</properties>
```

``` bash
mvn package -DskipTests
```

然后将 storm-yarn-master/lib/storm.zip 放到 hdfs 中作为 storm 环境 (如果有个性需要，添加Storm工程需要的额外Jar包到storm-0.9.0-wip21的lib下，重新压缩成storm.zip文件，上传至HDFS的指定目录中)

``` bash
hadoop fs -mkdir -p /lib/storm/0.9.0-wip21
hadoop fs -put storm.zip /lib/storm/0.9.0-wip21/
```

网上有说要建 storm 用户主目录，我并没有用 storm 用户下提交任务，所以这里具体是否真的必须，不是很清楚，但还是建了一个目录

``` bash
hadoop fs -mkdir -p /user/storm
hadoop fs -chown storm /user/storm
```

5.修改storm.yaml文件 `storm-0.9.0-wip21/conf/storm.yaml` (每行开头需要有个空格 ，属性和中划线中间也是)

``` java
 storm.zookeeper.servers:
     - "10.200.8.46"
     - "10.200.8.47"
     - "10.200.8.48"

 supervisor.slots.ports:
    - 6700
    - 6701
    - 6702

 storm.local.dir: /var/lib/hadoop-yarn

 master.initial-num-supervisors: 3

 master.container.size-mb: 128

```

6.运行 `storm-yarn launch <your dir>/storm-0.9.0-wip21/conf/storm.yaml`

提交成功后，会在 yarn 上产生一个 application ，这个就是 storm nimbus 和 ui 的进程

![](/images/storm/20150807/2.png)

获取提交后的属性文件

``` bash
storm-yarn getStormConfig -appId application_1438831879245_0021  -output ~/.storm/storm.yaml
# 通过以下命令得到Nimbus host
cat ~/.storm/storm.yaml | grep nimbus.host
```

然后提交自己的 storm jar 工程

``` bash
storm jar /opt/storm/storm-yarn-master/lib/myStorm-0.0.1-SNAPSHOT-jar-with-dependencies.jar com.myStorm.App WordCountTopology -c nimbus.host=10.200.8.88
```

这时进入 nimbus 的 7070 默认UI端口，就可以看到我们的任务 和 supervisor 进程

![](/images/storm/20150807/3.png)


其他命令
  * 关闭Topology： `storm kill [Topology_name]`
  * 关闭Storm on yarn集群：`storm-yarn shutdown –appId [applicationId]`

---

## 遇到的问题

### <font color="#2798a2" size="3"> storm-yarn launch 失败 </font>

总报错 shell exit code 1 ，yarn 上有 application 的任务

我们可以看 hdfs 上 `/tmp/logs/<user>/logs/<application_id>/<机器_8041>` 里的内容 会看到 thrift 端口默认在 9000 启动，而9000 被占用了，启动不起来，修改`storm-yarn-master/src/main/resource/master_defaults.yaml` 里

``` bash
#master.thrift.port: 9000
master.thrift.port: 9009
```

### <font color="#2798a2" size="3"> supervisor 进程启动失败 </font>

启动后 nimbus 和 ui 进程正常启动，但 storm web UI 里没有 supervisor 进程。网上99% 的图都是没有 supervisor 进程，都是错的。

原因是这样的，由于 storm nimbus 是作为 yarn 的 application 运行的，所以一开始 是独占资源的，会按照 yarn 上 ResourceManager 中配置的最大值来申请

应该在日志里有 <mem:XXX,vCores:YYY> ，最早 yarn 上的配置 RM 内存可最大申请 6G，16vCores，那么 在 hadoop1-6 由于内存偏小，所以被PASS掉了。只有hadoop7-8满足。

但是申请 vCores 时，只有 hadoop3-4 是16核，其他都是8核，按最大申请，又PASS了其他机器，因此 内存 和 CPU 同时满足的机器，为0

修改 yarn 的内存 和 cpu 即可


### <font color="#2798a2" size="3"> 虚拟内存过大 导致 storm on yarn 的 application 被 kill 掉 </font>

运行几个小时后，就挂了。原因不明，现只知道，启动时就会申请 16.8G 的虚拟内存，可以从 top 里看到。原因待查

``` bash
Application application_1438831879245_0001 failed 2 times due to AM Container
for appattempt_1438831879245_0001_000002 exited with exitCode: 143 due to: Container
[pid=44545,containerID=container_1438831879245_0001_02_000001] is running beyond physical memory limits.
Current usage: 1.0 GB of 1 GB physical memory used; 19.8 GB of 2.1 GB virtual memory used. Killing container.
```


---

参考资料：

[http://www.tuicool.com/articles/BFr2Yv](http://www.tuicool.com/articles/BFr2Yv)
[http://blog.csdn.net/jiushuai/article/details/26693311](http://blog.csdn.net/jiushuai/article/details/26693311)
[http://www.cnblogs.com/prisoner/p/4647461.html](http://www.cnblogs.com/prisoner/p/4647461.html)
