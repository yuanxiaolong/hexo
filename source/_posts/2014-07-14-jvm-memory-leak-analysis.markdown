---
layout: post
title: "jvm memory leak analysis"
date: 2014-07-14 21:57:59 +0800
comments: true
category: jvm
tags: jvm
share: true
description: 排查jvm内存泄露
toc: true
---

一次jvm查错经历，分享给大家
<!--more-->

---

## 背景
C/S模型，Client和Server之间进行通信。Client定时发送心跳消息给Server，利用netty框架

---

## 环境准备(可选)
原始代码很多，不过我为了说明问题，用netty写了一个demo，旨在分析的过程。

<font color="#888689"> 可以通过git把源码下载下来，导入eclipse，设置一下build path，添加里面的netty jar包路径。</font>

``` bash
git clone git@github.com:yuanxiaolong/netty.git  yourLocalDirectory
```

当然我是已经知道这个问题，现在是分析这个BUG的过程。

---

## 开始分析

客户端 核心代码如下，将 *<font color="green">行A(line29)</font>* 注释掉， *<font color="green">行B(line30)</font>* 打开，模拟问题

``` java
//创建Client端线程池工厂
private static final NioClientSocketChannelFactory FACTORY = new NioClientSocketChannelFactory(
		Executors.newCachedThreadPool(), Executors.newCachedThreadPool());

//客户端启动器
public static final ClientBootstrap bootstrap = new ClientBootstrap(FACTORY);

//服务器地址
public static final InetSocketAddress remoteServerAddress = new InetSocketAddress("127.0.0.1", 8080);

private static final HashedWheelTimer timer = new HashedWheelTimer();

private static ReadTimeoutHandler timeoutHandler = new ReadTimeoutHandler(timer,3);

//客户端执行服务器返回response的处理链pipeline
//这里是连接失败时导致大量TCP连接出现的原因,http://javatar.iteye.com/blog/1138527
//由于外层调用采用单线程池,这里又将pipeline静态化,实验查看后并未出现TCP大量存在的情况
private static final ChannelPipelineFactory CHANNEL_PIPELINE_FACTORY = new ChannelPipelineFactory() {

	@Override
	public ChannelPipeline getPipeline() throws Exception {
		//pipeline 顺序调用,类似web的请求filter
		ChannelPipeline pipleline = pipeline();
		pipleline.addLast("encode", new StringEncoder());
		pipleline.addLast("decode", new StringDecoder());
		/**
		 * 这里2选1,分别是行A和行B
		 */
		pipleline.addLast("timeout", timeoutHandler);//行A
		pipleline.addLast("timeout", new ReadTimeoutHandler(new HashedWheelTimer(),3));//行B

		pipleline.addLast("handler", new ClientHanlder());//客户端handler
		return pipleline;
	}
};

static{
	bootstrap.setPipelineFactory(CHANNEL_PIPELINE_FACTORY);
}
```

1.在eclipse中分别启动Server和Client
2.启动完后，运行jps查看Client进程号PID，并在任务管理器里观察内存情况（发现内存不断上涨）

``` bash
D:\>jps
6212 ServerMain
5960 org.eclipse.equinox.launcher_1.3.0.v20130327-1440.jar
8908 Jps
8656 FirstClientMain
```

3.这时候其实内存泄露已经发生了（我们发现内存不断上涨，GC回收不掉），因此需要dump内存

``` bash
jmap -dump:format=b,file=test.bin 8656
```

将此时的内存信息dump到 **test.bin** 文件里（一般来说需要dump多次，而且间隔开）。
由于我们的demo就是模拟内存泄露场景的，因此只需要dump一次即可

4.在test.bin同级目录下，放置 [<font color="#4274c3">ibm analyzer</font>](http://pan.baidu.com/s/1ntzBeAH)
启动它

``` bash
java -Xmx1000m -jar ha442.jar test.bin
```

5.这时，我们就可以通过图形界面GUI来直观的看内存情况了。
![](/images/jvm/ibm_ha.png)

我们发现60%~70%的内存都是 <font color="#6111a3">org.jboss.netty.util.HashedWheelTimer</font>，定位出了。

6.至于怎么解决，就是另外一回事了。（ *<font color="green">行B</font>* 注释掉，打开 *<font color="green">行A</font>* 即可解决）

---

## 为什么会出现这样的现象？
用的不熟悉呗，我们看一下HashedWheelTimer的类说明就能明白了。

``` java
<h3>Do not create many instances.</h3>
 *
 * {@link HashedWheelTimer} creates a new thread whenever it is instantiated and
 * started.  Therefore, you should make sure to create only one instance and
 * share it across your application.  One of the common mistakes, that makes
 * your application unresponsive, is to create a new instance in
 * {@link ChannelPipelineFactory}, which results in the creation of a new thread
 * for every connection.
```

---

## 本地开发、线上linux机排查
* 本地开发
  * 可以利用 **jconsole** 观察，一目了然。
* 线上linux
  * 可以先dump到线上机，然后scp到本地机器


PS：附上内存泄露的dump文件 [<font color="#4274c3">test.bin</font>](http://pan.baidu.com/s/1c0vjJNU)
