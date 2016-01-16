---
layout: post
title: "introduce hadoop"
date: 2014-07-12 20:44:07 +0800
comments: true
category: hadoop
tags: hadoop
share: true
description: 简介hadoop
toc: true
---

用通俗易懂的方式来介绍hadoop

<!--more-->


## hadoop 是什么？

宏观上的hadoop是一组数据应用的统称，有些类似j2ee。因此有人把hadoop比作一个“生态圈”。如果说j2ee是面向web应用开发的一套技术的总称，那么hadoop就是面向数据开发的另一套技术的总称了。


那么微观上的hadoop是什么呢？我们来一起看一下github上的hadoop （注：这是此时最新的trunk，3.0.0-SNAPSHOT）
![](/images/hadoop/hadoop-project-1.png)
![](/images/hadoop/hadoop-project-2.png)

由此可见，它是一个多工程的依赖的maven工程。（从hadoop源码的进化过程来看，可以看到最初是ant，后面才转为maven）

## hadoop 是如何运作的？

hadoop是用java语言开发的，因此是基于jvm的一个工程，因此要运行hadoop，首先要先安装java。

hadoop一般是运行在linux机器上，因此需要在linux机上安装java及hadoop。

下一篇： [install-hadoop-on-local](http://blog.yuanxiaolong.cn/blog/2014/07/12/install-hadoop-on-local/) 将会简单安装一个单机伪分布式的hadoop，用于学习之用

## hadoop 解决了什么问题？

hadoop几乎是大数据的代名词了，那么我们需要先了解什么是大数据？

*<font color="green">大数据</font>* 其实是指的，当数据处理超过了机器所能承受的极限时，这个数据对你来说，就可以称之为*<font color="green">大数据</font>* 。
因为数据大，例如几十PB的数据，放到什么机器上都搞不定，因此符合大数据的范畴，也就给我们造成了*<font color="green">大数据</font>*==*<font color="green">数据大</font>* 的感觉。

因此 **<font color="green">横向扩展</font>** 的设计代替了 **<font color="green">纵向扩展</font>**。

传统的纵向，指的是当业务发展后，现有的机器满足不了现有的业务时，人们选择买更好的CPU，更多的硬盘、内存等，添加在这台机器上，或者替换成更好的小型机。因此当业务再次发展时，任何机器都不能承受，才产生了横向的设计。

人们更愿意用10台较差的机器代替1台较好的机器，而且更节约资金。

因此分治的思想，在hadoop上体现了。并行计算在多台机器上，来处理大数据。
那么回到问题,这样的hadoop部署方案，适合于什么场景？

<font color="#851666">高吞吐、高延迟、批处理</font>
