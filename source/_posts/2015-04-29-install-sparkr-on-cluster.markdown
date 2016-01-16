---
layout: post
title: "install sparkR on cluster"
date: 2015-04-29 17:13:42 +0800
comments: true
categories: ['spark']
tags: spark
share: true
description: sparkR集群安装实践
toc: true
---

sparkR是一个以R语言包装的 spark 客户端

<!--more-->

安装sparkR ，运行在 yarn 的集群上

---

## 环境准备

分析人员熟悉R，但不熟悉scala，R只能单机，遇到大文件，则无法分布式分析。spark是高效的计算框架，资源可以委托yarn管理，sparkR就是各取所长的客户端。

---

### <font color="#dc721c">编译安装R，在所有hadoop节点</font>


1.进入 [<font color="#465999"> R官网 </font>](http://www.r-project.org/)，点击 ` download R ` 进入镜像站点，点击第一个 `http://cran.rstudio.com/` （不要进入China站点，因为里面没有src源码）

2.点击左侧导航 ` R Sources ` 进入下载页面，复制 [<font color="#465999"> http://cran.rstudio.com/src/base/R-3/R-3.1.2.tar.gz </font>](http://cran.rstudio.com/src/base/R-3/R-3.1.2.tar.gz),可以通过wget下载到服务器上 ,我的云盘共享[<font color="#465999"> 点击这里 </font>](http://pan.baidu.com/s/1qW7e6Zy)

3.在编译R之前，需要通过yum安装以下几个程序

* ` yum install gcc-gfortran `
否则报”configure: error: No F77 compiler found”错误
* ` yum install gcc gcc-c++ `
否则报”configure: error: C++ preprocessor “/lib/cpp” fails sanity check”错误
* ` yum install readline-devel `
否则报”–with-readline=yes (default) and headers/libs are not available”错误
* ` yum install libXt-devel `
否则报”configure: error: –with-x=yes (default) and X11 headers/libs are not available”错误
* `yum install  curl curl-devel `
 是为了保证 R 的 devtool 包能加载成功，如果使用到这个包的话

4.` tar zxvf R-3.1.2.tar.gz && cd R-3.1.2 `

5.` ./configure`

6.` make && make install `

<font color="#9a9f95">PS:如果需要预先实验编译步骤是否正确，可以在一台机器上手动编译，再测试一下R </font>

``` bash
Type 'demo()' for some demos, 'help()' for on-line help, or
'help.start()' for an HTML browser interface to help.
Type 'q()' to quit R.

> x<-c(1,2,3)
> y<-c(102,299,301)
> model<-lm(y~x)
> summary(model)

Call:
lm(formula = y ~ x)

Residuals:
2 3
-32.5 65.0 -32.5

Coefficients:
Estimate Std. Error t value Pr(>|t|)
(Intercept) 35.00 121.60 0.288 0.822
x 99.50 56.29 1.768 0.328

Residual standard error: 79.61 on 1 degrees of freedom
Multiple R-squared: 0.7575, Adjusted R-squared: 0.5151
F-statistic: 3.124 on 1 and 1 DF, p-value: 0.3278

> proc.time()
user system elapsed
0.300 0.024 97.456
```

---

### <font color="#dc721c">编译sparkR</font>

可以通过git clone 代码到机器上编译好后，把编译好的fat jar 放到集群上，不需要在集群上每个机器编译sparkR

``` bash
git clone https://github.com/amplab-extras/SparkR-pkg.git
cd SparkR-pkg/
SPARK_HADOOP_VERSION=2.3.0-cdh5.0.1 ./install-dev.sh
```

``` bash
[root@com SparkR-pkg]# SPARK_HADOOP_VERSION=2.3.0-cdh5.0.1 ./install-dev.sh
* installing *source* package ‘SparkR’ ...
** libs
** arch -
./sbt/sbt assembly
Attempting to fetch sbt
######################################################################## 100.0%
Launching sbt from sbt/sbt-launch-0.13.6.jar
##########中间很多输出
[success] Total time: 103 s, completed Apr 29, 2015 5:41:21 PM
cp -f target/scala-2.10/sparkr-assembly-0.1.jar ../inst/
R CMD SHLIB -o SparkR.so string_hash_code.c
make[1]: Entering directory `/data/xiaolong.yuanxl/SparkR-pkg/pkg/src'
gcc -std=gnu99 -I/usr/local/lib64/R/include -DNDEBUG  -I/usr/local/include    -fpic  -g -O2  -c string_hash_code.c -o string_hash_code.o
gcc -std=gnu99 -shared -L/usr/local/lib64 -o SparkR.so string_hash_code.o
make[1]: Leaving directory `/data/xiaolong.yuanxl/SparkR-pkg/pkg/src'
installing to /data/xiaolong.yuanxl/SparkR-pkg/lib/SparkR/libs
** R
** inst
** preparing package for lazy loading
Creating a generic function for ‘lapply’ from package ‘base’ in package ‘SparkR’
Creating a generic function for ‘Filter’ from package ‘base’ in package ‘SparkR’
** help
*** installing help indices
** building package indices
** testing if installed package can be loaded
* DONE (SparkR)
```

---

### <font color="#dc721c">[本地] 测试sparkR</font>

提交方式有2种，一种交互命令行方式，另一种通过脚本提交（类似hive 和 hive -e）。

1.命令行交互模式

``` bash
[root@com SparkR-pkg]# ./sparkR

R version 3.1.2 (2014-10-31) -- "Pumpkin Helmet"
Copyright (C) 2014 The R Foundation for Statistical Computing
Platform: x86_64-unknown-linux-gnu (64-bit)

R is free software and comes with ABSOLUTELY NO WARRANTY.
You are welcome to redistribute it under certain conditions.
Type 'license()' or 'licence()' for distribution details.

  Natural language support but running in an English locale

R is a collaborative project with many contributors.
Type 'contributors()' for more information and
'citation()' on how to cite R or R packages in publications.

Type 'demo()' for some demos, 'help()' for on-line help, or
'help.start()' for an HTML browser interface to help.
Type 'q()' to quit R.

[SparkR] Initializing with classpath /data/xiaolong.yuanxl/SparkR-pkg/lib/SparkR/sparkr-assembly-0.1.jar
###中间省略
Launching java with command  java   -Xmx512m -cp '/data/xiaolong.yuanxl/SparkR-pkg/lib/SparkR/sparkr-assembly-0.1.jar:' edu.berkeley.cs.amplab.sparkr.SparkRBackend /tmp/RtmpFB317a/backend_port455e439257bf
15/04/29 17:42:59 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable

 Welcome to SparkR!
 Spark context is available as sc
>
```

2.脚本提交

``` bash
[root@com SparkR-pkg]# ./lib/SparkR/sparkR-submit  ./examples/pi.R local[2]
Running /root/spark-1.1.0//bin/spark-submit --class edu.berkeley.cs.amplab.sparkr.SparkRRunner --files ./examples/pi.R  /root/xiaolong.yuanxl/SparkR-pkg/lib/SparkR/sparkr-assembly-0.1.jar ./examples/pi.R local[2]
------------------------------------------------->
old java home=> /usr/java/jdk1.7.0_55-cloudera/
/etc/hadoop/conf
new java home /usr/java/jdk1.6.0_31/

Spark assembly has been built with Hive, including Datanucleus jars on classpath
WARNING: ignoring environment value of R_HOME
Loading required package: methods
[SparkR] Initializing with classpath /root/xiaolong.yuanxl/SparkR-pkg/lib/SparkR/sparkr-assembly-0.1.jar

15/04/23 17:51:52 INFO spark.SecurityManager: Changing view acls to: root,
15/04/23 17:51:52 INFO spark.SecurityManager: Changing modify acls to: root,
15/04/23 17:51:52 INFO spark.SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(root, ); users with modify permissions: Set(root, )
15/04/23 17:51:53 INFO slf4j.Slf4jLogger: Slf4jLogger started
15/04/23 17:51:53 INFO Remoting: Starting remoting
15/04/23 17:51:53 INFO Remoting: Remoting started; listening on addresses :[akka.tcp://sparkDriver@com.hunantv.logservernode2:47089]
15/04/23 17:51:53 INFO Remoting: Remoting now listens on addresses: [akka.tcp://sparkDriver@com.hunantv.logservernode2:47089]
15/04/23 17:51:53 INFO util.Utils: Successfully started service 'sparkDriver' on port 47089.
15/04/23 17:51:53 INFO spark.SparkEnv: Registering MapOutputTracker
15/04/23 17:51:53 INFO spark.SparkEnv: Registering BlockManagerMaster
15/04/23 17:51:53 INFO storage.DiskBlockManager: Created local directory at /tmp/spark-local-20150423175153-0caa
15/04/23 17:51:54 INFO util.Utils: Successfully started service 'Connection manager for block manager' on port 60352.
15/04/23 17:51:54 INFO network.ConnectionManager: Bound socket to port 60352 with id = ConnectionManagerId(com.hunantv.logservernode2,60352)
15/04/23 17:51:54 INFO storage.MemoryStore: MemoryStore started with capacity 265.0 MB
15/04/23 17:51:54 INFO storage.BlockManagerMaster: Trying to register BlockManager
15/04/23 17:51:54 INFO storage.BlockManagerMasterActor: Registering block manager com.hunantv.logservernode2:60352 with 265.0 MB RAM
15/04/23 17:51:54 INFO storage.BlockManagerMaster: Registered BlockManager
15/04/23 17:51:54 INFO spark.HttpFileServer: HTTP File server directory is /tmp/spark-92c60115-974e-4178-86c1-d5ce689383f4
15/04/23 17:51:54 INFO spark.HttpServer: Starting HTTP Server
15/04/23 17:51:54 INFO server.Server: jetty-8.y.z-SNAPSHOT
15/04/23 17:51:54 INFO server.AbstractConnector: Started SocketConnector@0.0.0.0:33381
15/04/23 17:51:54 INFO util.Utils: Successfully started service 'HTTP file server' on port 33381.
15/04/23 17:51:54 INFO server.Server: jetty-8.y.z-SNAPSHOT
15/04/23 17:51:54 INFO server.AbstractConnector: Started SelectChannelConnector@0.0.0.0:4040
15/04/23 17:51:54 INFO util.Utils: Successfully started service 'SparkUI' on port 4040.
15/04/23 17:51:54 INFO ui.SparkUI: Started SparkUI at http://com.hunantv.logservernode2:4040
15/04/23 17:51:54 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
15/04/23 17:51:55 INFO spark.SparkContext: Added JAR file:///root/xiaolong.yuanxl/SparkR-pkg/lib/SparkR/sparkr-assembly-0.1.jar at http://com.hunantv.logservernode2:33381/jars/sparkr-assembly-0.1.jar with timestamp 1429782715609
15/04/23 17:51:55 INFO util.Utils: Copying /root/xiaolong.yuanxl/SparkR-pkg/./examples/pi.R to /tmp/spark-64f7170f-21ed-4551-8c1d-1843895a8e47/pi.R
15/04/23 17:51:55 INFO spark.SparkContext: Added file file:/root/xiaolong.yuanxl/SparkR-pkg/./examples/pi.R at http://com.hunantv.logservernode2:33381/files/pi.R with timestamp 1429782715611
15/04/23 17:51:55 INFO util.AkkaUtils: Connecting to HeartbeatReceiver: akka.tcp://sparkDriver@com.hunantv.logservernode2:47089/user/HeartbeatReceiver
15/04/23 17:51:56 INFO spark.SparkContext: Starting job: collect at NativeMethodAccessorImpl.java:-2
15/04/23 17:51:56 INFO scheduler.DAGScheduler: Got job 0 (collect at NativeMethodAccessorImpl.java:-2) with 2 output partitions (allowLocal=false)
15/04/23 17:51:56 INFO scheduler.DAGScheduler: Final stage: Stage 0(collect at NativeMethodAccessorImpl.java:-2)
15/04/23 17:51:56 INFO scheduler.DAGScheduler: Parents of final stage: List()
15/04/23 17:51:56 INFO scheduler.DAGScheduler: Missing parents: List()
15/04/23 17:51:56 INFO scheduler.DAGScheduler: Submitting Stage 0 (RRDD[1] at RDD at RRDD.scala:19), which has no missing parents
15/04/23 17:51:56 INFO storage.MemoryStore: ensureFreeSpace(73784) called with curMem=0, maxMem=277842493
15/04/23 17:51:56 INFO storage.MemoryStore: Block broadcast_0 stored as values in memory (estimated size 72.1 KB, free 264.9 MB)
15/04/23 17:51:56 INFO scheduler.DAGScheduler: Submitting 2 missing tasks from Stage 0 (RRDD[1] at RDD at RRDD.scala:19)
15/04/23 17:51:56 INFO scheduler.TaskSchedulerImpl: Adding task set 0.0 with 2 tasks
15/04/23 17:51:56 WARN scheduler.TaskSetManager: Stage 0 contains a task of very large size (391 KB). The maximum recommended task size is 100 KB.
15/04/23 17:51:56 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 0.0 (TID 0, localhost, PROCESS_LOCAL, 401308 bytes)
15/04/23 17:51:56 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 0.0 (TID 1, localhost, PROCESS_LOCAL, 401308 bytes)
15/04/23 17:51:56 INFO executor.Executor: Running task 0.0 in stage 0.0 (TID 0)
15/04/23 17:51:56 INFO executor.Executor: Running task 1.0 in stage 0.0 (TID 1)
15/04/23 17:51:56 INFO executor.Executor: Fetching http://com.hunantv.logservernode2:33381/files/pi.R with timestamp 1429782715611
15/04/23 17:51:56 INFO util.Utils: Fetching http://com.hunantv.logservernode2:33381/files/pi.R to /tmp/fetchFileTemp5460076502660272005.tmp
15/04/23 17:51:56 INFO executor.Executor: Fetching http://com.hunantv.logservernode2:33381/jars/sparkr-assembly-0.1.jar with timestamp 1429782715609
15/04/23 17:51:56 INFO util.Utils: Fetching http://com.hunantv.logservernode2:33381/jars/sparkr-assembly-0.1.jar to /tmp/fetchFileTemp4739048988763487952.tmp
15/04/23 17:51:57 INFO executor.Executor: Adding file:/tmp/spark-64f7170f-21ed-4551-8c1d-1843895a8e47/sparkr-assembly-0.1.jar to class loader
WARNING: ignoring environment value of R_HOME
100000
100000
15/04/23 17:51:57 INFO sparkr.RRDD: Times: boot = 0.401 s, init = 0.010 s, broadcast = 0.000 s, read-input = 0.004 s, compute = 0.255 s, write-output = 0.001 s, total = 0.671 s
15/04/23 17:51:57 INFO executor.Executor: Finished task 1.0 in stage 0.0 (TID 1). 622 bytes result sent to driver
15/04/23 17:51:57 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 0.0 (TID 1) in 1324 ms on localhost (1/2)
15/04/23 17:51:57 INFO sparkr.RRDD: Times: boot = 0.396 s, init = 0.006 s, broadcast = 0.003 s, read-input = 0.005 s, compute = 0.302 s, write-output = 0.001 s, total = 0.713 s
15/04/23 17:51:57 INFO executor.Executor: Finished task 0.0 in stage 0.0 (TID 0). 622 bytes result sent to driver
15/04/23 17:51:57 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 0.0 (TID 0) in 1365 ms on localhost (2/2)
15/04/23 17:51:57 INFO scheduler.DAGScheduler: Stage 0 (collect at NativeMethodAccessorImpl.java:-2) finished in 1.386 s
15/04/23 17:51:57 INFO scheduler.TaskSchedulerImpl: Removed TaskSet 0.0, whose tasks have all completed, from pool
15/04/23 17:51:57 INFO spark.SparkContext: Job finished: collect at NativeMethodAccessorImpl.java:-2, took 1.663123356 s
Pi is roughly 3.14104
15/04/23 17:51:57 INFO spark.SparkContext: Starting job: collect at NativeMethodAccessorImpl.java:-2
15/04/23 17:51:57 INFO scheduler.DAGScheduler: Got job 1 (collect at NativeMethodAccessorImpl.java:-2) with 2 output partitions (allowLocal=false)
15/04/23 17:51:57 INFO scheduler.DAGScheduler: Final stage: Stage 1(collect at NativeMethodAccessorImpl.java:-2)
15/04/23 17:51:57 INFO scheduler.DAGScheduler: Parents of final stage: List()
15/04/23 17:51:57 INFO scheduler.DAGScheduler: Missing parents: List()
15/04/23 17:51:57 INFO scheduler.DAGScheduler: Submitting Stage 1 (RRDD[2] at RDD at RRDD.scala:19), which has no missing parents
15/04/23 17:51:57 INFO storage.MemoryStore: ensureFreeSpace(8000) called with curMem=73784, maxMem=277842493
15/04/23 17:51:57 INFO storage.MemoryStore: Block broadcast_1 stored as values in memory (estimated size 7.8 KB, free 264.9 MB)
15/04/23 17:51:57 INFO scheduler.DAGScheduler: Submitting 2 missing tasks from Stage 1 (RRDD[2] at RDD at RRDD.scala:19)
15/04/23 17:51:57 INFO scheduler.TaskSchedulerImpl: Adding task set 1.0 with 2 tasks
15/04/23 17:51:57 WARN scheduler.TaskSetManager: Stage 1 contains a task of very large size (391 KB). The maximum recommended task size is 100 KB.
15/04/23 17:51:57 INFO scheduler.TaskSetManager: Starting task 0.0 in stage 1.0 (TID 2, localhost, PROCESS_LOCAL, 401308 bytes)
15/04/23 17:51:57 INFO scheduler.TaskSetManager: Starting task 1.0 in stage 1.0 (TID 3, localhost, PROCESS_LOCAL, 401308 bytes)
15/04/23 17:51:57 INFO executor.Executor: Running task 1.0 in stage 1.0 (TID 3)
15/04/23 17:51:57 INFO executor.Executor: Running task 0.0 in stage 1.0 (TID 2)
15/04/23 17:51:58 INFO sparkr.RRDD: Times: boot = 0.009 s, init = 0.007 s, broadcast = 0.000 s, read-input = 0.005 s, compute = 0.000 s, write-output = 0.001 s, total = 0.022 s
15/04/23 17:51:58 INFO executor.Executor: Finished task 0.0 in stage 1.0 (TID 2). 618 bytes result sent to driver
15/04/23 17:51:58 INFO scheduler.TaskSetManager: Finished task 0.0 in stage 1.0 (TID 2) in 42 ms on localhost (1/2)
15/04/23 17:51:58 INFO sparkr.RRDD: Times: boot = 0.016 s, init = 0.007 s, broadcast = 0.001 s, read-input = 0.004 s, compute = 0.000 s, write-output = 0.001 s, total = 0.029 s
15/04/23 17:51:58 INFO executor.Executor: Finished task 1.0 in stage 1.0 (TID 3). 618 bytes result sent to driver
15/04/23 17:51:58 INFO scheduler.TaskSetManager: Finished task 1.0 in stage 1.0 (TID 3) in 48 ms on localhost (2/2)
15/04/23 17:51:58 INFO scheduler.TaskSchedulerImpl: Removed TaskSet 1.0, whose tasks have all completed, from pool
15/04/23 17:51:58 INFO scheduler.DAGScheduler: Stage 1 (collect at NativeMethodAccessorImpl.java:-2) finished in 0.054 s
15/04/23 17:51:58 INFO spark.SparkContext: Job finished: collect at NativeMethodAccessorImpl.java:-2, took 0.066888288 s
Num elements in RDD  200000
```

---

### <font color="#dc721c">[集群] 测试sparkR</font>

集群测试需要修改 sparkR的部分脚本 （sparkR 和 sparkR-submit）

1. 添加 yarn 依赖包 <font color="#209a1c">（/root/spark-1.1.0/assembly/target/scala-2.10/spark-assembly-1.1.0-hadoop2.3.0-cdh5.0.1.jar）</font>
2. 添加配置环境变量 <font color="#209a1c">（MASTER、SPARK_HOME、YARN_CONF_DIR、JAVA_HOME、R_PROFILE_USER）</font>
3. 指定 master 为 yarn <font color="#209a1c">（MASTER="yarn-client"）</font>
4. 设置 spark 的 driver 的内存大小<font color="#209a1c">（SPARK_MEM）</font>
5. 设置 spark driver 的个数 <font color="#209a1c"> (没设置，脚本里有变量) </font>


* <font color="#aa1613"> sparkR 脚本 </font>

``` bash
#!/bin/bash

FWDIR="$(cd `dirname $0`; pwd)"
export MASTER="yarn-client"
export PROJECT_HOME="$FWDIR"
export SPARK_HOME=/root/spark-1.1.0/
export YARN_CONF_DIR=/etc/hadoop/conf
export JAVA_HOME=/usr/java/jdk1.7.0_55-cloudera/
export SPARK_MEM=1g
MASTER=yarn-client
echo $CLASSPATH
export CLASSPATH=$CLASSPATH:/root/spark-1.1.0/assembly/target/scala-2.10/spark-assembly-1.1.0-hadoop2.3.0-cdh5.0.1.jar

unset YARN_CONF_DIR
unset JAVA_HOME
export YARN_CONF_DIR=/etc/hadoop/conf:/root/spark-1.1.0/assembly/target/scala-2.10/spark-assembly-1.1.0-hadoop2.3.0-cdh5.0.1.jar

export R_PROFILE_USER="/tmp/sparkR.profile"

if [ $# -gt 0 ]; then
  # If we are running an R program, only set libPaths and use Rscript
cat > /tmp/sparkR.profile << EOF
.First <- function() {
  projecHome <- Sys.getenv("PROJECT_HOME")
  .libPaths(c(paste(projecHome,"/lib", sep=""), .libPaths()))
  Sys.setenv(NOAWT=1)
}
EOF

  Rscript "$@"
else

  # If we don't have an R file to run, initialize context and run R
cat > /tmp/sparkR.profile << EOF
.First <- function() {
  projecHome <- Sys.getenv("PROJECT_HOME")
  Sys.setenv(NOAWT=1)
  .libPaths(c(paste(projecHome,"/lib", sep=""), .libPaths()))
  library(SparkR)
  sc <- sparkR.init(Sys.getenv("MASTER", unset = ""))
  assign("sc", sc, envir=.GlobalEnv)
  cat("\n Welcome to SparkR!")
  cat("\n Spark context is available as sc\n")
}
EOF

  R
fi
```

* <font color="#aa1613"> sparkR-submit 脚本 </font>

``` bash
export MASTER="yarn-client"
export PROJECT_HOME="$FWDIR"
export SPARK_HOME=/root/spark-1.1.0/
export YARN_CONF_DIR=/etc/hadoop/conf
export JAVA_HOME=/usr/java/jdk1.7.0_55-cloudera/
export SPARK_MEM=1g
MASTER=yarn-client
echo $CLASSPATH
export CLASSPATH=$CLASSPATH:/data/xiaolong.yuanxl/SparkR-pkg/lib/SparkR/spark-assembly-1.1.0-hadoop2.3.0-cdh5.0.1.jar
export R_LIBS=/data/xiaolong.yuanxl/SparkR-pkg/lib

unset YARN_CONF_DIR
unset JAVA_HOME
export YARN_CONF_DIR=/etc/hadoop/conf:/data/xiaolong.yuanxl/SparkR-pkg/lib/SparkR/spark-assembly-1.1.0-hadoop2.3.0-cdh5.0.1.jar

export R_PROFILE_USER="/tmp/sparkR.profile"
```

---

### <font color="#9fbf1e">测试命令</font>

``` bash
./sparkR-submit --master yarn-client ../../examples/pi.R yarn-client 4
```

---

## 备注
用户权限问题可以通过组解决，后面2个是启动sparkR和install.packages时没有权限的错误

``` bash
usermod -a -G groupA user
chmod 777 /tmp/sparkR.profile
chmod 777 /data/xiaolong.yuanxl/SparkR-pkg/lib
```

---

## 多用户环境

```bash
ln -s /data/xiaolong.yuanxl/SparkR-pkg/sparkR  /usr/bin/sparkR
ln -s /data/xiaolong.yuanxl/SparkR-pkg/lib/SparkR/sparkR-submit  /usr/bin/sparkR-submit
```
