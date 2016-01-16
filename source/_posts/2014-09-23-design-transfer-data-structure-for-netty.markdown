---
layout: post
title: "design transfer data structure for netty"
date: 2014-09-23 22:56:39 +0800
comments: true
categories: netty
tags: netty
share: true
description: 设计netty的数据结构，用于传输
toc: true
---
介绍如何设计一种数据结构，满足向下兼容，又能扩展有弹性，且简单有效，利用netty。

<!--more-->

netty是一个高效的nio框架，让使用者脱离nio的一些底层细节，常被用来作为一种高性能网络传输的解决方案。

## 背景介绍

用netty做传输，netty最常用的有2种字节流包装方式，一种叫<font color="green"> StringEncoder/StringDecoder </font>，另一种叫
<font color="green"> ObjectEncoder/ObjectDecoder </font>。

最开始必然考虑，如何扩展的问题，因此传输格式用json的方式，而包装器用String的形式，这样满足可扩展，且无需装包、拆包，而且最主要的由于
Client和Server都是java编写的，因此无须再引用google protobuf这样的第三方序列化工具。

随着系统越来越成熟，所有信息用一个大json来搞，确实从代码可读性和维护上说是不太良好的，尽管注释已经很充分了


## 问题1

遇到了第一个问题，由于netty自带缓冲的byte buffer，类似TCP的Nagle算法的道理。会偶现，一个完整的json String 虽然write了进去，但是，如果在短时间内频繁write后，netty就“拆开”发送... ，这样Server端获取到数据时 <font color="#c97911"> JsonObject.fromObject(String) </font>就会抛异常。

解决这个问题的办法，可以让write之间的间隔加大一些，例如：<font color="#c97911">Thread.sleep(100) </font>。但是这个不是解决问题的根本之道，因此选择了另一种包装器Object，让netty自己“自动识别” 字节流的边界，有点类似TCP的帧 ，的情况那么个意思。

## 问题2

采用了Object来传输，果然没有出现这样的问题。最初的对象设计

``` java
public class MessageVO{
  private String fieldA;
  private String fieldB;
  //等等属性
}

```
这样仅将json的key/value结构拉平到了一个对象里，但是自测的时候，发现了问题。<font color="#823cb1"> java.io.EOFException </font>

当客户端的版本低于服务器的版本时，即Client的MessageVO里有5个字段，而Server的MessageVO里有10个字段，就会出现这样的异常。为什么会出现这样的异常？

先来看一张图
![](/images/netty/20140923/1.png)

由于Client的字段少于Server，因此当Client发送完毕时，Server将byte序列化成Object时，认为还没有完，因此就会继续构造D这个字段，因此当读到EOF时，就抛出了异常。

好了，知道了问题所在，就是解决一个序列化的问题，还要可扩展。后来修改的对象

``` java
public class MessageVO{
  private String type;
  private Object detailVO;
}
```

这样，由于Cilent和Server都是2个字段，因此，不会出现序列化问题。而且，增加一个type的值，就新增一种detailVO的类，满足了添加扩展，那同一个type下的，内部修改扩展问题呢？

答案是，根据type强转成子类型，而子类型里包含JsonObject就可以了。

``` java
public class DetailVO{
  private String fixField;//一些固定属性
  private JsonObject ext;//扩展属性
}
```

如果，DetailVO里还是跟第一种设计一样，拉平所有属性，那么本质上还是在组合属性，Client传输的字段少于Server需要的时候，那么还是会出现EOFException，就跟第一层for循环没抛出异常，但到第二层for循环的时候，就会抛出来。

就跟下面的图一样，传输的时候序列化是可以对得上的，唯一的不同就是Client的jsonObject力put的对象，少于Server端，而且我们还可以在jsonObject里添加一个version这样的字段，用于标识出不同版本的Client，以便区分不同的业务逻辑。

![](/images/netty/20140923/2.png)

---

## 总结

第一次设计“感觉上”就有问题，因为违反了“开闭原则”，对扩展开放，对修改关闭。因此两边的协议，应该是固定的，例如上面只有2个字段，一个type，一个Object。
