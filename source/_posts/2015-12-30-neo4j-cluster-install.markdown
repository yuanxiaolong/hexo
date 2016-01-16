---
layout: post
title: "neo4j cluster install"
date: 2015-12-30 21:26:59 +0800
comments: true
categories: neo4j
tags: neo4j
description: neo4j cluster install
toc: true
---

介绍 neo4j 图数据库的集群部署

<!--more-->

neo4j有企业版和社区版，本文介绍企业版的部署配置。启用HA功能

---

## 环境准备

* centos 3台机器
* neo4j-enterprise-2.3.1-unix.tar.gz [<font color="#517bd2">我的百度云共享</font>](http://pan.baidu.com/s/1c1kz1i4)

---

## 配置

1.将neo4j解压到3台机器，统一目录下，设置 NEO4J_HOME 环境变量（建议解压的linux 用户，就是运行neo4j时的用户）

2.修改配置 $NEO4J_HOME/conf 下 `neo4j.properties`

``` java
remote_shell_enabled=true
remote_shell_host=192.168.7.11
remote_shell_port=1337
online_backup_enabled=true
online_backup_server=192.168.7.11:6362
ha.server_id=1
ha.initial_hosts=192.168.7.11:5001,192.168.7.12:5001,192.168.7.13:5001
ha.cluster_server=192.168.7.11:5001
ha.server=192.168.7.11:6001
```

其中 `ha.server_id` 代表集群中实例号，跟zookeeper类似

3.修改配置 $NEO4J_HOME/conf 下 `neo4j-server.properties` 我由于需要，关闭了权限校验。

``` java
org.neo4j.server.database.mode=HA
org.neo4j.server.webserver.address=192.168.7.11
dbms.security.auth_enabled=false
dbms.browser.remote_content_hostname_whitelist=*
dbms.security.allow_outgoing_browser_connections=true
```

4.打开配置 $NEO4J_HOME/conf 下 `neo4j-wrapper.conf` 里

``` bash
wrapper.java.additional=-Dcom.sun.management.jmxremote.port=3637
wrapper.java.additional=-Dcom.sun.management.jmxremote.password.file=conf/jmx.password
wrapper.java.additional=-Dcom.sun.management.jmxremote.access.file=conf/jmx.access
```

其中需要注意这2个文件的权限

``` bash
-rw------- 1 hadoop hadoop  146 Nov 10 20:15 jmx.access
-rw------- 1 hadoop hadoop   95 Nov 10 20:15 jmx.password
```

5.启动 neo4j ，执行 `$NEO4J_HOME/bin/neo4j start`

访问任意一个节点的 7474端口，进入web UI 查看集群情况

![](/images/neo4j/20151230/1.png)
![](/images/neo4j/20151230/2.png)


可以通过左边的「收藏栏」执行 创建 节点，获取节点等信息。
