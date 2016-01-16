---
layout: post
title: "zookeeper and dubbo"
date: 2015-01-11 23:17:17 +0800
comments: true
categories: dubbo
tags: duboo
share: true
description: 利用zookeeper搭建dubbo
toc: true
---

安装Dubbo，并利用zookeeper作为注册中心

<!--more-->

Dubbo 是在阿里广泛应用的RPC框架，简单方便。zookeeper用于实现HA常用的一种方案。

---

## 环境准备:

Dubbo [<font color="#5365c6">首页</font>](http://alibaba.github.io/dubbo-doc-static/Home-zh.htm)

* zookeeper [<font color="#5365c6">download</font>](http://www.apache.org/dist/zookeeper/)
* tomcat6 [<font color="#5365c6">download</font>](http://www.apache.org/dist/tomcat/tomcat-6/v6.0.41/bin/)

---

## zookeeper Install 伪分布式

1.解压下载好的zookeeper，到服务器上，解压并重命名

``` bash
wget http://www.apache.org/dist/zookeeper/stable/zookeeper-3.4.6.tar.gz
tar zxvf zookeeper-3.4.6.tar.gz
mv zookeeper-3.4.6 zookeeper-3.4.6-server-1

```

2.进入conf目录，复制配置文件，并修改参数

``` bash
cp zoo_sample.cfg zoo.cfg
vi zoo.cfg

tickTime=2000  
initLimit=5  
syncLimit=2  
dataDir=/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/data
dataLogDir=/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/logs
clientPort=2182
server.1=127.0.0.1:8880:7770  
server.2=127.0.0.1:8881:7771  
server.3=127.0.0.1:8882:7772

```

其中需要修改的 `dataDir` `dataLogDir` `clientPort` ，端口不能重复! 一般我喜欢在conf同级目录下 新建 `data` 和 `logs` 文件夹

* 引用 http://coolxing.iteye.com/blog/1871009
	* <b><font color="#323d89">initLimit</font></b>: zookeeper集群中的包含多台server, 其中一台为leader, 集群中其余的server为follower. initLimit参数配置初始化连接时, follower和leader之间的最长心跳时间. 此时该参数设置为5, 说明时间限制为5倍tickTime, 即5*2000=10000ms=10s.
	* <b><font color="#323d89">syncLimit</font></b>: 该参数配置leader和follower之间发送消息, 请求和应答的最大时间长度. 此时该参数设置为2, 说明时间限制为2倍tickTime, 即4000ms.
	* <b><font color="#323d89">server.X=A:B:C</font></b> 其中X是一个数字, 表示这是第几号server. A是该server所在的IP地址. B配置该server和集群中的leader交换消息所使用的端口. C配置选举leader时所使用的端口. 由于配置的是伪集群模式, 所以各个server的B, C参数必须不同.

3.进入 conf/zoo.cfg 里指定的 data 文件夹，新建`myid`文件，用于标识此zk实例是集群中的哪一个,该数字必须和 `zoo.cfg` 文件中的 `server.X` 中的X相对应.

``` bash
cd /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/data
echo "1" > myid
```

4.同理 cp 出 zk2 和 zk3 ，注意修改 data 、 logs、clientPort 、myid 这4个地方，然后依次启动zk ( 关闭stop 重启restart 状态status ) 并看启动情况

``` bash
cd /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/
sh zkServer.sh start
less ./zookeeper.out
```

5.这时查看zk的进程，是否是预期的3个，以及状态（2个foller、1个leader）

``` bash
[web@web02 bin]$ ps -ef | grep zoo
web        553     1  0 10:22 ?        00:00:01 /usr/java/jdk1.6.0_20/bin/java -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -cp /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/../build/classes:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/../build/lib/*.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/../lib/slf4j-log4j12-1.6.1.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/../lib/slf4j-api-1.6.1.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/../lib/netty-3.7.0.Final.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/../lib/log4j-1.2.16.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/../lib/jline-0.9.94.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/../zookeeper-3.4.6.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/../src/java/lib/*.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/../conf: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false org.apache.zookeeper.server.quorum.QuorumPeerMain /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/../conf/zoo.cfg
web        778     1  0 10:26 ?        00:00:01 /usr/java/jdk1.6.0_20/bin/java -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -cp /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-3/bin/../build/classes:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-3/bin/../build/lib/*.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-3/bin/../lib/slf4j-log4j12-1.6.1.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-3/bin/../lib/slf4j-api-1.6.1.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-3/bin/../lib/netty-3.7.0.Final.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-3/bin/../lib/log4j-1.2.16.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-3/bin/../lib/jline-0.9.94.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-3/bin/../zookeeper-3.4.6.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-3/bin/../src/java/lib/*.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-3/bin/../conf: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false org.apache.zookeeper.server.quorum.QuorumPeerMain /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-3/bin/../conf/zoo.cfg
web       1167     1  0 10:37 ?        00:00:01 /usr/java/jdk1.6.0_20/bin/java -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -cp /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-2/bin/../build/classes:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-2/bin/../build/lib/*.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-2/bin/../lib/slf4j-log4j12-1.6.1.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-2/bin/../lib/slf4j-api-1.6.1.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-2/bin/../lib/netty-3.7.0.Final.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-2/bin/../lib/log4j-1.2.16.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-2/bin/../lib/jline-0.9.94.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-2/bin/../zookeeper-3.4.6.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-2/bin/../src/java/lib/*.jar:/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-2/bin/../conf: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false org.apache.zookeeper.server.quorum.QuorumPeerMain /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-2/bin/../conf/zoo.cfg
web       6057  2023  0 15:47 pts/6    00:00:00 grep zoo

[web@web02 bin]$ sh /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/zkServer.sh status
JMX enabled by default
Using config: /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1/bin/../conf/zoo.cfg
Mode: follower

[web@web02 bin]$ sh /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-2/bin/zkServer.sh status
JMX enabled by default
Using config: /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-2/bin/../conf/zoo.cfg
Mode: follower

[web@web02 bin]$ sh /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-3/bin/zkServer.sh status
JMX enabled by default
Using config: /home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-3/bin/../conf/zoo.cfg
Mode: leader

```

---

## dubbo admin 安装（web管理工具）

由于dubbo官网已经打不开了，这里是我的云盘 [<font color="#5365c6">共享</font>](http://pan.baidu.com/s/1dDlI7aL)

官网给出的 [<font color="#d08432">zk安装</font>](http://alibaba.github.io/dubbo-doc-static/Administrator+Guide-zh.htm#AdministratorGuide-zh-Zookeeper%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83%E5%AE%89%E8%A3%85)

官网给出的 [<font color="#d08432">dubbo-admin安装</font>](http://alibaba.github.io/dubbo-doc-static/Administrator+Guide-zh.htm#AdministratorGuide-zh-%E7%AE%A1%E7%90%86%E6%8E%A7%E5%88%B6%E5%8F%B0%E5%AE%89%E8%A3%85)

1.下载并将 tomcat 解压，删除默认的 `webapp/ROOT` （也可以通过设置 tomcat的 server.xml 里 Context 来指定路径）

``` bash
wget http://www.apache.org/dist/tomcat/tomcat-6/v6.0.41/bin/apache-tomcat-6.0.41.tar.gz
tar zxvf apache-tomcat-6.0.41.tar.gz
rm -rf webapps/ROOT

```

2.下载并解压 dubbo-admin ，并修改配置 (我指定了其中的一个zk实例)，默认用户名和密码都是root

``` bash
mkdir ./apache-tomcat-6.0.41/webapps/ROOT
cp dubbo-admin-2.5.4.war ./apache-tomcat-6.0.41/webapps/ROOT
jar -xvf dubbo-admin-2.5.4.war
rm -rf dubbo-admin-2.5.4.war
```

``` bash
vi webapps/ROOT/WEB-INF/dubbo.properties

dubbo.registry.address=zookeeper://127.0.0.1:2182
dubbo.admin.root.password=root
dubbo.admin.guest.password=guest

```
3.浏览器访问，我这里修改了tomcat端口，默认http路由的是8080端口

![](/images/dubbo/20150109/1.png)

---

## dubbo Provider Consumer 示例

即官网 [<font color="#d08432">首页示例</font>](http://alibaba.github.io/dubbo-doc-static/Home-zh.htm)

### Provider

``` java DemoService.java
package com.yxl.provider;

public interface DemoService {

	 String sayHello(String name);

}

```
``` java DemoServiceImpl.java
package com.yxl.provider;

public class DemoServiceImpl implements DemoService {

	@Override
	public String sayHello(String name) {
		return "Hello " + name;
	}

}
```

Provider 配置文件

``` xml provider.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd        http://code.alibabatech.com/schema/dubbo        http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="hello-world-app"  />

    <!-- 使用zookeeper注册中心暴露服务地址 -->
    <dubbo:registry address="zookeeper://116.211.20.207:2182?backup=116.211.20.207:2183,116.211.20.207:2184" />

    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880" />

    <!-- 声明需要暴露的服务接口 -->
    <dubbo:service interface="com.yxl.provider.DemoService" ref="demoService" />

    <!-- 和本地bean一样实现服务 -->
    <bean id="demoService" class="com.yxl.provider.DemoServiceImpl" />

</beans>

```

启动 Provider

``` java Provider.java
package com.yxl.main;

import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Provider {

	public static void main(String[] args) throws Exception {
		ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(
				new String[] { "provider.xml" });
		context.start();

		System.in.read();
	}

}

```

### Consumer

为了便于测试,建立线程,轮询调用Provider

``` java LogicThread.java
package com.yxl.consumer;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.atomic.AtomicInteger;

import com.yxl.provider.DemoService;
import com.yxl.util.SpringBeanHelper;

public class LogicThread implements Runnable{

	private static AtomicInteger ai = new AtomicInteger(1);

	@Override
	public void run() {
		SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		DemoService demoService = (DemoService) SpringBeanHelper.getBean("demoService");
		String hello = demoService.sayHello("world");


		System.out.println(sdf.format(new Date()) + " = " + ai.getAndIncrement() + " : " + hello);
	}

}
```

建立个工具类，以便在线程里面获取bean

``` java SpringBeanHelper.java
package com.yxl.util;

import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;

public class SpringBeanHelper{

	private static  ApplicationContext context;

	/**
	 * 手工获取bean
	 */
	public static Object getBean(String beanId) {
		return context.getBean(beanId);
	}


	public static void setApplicationContext(ApplicationContext ctx)
			throws BeansException {
		context = ctx;
	}

}
```
Consumer配置文件

``` xml consumer.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans        http://www.springframework.org/schema/beans/spring-beans.xsd        http://code.alibabatech.com/schema/dubbo        http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
    <dubbo:application name="consumer-of-helloworld-app"  />

    <!-- 使用zookeeper注册中心暴露发现服务地址 -->
    <dubbo:registry address="zookeeper://116.211.20.207:2182?backup=116.211.20.207:2183,116.211.20.207:2184" />

    <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
    <dubbo:reference id="demoService" interface="com.yxl.provider.DemoService" />

</beans>
```


启动Consumer

``` java Consumer.java
package com.yxl.main;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

import org.springframework.context.support.ClassPathXmlApplicationContext;

import com.yxl.consumer.LogicThread;
import com.yxl.util.SpringBeanHelper;

public class Consumer {

	// 初始延迟1秒
	private static long INIT_DELAY = 1;

	// 30秒周期
	private static long PERIOD = 5;

	// 初始化1个定时线程池
	private static ScheduledExecutorService service = Executors
			.newScheduledThreadPool(1);


	public static void main(String[] args) throws Exception {
		ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(
				new String[] { "consumer.xml" });

		SpringBeanHelper.setApplicationContext(context);

		service.scheduleAtFixedRate(new LogicThread(), INIT_DELAY,
				PERIOD, TimeUnit.SECONDS);

		context.start();

	}
}

```

### <font color="#d16a3e">先启动Provider，再启动Consumer</font>

此时能看到有生产者和消费者,以及输出. 代码在 [<font color="#388014">Github</font>](https://github.com/yuanxiaolong/DubboTest) 你可以 `git clone` 然后 `mvn eclipse:eclipse`获取依赖,运行demo

![](/images/dubbo/20150109/2.png)
