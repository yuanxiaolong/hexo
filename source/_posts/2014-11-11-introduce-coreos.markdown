---
layout: post
title: "introduce CoreOS"
date: 2014-11-11 22:38:59 +0800
comments: true
categories: coreos
tags: CoreOS
share: true
description: 介绍CoreOS
toc: true
---

介绍一下新一代操作系统CoreOS

<!--more-->

之前用的是CentOS6，发现CentOS如果要进行，同一局域网多物理主机连通的话，要自定义网桥。如果是不同网段，则还需要用交换机打通不同网段，

这就需要很大的运维成本，和网络管理经验。如何降低这一成本，发现了CoreOS，无论是否可行，先试验一下吧。

CoreOS是一个基于Linux 内核的轻量级操作系统，除了精简，而且拥有天生面向分布式的优点。
[<font color="#4b80fe"> 官网 </font>](https://coreos.com/)

---

## 精简

CoreOS 比其它linux系统精简 40%的内存占用
![](/images/coreos/20141110/1.png)

---

## 无缝升级

CoreOS 有2个分区 ，可以把它们称为 ROOT-A和ROOT-B，用户工作时用ROOT-A，升级CoreOS时，在ROOT-B中进行。
且ROOT是只读的，因此当升级完成时，只需要将 A、B角色切换即可，无需停机升级。
![](/images/coreos/20141110/2.png)

---

## 以Docker Container为单位管理

CoreOS 天生支持docker，并将Docker作为一个服务，进行启动、运行。由于ROOT是只读的，因此不能用yum、apt-get这样的命令去安装软件，而需要把你所需要的封装到 自己的 Docker容器里。
![](/images/coreos/20141110/3.png)

---

## 面向分布式的OS

CoreOS，把所有docker container的信息放到 etcd组件里，无论在哪个机器，所拿到的信息都是一致的。
![](/images/coreos/20141110/4.png)

---

## 服务检测及切换

在CoreOS里，虽然天生支持docker，但一般会把docker container封装成 linux管理工具 systemd 的service，来运行。
一般来说，每一个service，需要写一个检测它运行正常的另一个service，有点“守护进程”的意思。一旦发现不可用的情况，
则按照“守护进程”的service策略，自动选择机器进行，fail-over

---
