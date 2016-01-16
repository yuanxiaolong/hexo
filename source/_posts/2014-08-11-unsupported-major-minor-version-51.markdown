---
layout: post
title: "Unsupported major minor version 51"
date: 2014-08-11 22:05:35 +0800
comments: true
categories: java
tags: java
share: true
description: jdk不能向上兼容
toc: true
---

今天碰见一个普通的问题，期初以为是环境问题，后来才发现是maven依赖的问题。

<!--more-->
把过程分享给大家，解决问题很简单，定位问题不容易啊，因为方向考虑错了……

---

## 背景

1.  在centos上运行一个jetty启动的服务jar包。
2.  本机<font color="green"> windows，jdk1.7，maven3.1.1，eclipse </font>

本地写的jar包放到linux机上报错。

``` java
 java.lang.UnsupportedClassVersionError: javax/servlet/Servlet : Unsupported major.minor version 51.0
	at java.lang.ClassLoader.defineClass1(Native Method)
	at java.lang.ClassLoader.defineClassCond(ClassLoader.java:631)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:615)
	at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:141)
	at java.net.URLClassLoader.defineClass(URLClassLoader.java:283)
	at java.net.URLClassLoader.access$000(URLClassLoader.java:58)
	at java.net.URLClassLoader$1.run(URLClassLoader.java:197)
	at java.security.AccessController.doPrivileged(Native Method)
	at java.net.URLClassLoader.findClass(URLClassLoader.java:190)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:306)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:301)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:247)
Exception in thread "main"
```

---

## 分析过程

1.先百度google了一下，本质是高版本jdk编写的class文件在低版本jdk上不能运行。于是到linux上看了一下

``` bash
[hadoop@wh-9-103 ~]$ java -version
java version "1.6.0_20"
Java(TM) SE Runtime Environment (build 1.6.0_20-b02)
Java HotSpot(TM) 64-Bit Server VM (build 16.3-b01, mixed mode)
```

果然 1.7的jdk程序不能在1.6上运行。。。。

2.卸载jdk1.7，重装jdk1.6，本机运行。同样报错如上，很奇怪，本机是1.6的jdk。
而之前1.7的jdk本机运行良好，怀疑是不是运行程序时，找之前的jdk1.7？

百度后，有人说windows注册表里还是1.7jdk，因此出错。重装3遍无果....

3.运行时候加上 vm参数 <font color="#0ea373"> -verbose </font>，运行main，发现还是一样，但可以看出，已经是加载jdk1.6的了。

4.从异常栈出发，<font color="#0c22a1"> javax/servlet/Servlet </font>肯定是这里的问题，maven看一下依赖。跟这个类有关的竟然有2个jar

*  javax.servlet-api:3.1.0
*  servlet-api:2.5

![](/images/java/20140811/mvn-dep.png)


一般是2.5的那个，而3.1的那个是从哪里依赖过来的？

原来是jetty-server，高版本的它依赖了最后面用jdk1.7编译的jar。因此换个低版本的jetty-server就好了。
