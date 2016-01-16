---
layout: post
title: "jvm MaxHeapSize limit"
date: 2014-10-18 21:50:55 +0800
comments: true
categories: jvm
tags: jvm
share: true
description: 设置jvm的最大分配内存
toc: true
---
调整jvm最大堆内存参数，限制操作系统分配太多的内存给jvm

<!--more-->

最近在做简单的压力测试时，发现jvm进程会吃掉1G+的内存，并持续压测后，稳定在400M。

但是，由于进程是跑在别人的数据库服务器上，当然越精简越好。eclipse、tomcat之前也设置过-xMx，但印象不深，这次深了。


---

## 环境

测试机win7,16G内存。没错，是windows机器。

<font color="#7c837f">(如果是linux下只用加 java -xMx1024m 这样就可以了，文章主要目的是对比设置参数前后的情况)</font>

![](/images/jvm/20141016/1.png)

---

## 过程

先看一下最开始的jvm情况，用jconsole连上进程，可以看到最开始吃掉了1G内存！后来下滑

![](/images/jvm/20141016/2.png)

![](/images/jvm/20141016/3.png)

后来设置了 -xMx128m ，新启动另一个进程，起作用了.

![](/images/jvm/20141016/4.png)


来一张任务管理器的图，方便对比
![](/images/jvm/20141016/5.png)


用 <font color="#976d24">jinfo</font>命令看一下参数，确实对的上

```bash
C:\Program Files\Java\jdk1.6.0_39\bin>jinfo -flag MaxHeapSize 10400
-XX:MaxHeapSize=134217728

C:\Program Files\Java\jdk1.6.0_39\bin>jinfo -flag MaxHeapSize 5728
-XX:MaxHeapSize=4292870144
```

---

## 资料

JVM初始分配的内存由-Xms指定，默认是物理内存的1/64；JVM最大分配的内存由-Xmx指定，默认是物理内存的1/4。

默认空余堆内存小于 40%时，JVM就会增大堆直到-Xmx的最大限制；空余堆内存大于70%时，JVM会减少堆直到-Xms的最小限制。

---

## exe4j

附上一张exe4j改vm参数的地方，是用exe4j将java程序做成了exe，在win下方便运行
![](/images/jvm/20141016/6.png)
