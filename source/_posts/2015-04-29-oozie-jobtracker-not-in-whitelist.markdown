---
layout: post
title: "oozie jobtracker not in whitelist"
date: 2015-04-29 11:16:51 +0800
comments: true
categories: ['oozie']
tags: oozie
share: true
description: oozie的白名单问题
toc: true
---

oozie 分配任务到 jobtracker 后，报此jobtracker不在oozie的白名单内

<!--more-->

有2种原因会导致这个报错


---

## 环境介绍

* hadoop 2.3.0-cdh5.0.1
* oozie 4.0.0-cdh5.0.1
* hue 3.5.0

---

## 现象

报错

``` bash
OOZIE error E0900: Jobtracker [datanode2:8032] not allowed, not in Oozies whitelist]
```

分析

查看任务，发现传入的job.properties（我是通过HUE调用oozie的，可能传入的不是文件）里 jobTracker 值确实为 `datanode2:8032`

其中这里的 `datanode2:8032` 是 yarn-HA 中的一台，但目前已经去掉了HA，因此只剩 datanode31:8032 了。为什么会去 datanode2 上的 RM 申请资源，是个问题。


查看了 oozie-site.xml 发现这里确实只有 datanode31

``` xml
<property>
    <name>oozie.service.HadoopAccessorService.jobTracker.whitelist</name>
    <value> datanode31:8032 </value>
    <description>
        Whitelisted job tracker for Oozie service.
    </description>
</property>

```

所以，报白名单错误，有可能是 少配了一个 datanode2:8032 （hadoop集群配置了多个RM），导致 check 不过。

但是，我们集群之前配过 yarn-HA 但是目前已经去掉了 HA，datanode2上确实已经没有RM了，为什么oozie还会让job 去 datanode2上申请资源，明知不可为而为之……

因为 `${jobTracker}` 变量是oozie传的，所以这个变量必然产生于 oozie，会是哪里？ 页面 or 数据库

---

## 实验

我们在HUE的界面上，管理oozie的 Coordinator job 手动添加 指定一个 jobTracker 变量 `datanode999:8032` 。提交、查看、验证，发现最后传入到job时，
jobTracker变成了 `datanode999:8032` ，由于所有任务并未手动指定 `${jobTracker}` 变量，而这个值最后又被指定成了 `datanode2:8032`上

那么，这个值肯定在数据库里。通过 wf_id 类似 <font color="#367eca">0000341-150419032313074-oozie-oozi-W</font> 去 `COORD_JOBS` 表里查记录，
发现`conf` 字段里果然保存了，历史信息。

解决方法：把处于 <font color="#1f8c27"> RUNNING </font>状态的，里 `conf` 字段（BLOB）里脏 JobTracker 信息修改正确（可以利用客户端工具 Sequel Pro）
，下次执行的时候就会用正确的 jobTracker 变量。如果等不到下次调度，则手动修改 `WF_JOBS`表里对应的记录，修改`conf`字段里的`${jobTracker}` 变量


---

## 总结

为什么会在 YARN-HA 去掉后，有的任务仍然去 已经被去掉的 RM 节点上，申请资源呢？

查看了数据库里JOB的创建时间，发现去掉 YARN-HA 后，新生成的任务都没有问题。而在YARN-HA时，部分任务被指向了当时其中一个 RM `datanode2:8032`。被保存到了数据库里，
因为后面从未修改过这些 COORD JOB ，所以这些JOB每次调度，都用的历史信息，即总会报错。

当COORD JOB 修改信息，提交更新后，则不会有问题。

* <b>Tips</b>
  * oozie的COORD JOB 状态处于 <font color="#1f8c27"> RUNNING </font> 状态的，才会每天被调度。
