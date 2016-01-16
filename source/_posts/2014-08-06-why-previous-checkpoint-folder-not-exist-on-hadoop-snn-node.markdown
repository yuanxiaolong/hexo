---
layout: post
title: "why previous.checkpoint folder not exist on hadoop snn node"
date: 2014-08-06 22:41:59 +0800
comments: true
categories: hadoop
tags: hadoop
share: true
description: 分析为何在hadoop snn节点上，没生成previous.checkpoint文件夹
toc: true
---

分析为何在hadoop snn（secondary namenode）节点上，没生成previous.checkpoint文件夹

<!--more-->
偶然发现，在《权威指南》第二版中文版，第10章p296-p297，给出的目录结构，并没有在我本机伪分布式出现。

少了一个previous.checkpoint目录，为何？随着深挖，发现了一些有意思的事情<font color="#7c837f">（我用的是hadoop-1.2.1）</font>

---

## 开启debug，进行debug本机

(关于debug的开启，可以参考 [<font color="#6868b4">这里</font>](http://blog.yuanxiaolong.cn/blog/2014/07/21/how-to-debug-hadoop-on-local/)）

1.由于是在snn节点上没有生成previous.checkpoint目录，因此，我们需要debug这个节点。
![](/images/hadoop/20140806/debug-snn.png)

2.进入snn的类，checkpoint是一个周期性的事情，因此必须由定时器执行线程的方式来处理，因此逻辑都在守护线程里
![](/images/hadoop/20140806/snn-java.png)

3.进入守护线程，发现startCheckpoint方法，分析它
![](/images/hadoop/20140806/thread-startcheckpoint.png)

4.一起看一下startCheckpoint做了什么事情
![](/images/hadoop/20140806/checkpoint-localfolder.png)

5.分析完startCheckpoint后，跳出它，逐个分析其他函数，发现doMerge又是一个核心逻辑方法，分析它
![](/images/hadoop/20140806/domerge1.png)
![](/images/hadoop/20140806/domerge2.png)
![](/images/hadoop/20140806/domerge3.png)
![](/images/hadoop/20140806/domerge4.png)
![](/images/hadoop/20140806/domerge5.png)

6.刚才在doMerge方法里，已经看到snn生成了previous.checkpoint目录，现在进入守护线程的最后一段逻辑 endCheckpoint方法
![](/images/hadoop/20140806/endcheckpoint1.png)
![](/images/hadoop/20140806/endcheckpoint2.png)
![](/images/hadoop/20140806/endcheckpoint3.png)
![](/images/hadoop/20140806/endcheckpoint4.png)

---

## 结论
<font color="#541f8c">
至此我们已经可以得出结论，是由于调用了2次moveLastCheckpoint (doMerge一次，endCheckpoint一次)，导致第一次生成的previous.checkpoint被第二次删除了，
由于代码执行的很快，让我们感觉snn上的previous.checkpoint目录从未存在过一样，由于我是debug，因此可以看到变化。
</font>

---

## 进而深挖

这个是hadoop的一个bug，从0.20到至今一直有这个问题，以下是BUG说明。
![](/images/hadoop/20140806/apache-snn-bug1.png)
![](/images/hadoop/20140806/apache-snn-bug2.png)
![](/images/hadoop/20140806/apache-snn-bug3.png)
