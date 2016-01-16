---
layout: post
title: "analysis hadoop load configuration"
date: 2014-08-31 20:31:06 +0800
comments: true
categories: hadoop
tags: hadoop
share: true
description: 分析hadoop装载配置过程
toc: true
---
我们都知道core-site.xml是hadoop的核心配置文件，那这个文件是如何装载进去的呢？

<!--more-->

我们一起分析一下这个装载配置过程


## 思路

如果我要分析配置加载过程，那么应该从配置使用着手，为什么？因为配置使用的时候已经加载好了，从而从NameNode入手。这样的逆向思维，顺藤摸瓜。


## 分析

1.我们从NameNode的main函数入手。

果然发现了 脚本启动  <font color="green" > -> </font> 进入namenode的main函数 <font color="green" > -> </font> new Configuration()

![](/images/hadoop/20140831/1.png)

看一下构造函数。

* 先执行static域，做了2件事，a.用weakHashMap建个缓存 b. 加载默认资源core-site.xml
* 调用辅助重载构造器，传入true。这个true会让程序读取core-site.xml 后面可以看到。

![](/images/hadoop/20140831/2.png)

进入添加资源方法，解释如下。调用reload。（reload自己看吧，里面就是清空properties）
![](/images/hadoop/20140831/3.png)

---

问题来了，既然构造方法里，只做了清空、添加资源等 准备工作，那什么时候做加载工作？

有经验的同学应该明白一种设计叫  lazy ，即使用的时候再加载。这样可以减少系统开销。

那么获取属性我们都知道用 get方法，无二话，直接看get。果然有个getProps方法
![](/images/hadoop/20140831/4.png)

![](/images/hadoop/20140831/5.png)

深入loadResources方法。这里的 if 条件里就是true了，通过无参构造器调用重载构造器，进行的赋值。
![](/images/hadoop/20140831/6.png)

看到dom4j，就真相大白了。
![](/images/hadoop/20140831/7.png)

PS:还有另一种思路，简单粗暴效果好。凭经验，这种xml解析就是用dom4j的，全文搜索一下dom4j的类。从最底下向上打，不出5分钟就明白了。
