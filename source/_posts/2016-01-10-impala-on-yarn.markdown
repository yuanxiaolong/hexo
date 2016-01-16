---
layout: post
title: "impala on yarn"
date: 2016-01-10 18:51:21 +0800
comments: true
categories:  ['impala']
tags: impala
description: 利用llama进行 impala on yarn
toc: true
---

利用 llama 这个中间件 进行 impala on yarn

<!--more-->

Llama 用于 Impala on yarn 作为一个代理，向 yarn 申请资源提交任务 [下载地址](http://archive-primary.cloudera.com/cdh5/cdh/5/)


---

## Llama 安装启动

解压后在 `$LLAMA_HOME/llama-dist/target/llama-1.0.0-cdh5.4.0.tar.gz`

修改 `yarn-site.xml`

``` xml
<property>
  <name>yarn.scheduler.minimum-allocation-mb</name>
  <value>0</value>
</property>
<property>
  <name>yarn.scheduler.minimum-allocation-vcores</name>
  <value>0</value>
</property>
<property>
  <name>yarn.nodemanager.aux-services</name>
  <value>mapreduce_shuffle,llama_nm_plugin</value>
</property>
<property>
  <name>yarn.nodemanager.aux-services.llama_nm_plugin.class</name>
  <value>com.cloudera.llama.nm.LlamaNMAuxiliaryService</value>
</property>

```

修改 `core-site.xml`

``` xml
<property>
    <name>hadoop.proxyuser.llama.hosts</name>
    <value>*</value>
  </property>
  <property>
    <name>hadoop.proxyuser.llama.groups</name>
    <value>*</value>
  </property>
```

修改 `conf/llama-site.xml`

``` xml
 <property>
    <name>llama.am.server.thrift.address</name>
    <value>192.168.7.11:15000</value>
 </property>
 <property>
    <name>llama.nm.server.thrift.address</name>
    <value>192.168.7.11:15100</value>
 </property>
```

配置 HA

``` xml
 <property>
    <name>llama.am.cluster.id</name>
    <value>llama</value>
 </property>
 <property>
    <name>llama.am.ha.enabled</name>
    <value>true</value>
 </property>
<property>
    <name>llama.am.ha.zk-base</name>
    <value>/llama</value>
 </property>
 <property>
    <name>llama.am.ha.zk-quorum</name>
     <value>192.168.7.13:2181,192.168.7.15:2181,192.168.7.16:2181</value>
</property>

```

修改 `llamaadmin.xml`

``` xml
<property>
    <name>llamaadmin.server.thrift.address</name>
    <value>192.168.7.11:15002</value>
</property>
```

修改 `libexec/llama-env.sh`  ，建好 log文件夹，修改777权限

``` bash
export LLAMA_AM_SERVER_CONF=/usr/local/datacenter/llama/conf
export LLAMA_AM_SERVER_LOG=/usr/local/datacenter/llama/log
```

把 $HADOOP_HOME/share/hadoop 下  hdfs 、common、 yarn 的jar ，软连接到 $LLAMA_HOME/lib 下，后同样，需要把 llama 的 jar 加入到 hadoop classpath 里，我用软连接的方式。

``` bash
ln -s /usr/local/datacenter/llama/lib/llama-1.0.0-cdh5.4.0.jar /usr/local/datacenter/hadoop/lib/
ln -s /usr/local/datacenter/llama/lib/metrics-core-3.0.1.jar /usr/local/datacenter/hadoop/lib/
ln -s /usr/local/datacenter/llama/lib/libthrift-0.9.0.jar /usr/local/datacenter/hadoop/lib/
```

环境变量设置 `HADOOP_HOME` 在 2 台机器分别启动

``` bash
nohup ./llama &
```

访问 http://192.168.7.11:15001 和 http://192.168.7.12:15001 （这里的 queueCrossref 和 nodesCrossref 为空是正常的，当有任务提交到 yarn 上时，资源分配后，这里才会显示很多其他信息）

![](/images/impala/20160110/1.png)

![](/images/impala/20160110/2.png)


---

## 修改 Impala 让其感知 Llama

1.准备工作已经完毕，下面设置 impala 的文件 `/etc/default/impala` 里添加 `enable_rm` `llama_addresses` `fair_scheduler_allocation_path` `cgroup_hierarchy_path`

2.首先启动 cgroup 服务，如果没有安装先通过yum安装

```
service cgconfig start
```

注意：设置`cgroup_hierarchy_path` 为 `/cgroup/cpu`  注意需要修改子目录权限，让其可写

如果是CentOS6（centos5 不支持cgroup好像，所以不能用 impala on yarn）

```
IMPALA_SERVER_ARGS=" \
    -log_dir=${IMPALA_LOG_DIR} \
    -catalog_service_host=${IMPALA_CATALOG_SERVICE_HOST} \
    -state_store_port=${IMPALA_STATE_STORE_PORT} \
    -use_statestore \
    -state_store_host=${IMPALA_STATE_STORE_HOST} \
    -enable_rm=true \
    -rm_always_use_defaults=true \
    -llama_addresses=192.168.7.11:15000,192.168.7.12:15000 \
    -fair_scheduler_allocation_path='/usr/local/datacenter/hadoop/etc/hadoop/fair-scheduler.xml' \
    -cgroup_hierarchy_path='`/cgroup/cpu' \
    -be_port=${IMPALA_BACKEND_PORT}"
```

3.然后，重启yarn 重启 impala-server，发现有4个客户端连接到 llama 上

![](/images/impala/20160110/3.png)

4.然后提交一个任务，可以发现一个常驻的 application 在 yarn 上，无论开多少个 impala-shell 均只有一个。10分钟后此 application 状态变为 fail 且自动关闭

![](/images/impala/20160110/4.png)

---

## 碰见的问题

### <font color="#c89226" >UNKNOW HOST</font>

```
com.cloudera.llama.util.LlamaException: RESERVATION_ASKING_UNKNOWN_NODE
- Reservation '9348b4b355578e9f:16ce0ead700fff95',
  expansion 'null' asking for a resource on node 'node007013' that does not exist.
```

是因为 我的 yarn 虽然 ResourceManager 启动起来了，yarn web ui 正常，但各个节点 NodeManager 启动失败了，逐一启动，排除自身问题，直到正常为止  `yarn-deamon.sh start nodemanager`

### <font color="#c89226" >CGroup</font>

```
ERROR: CGroup xxxxx does not have a /tasks file
```

这个原因可能是   没开启CGroup服务 且 没有把 cgroup 路径设置 为  `/cgroup/cpu` 下

### <font color="#c89226" >llama HA</font>

llama HA 本身是没有问题的，验证方式可以 kill 掉 active llama pid ，然后看 standby 的是否会转变成 active，
然后此时 通过 web ui 是看不到有客户端 impala-server 连接的，可以启动一个 impala-shell ，就能看到此机器连接上了 llama active了



资料

- http://cloudera.github.io/llama/RunningLlama.html
- http://www.cloudera.com/content/www/en-us/documentation/archive/impala/2-x/2-1-x/topics/impala_resource_management.html
- http://www.cloudera.com/content/www/en-us/documentation/enterprise/latest/topics/cdh_hag_llama_ha.html
- http://ae.yyuap.com/pages/viewpage.action?pageId=918464
