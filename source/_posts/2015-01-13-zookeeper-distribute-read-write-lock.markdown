---
layout: post
title: "zookeeper distribute read write lock"
date: 2015-01-13 23:02:26 +0800
comments: true
categories: zookeeper
tags: zookeeper
share: true
description: zookeeper分布式锁
toc: true
---
利用zookeeper实现分布式读写锁，来协调业务一致

<!--more-->

zookeeper实现读写锁,读并发,写等待其他进程释放，并获取锁，执行写逻辑

---

## 环境准备

zookeeper实例，单机或伪分布式，全分布式任选，可以参照我上篇文章搭建个伪分布式的。

利用如下命令，利用zk客户端连接到zk实例，其中2182是 `zoo.cfg`的clientPort

``` bash
/home/web/xiaolong.yuanxl/zookeeper-3.4.6-server-1//bin/zkCli.sh -server 127.0.0.1:2182

```

新建`读锁`和`写锁`的路径，`/lock/read/` 和 `/lock/write/`

``` bash
[zk: 127.0.0.1:2182(CONNECTED) 8] create /lock/read readLock
Created /lock/read

[zk: 127.0.0.1:2182(CONNECTED) 8] create /lock/write writeLock
Created /lock/write
```

由于互斥锁，只需要在不同进程上，create 相同路径的 znode 即可由zk保证，并发下只有1个进程获得锁，十分简单。

这里实现的是,读写锁（读并发、写等待）。

---

## 思想

* 引用 http://www.wuzesheng.com/?p=2609
	* 6.读写锁(Read/Write Lock)
  * 我们知道，读写锁跟互斥锁相比不同的地方是，它分成了读和写两种模式，多个读可以并发执行，但写和读、写都互斥，不能同时执行行。利用ZooKeeper，在上面的基础上，稍做修改也可以实现传统的读写锁的语义，下面是基本的步骤:
 	* 每个进程都在ZooKeeper上创建一个临时的顺序结点(Ephemeral Sequential) /locks/lock_${seq}
	* ${seq}最小的一个或多个结点为当前的持锁者，多个是因为多个读可以并发
	* 需要写锁的进程，Watch比它次小的进程对应的结点
	* 需要读锁的进程，Watch比它小的最后一个写进程对应的结点
	* 当前结点释放锁后，所有Watch该结点的进程都会被通知到，他们成为新的持锁者，如此循环反复

解释一下：

1.<font color="#9b1fa1">写并发</font>时：zk保证了 `Ephemeral Sequential` 序号自增，且应该根据时序，让节点依次得到 `写锁`.

例如有write-01、write-02、write-03，那么
write-02 需要等 write-01 写完后才能执行写动作, write-03 在 write-02后写，这样保持了业务的并发，类似“九连环” 。
场景，对订单付款，产生3个并发，那么 write-01 ，先向账户扣款，同时标记订单状态为已完成。那么 write-02 执行时候，就不会产生重复扣了。

2.<font color="#9b1fa1">读并发</font>时：由于每个进程并不关心当前其他进程在读什么，相反需要关心最后一个写的进程，不然并发时其他进程未写完时就读，就产生了脏读。

所以，这里要获取 write-MaxNo 并所有读进程，都对这个 write-MaxNo znode 设置 watch，当此 znode 写动作执行完后,触发 `deleteNode` 事件,回调到 read上的 watcher。

---

## 代码

Step 1.入口我们模拟 3个读锁 3个写锁的并发，利用线程池

``` java
public static final int corePoolSize = 5;
public static final int maximumPoolSize = 10;
public static final int keepAliveTime = 3;
public static final int maximumLinkedQueueSize = 200000;

// 固定线程池
private static final ThreadPoolExecutor pool = new ThreadPoolExecutor(
		corePoolSize, maximumPoolSize, keepAliveTime, TimeUnit.SECONDS,
		new LinkedBlockingQueue<Runnable>(maximumLinkedQueueSize),
		new ThreadPoolExecutor.AbortPolicy());

public static void main(String[] args) throws IOException {
	//先并发写锁
	for (int i = 0; i < 3; i++) {
		WriteLockThread thread = new WriteLockThread("Thread-Write-" + (i+1));
		pool.execute(thread);
	}
	//后在写锁基础上,并发读锁
	for (int i = 0; i < 3; i++) {
		ReadLockThread thread = new ReadLockThread("Thread-Read-" + (i+1));
		pool.execute(thread);
	}

}
```

Step 2.线程产生读锁

``` java
public class ReadLockThread implements Runnable {

	private static final int TIMEOUT = 3000;

	private static final String WRITE_LOCK_PATH = "/lock/write";

	private static final String WRITE_SIGNAL = "write-";

	private String name;

	public ReadLockThread(String name) {
		this.name = name;
	}

	@Override
	public void run() {
		ZooKeeper zkp = null;
		try {
			// 1.建立zk connect
			zkp = new ZooKeeper("116.211.20.207:2182", TIMEOUT, null);
			// 2.创建一个EPHEMERAL类型的节点，会话关闭后它会自动被删除
			zkp.create("/lock/read/read-", this.name.getBytes(),
					Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
			// 3.sleep 1秒,故意让其它线程也create这个znode,形成读并发的效果
			Thread.sleep(1000);
			// 4.查看当前写锁,并在最后一个写锁上注册watcher
			List<String> children = zkp.getChildren(WRITE_LOCK_PATH, null);
			String maxWriteLockName = maxSeq(children);
			// 5.在写锁上注册wathcer
			String writeLock = WRITE_LOCK_PATH + "/" + maxWriteLockName;
			Stat stat = zkp.exists(writeLock , new MyReadBusinessLogic(this.name,zkp));
			if (stat != null) {
				System.out.println("[read theard] register watcher in " + writeLock + " success!");
			}else {
				//很有可能在40行到44行这段时间内,写锁已经被释放了,所以,这里直接执行读逻辑
				System.out.println("lastest writeLock node has been delete, now I can direct execute read bussiness");
				doReadBussiness(this.name);
				zkp.close();
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
		}

	}

	//获取最大seq
	private static synchronized String maxSeq(List<String> list) {
		String maxSeq = "";
		int max = -1; //初始化
		if (list != null && !list.isEmpty()) {
			if (StringUtils.containsIgnoreCase(list.get(0), WRITE_SIGNAL)) {
				String m = StringUtils.remove(list.get(0), WRITE_SIGNAL);
				max = Integer.parseInt(m);
				maxSeq = list.get(0);
			}
			for (int i = 1; i < list.size(); i++) {
				if (StringUtils.containsIgnoreCase(list.get(i), WRITE_SIGNAL)) {
					int temp = Integer.parseInt(StringUtils.remove(list.get(i), WRITE_SIGNAL));
					if (max < temp){
						max = temp;
						maxSeq = list.get(i);
					}
				}
			}

		}
		return maxSeq;
	}

	//mock
	private static synchronized void doReadBussiness(String bussinessNo){
		System.out.println("do read My Bussiness! " + bussinessNo);
	}

}
```

Step 3.产生写锁线程

``` java
package com.yxl.lock;

import java.util.Collections;
import java.util.List;

import org.apache.commons.lang.StringUtils;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.ZooDefs.Ids;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;
/**
 * 创建线程,产生并发写
 *
 * @author xiaolong.yuanxl
 */
public class WriteLockThread implements Runnable {

	private static final int TIMEOUT = 3000;

	private static final String WRITE_LOCK_PATH = "/lock/write";

	private String name;

	public WriteLockThread(String name) {
		this.name = name;
	}

	@Override
	public void run() {
		ZooKeeper zkp = null;
		try {
			// 1.建立zk connect
			zkp = new ZooKeeper("116.211.20.207:2182", TIMEOUT, null);

			// 2.创建一个EPHEMERAL类型的节点，会话关闭后它会自动被删除
			String currentPath = zkp.create(WRITE_LOCK_PATH + "/write-", this.name.getBytes(),
					Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
			System.out.println("current: " + currentPath);

			// 3.sleep 6秒,故意让其它线程也create这个znode,形成并发的效果,同时与读锁也能够一起产生并发
			Thread.sleep(6000);

			// 4.查看当前写锁,并在前一个写锁上注册watcher,形成“九连环”
			List<String> children = zkp.getChildren(WRITE_LOCK_PATH, null);
			Collections.sort(children);//由于前缀一样,因此可以用自然排序
			String current = StringUtils.remove(currentPath, WRITE_LOCK_PATH + "/");
			int index = children.indexOf(current);

			// 4.1 如果存在前一个节点,则向前一个节点注册watcher,并在watcher里执行完逻辑后再close
			if (-1 != index && index != 0) {
				System.out.println(this.name + " mine: " + current + " preview: " + children.get(index-1));
				//前一个节点
				String previewNodePath = WRITE_LOCK_PATH + "/" + children.get(index-1);
				Stat stat = zkp.exists(previewNodePath , new MyWriteBusinessLogic(this.name,zkp));
				if (stat != null) {
					System.out.println("[write thread] register watcher in " + previewNodePath + " success!");
				}else {
					//很有可能在46行到50行这段时间内,前一个节点已经释放了锁,所以这里会注册不成功,由于当前节点已经是最小的了,所以可以直接执行逻辑
					System.out.println("preview node has been delete, now I can direct execute write bussiness");
					doWriteBussiness(this.name);
					zkp.close();
				}
			}else {
				//4.2 如果当前节点为第一个,则close
				Thread.sleep(3000);//这里需要等待3秒再关闭连接,形成阻塞,让其他线程注册Watcher成功
				zkp.close();
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally {

		}
	}

	//mock
	private static synchronized void doWriteBussiness(String bussinessNo){
		System.out.println("do write My Bussiness! " + bussinessNo);
	}

}
```

Step 4.读锁watcher回调的业务逻辑

``` java
package com.yxl.lock;

import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.Watcher.Event.EventType;

/**
 *  读锁watch回调
 *
 * @author xiaolong.yuanxl
 */
public class MyReadBusinessLogic implements Watcher{

	private ZooKeeper zkp;

	private String bussinessNo;

	public MyReadBusinessLogic(String bussinessNo,ZooKeeper zkp) {
		this.bussinessNo = bussinessNo;
		this.zkp = zkp;
	}

	@Override
	public void process(WatchedEvent event) {
		//如果前一个对象上的锁释放了,这里回调获取感知
		if (EventType.NodeDeleted.getIntValue() == event.getType().getIntValue()) {
			System.out.println("[callback read thread] max write node has been deleted, path: " + event.getPath());
			doReadBussiness(bussinessNo,event.getPath());
			//close
			try {
				zkp.close();
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}

	//mock
	private static void doReadBussiness(String bussinessNo,String path){
		System.out.println("path deleted! " + path + " do My read Bussiness! " + bussinessNo);
	}

}
```

Step 5.写锁watcher回调的业务逻辑

``` java
package com.yxl.lock;

import org.apache.zookeeper.WatchedEvent;

import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.Watcher.Event.EventType;

/**
 * 写锁watch回调
 *
 * @author xiaolong.yuanxl
 */

public class MyWriteBusinessLogic implements Watcher{

	private ZooKeeper zkp;

	private String bussinessNo;

	public MyWriteBusinessLogic(String bussinessNo,ZooKeeper zkp) {
		this.bussinessNo = bussinessNo;
		this.zkp = zkp;
	}

	@Override
	public void process(WatchedEvent event) {
		//如果前一个对象上的锁释放了,这里回调获取感知
		if (EventType.NodeDeleted.getIntValue() == event.getType().getIntValue()) {
			System.out.println("[callback write thread] preview node has been deleted, path: " + event.getPath());
			doBussiness(bussinessNo,event.getPath());
			try {
				//close
				zkp.close();
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}

	//mock
	private static void doBussiness(String bussinessNo,String path){
		System.out.println("path deleted! " + path + " do My write Bussiness! " + bussinessNo);
	}

}
```

Step 6.执行后输出类似如下，可见 read thread 同时对最后一个写锁znode `write-0000000040`添加 watcher，等待最大写锁znode写完毕close后，触发watcher执行逻辑
，同时写锁 `write-0000000040`向`write-0000000039` 加watcher，`write-0000000039`向`write-0000000038`加watcher，`write-0000000038`由于没有
前一个节点，不加wathcer

``` bash
current: /lock/write/write-0000000038
current: /lock/write/write-0000000039
current: /lock/write/write-0000000040
[read theard] register watcher in /lock/write/write-0000000040 success!
[read theard] register watcher in /lock/write/write-0000000040 success!
[read theard] register watcher in /lock/write/write-0000000040 success!
Thread-Write-2 mine: write-0000000040 preview: write-0000000039
Thread-Write-3 mine: write-0000000039 preview: write-0000000038
[write thread] register watcher in /lock/write/write-0000000038 success!
[write thread] register watcher in /lock/write/write-0000000039 success!
[callback write thread] preview node has been deleted, path: /lock/write/write-0000000038
path deleted! /lock/write/write-0000000038 do My write Bussiness! Thread-Write-3
[callback write thread] preview node has been deleted, path: /lock/write/write-0000000039
path deleted! /lock/write/write-0000000039 do My write Bussiness! Thread-Write-2
[callback read thread] max write node has been deleted, path: /lock/write/write-0000000040
[callback read thread] max write node has been deleted, path: /lock/write/write-0000000040
[callback read thread] max write node has been deleted, path: /lock/write/write-0000000040
path deleted! /lock/write/write-0000000040 do My read Bussiness! Thread-Read-3
path deleted! /lock/write/write-0000000040 do My read Bussiness! Thread-Read-2
path deleted! /lock/write/write-0000000040 do My read Bussiness! Thread-Read-1

```

PS: 所有代码在 [<font color="#388014">Github</font>](https://github.com/yuanxiaolong/ZookeeperDemo)上，clone后 运行 `mvn eclipse:eclipse`获取依赖
