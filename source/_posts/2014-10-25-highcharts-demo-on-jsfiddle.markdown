---
layout: post
title: "highcharts demo on jsfiddle"
date: 2014-10-25 23:13:14 +0800
comments: true
categories: highcharts
tags: highcharts
share: true
description: highcharts的一个demo
toc: true
---

一个highcharts的demo，运行在jsfiddle上

<!--more-->

最近要搞个数据展现，好像专业些用R语言来绘图。但我暂时不会R，而且就是普通的线状图，考虑了一下用highcharts这个js插件吧。

---

## Jsfiddle

由于之前也没接触过[<font color="#2d58bd">Highcharts</font>](http://www.highcharts.com/)官网可能打不开，用 [Highcharts中文网](http://www.hcharts.cn/)一样的)，因此遇到了很多问题。在[<font color="#2d58bd">StackOverFlow</font>](http://stackoverflow.com/)
上 也翻阅了很多资料。发现大家更喜欢用另外一种方式来表达，就是[<font color="#2d58bd">Jsfiddle</font>](http://jsfiddle.net/)

废话不多说，直接上 [<font color="#2d58bd">demo</font>](http://jsfiddle.net/yuanxiaolong/fjL9kLzr/13/) (由于国内墙的原因，可能打开网页有些慢，请耐心等待)。

<iframe width="100%" height="550" src="http://jsfiddle.net/yuanxiaolong/fjL9kLzr/13/embedded/result/" ></iframe>


Highcharts 资料很全面，所以上手比较容易。作为一个纯javascript的图形插件，是你不二的选择，因此本文也就对使用方法不做赘述。


要说的是，Jsfiddle 由于安全性考虑，你是不能在 Jsfiddle 的 js 代码里，发送 ajax 请求，并处理响应结果的。

不过，它提供了一个 API ，可以 Mock echo 你想要的 html 、json、xml 等。[<font color="#2d58bd">说明文档</font>](http://doc.jsfiddle.net/use/echo.html)

国内也有一个叫RunJs的网站，跟这个差不多，速度很快。不过只限于国内交流，如果放到StackOverFlow上，国外友人就不知道该怎么使用了……


demo里的一个选择框 是用的[<font color="#7b5139">bootstrap-select.js </font>](http://silviomoreto.github.io/bootstrap-select/)


---

## Highcharts

一般都是动态图，即不知道有多少条线，服务器从DB捞出数据后，用json返回数据。然后客户端js接收，并解析数据，展现图表。

我提供了一个 http 服务，可以帮助使用 Highcharts 的人，关注Js解析数据就好了。

### <font color="#4c548c"> API 1.用于Mock数据库中，存在的线条名称列表 </font>

* url地址 ： [ <font color="#af7d27">http://www.yuanxiaolong.cn:49160/highchartLinename </font>](http://www.yuanxiaolong.cn:49160/highchartLinename)
* 参数 ： 无
* 返回 ： List<String>
* 格式 ：json
* 返回值示例

``` json
{"lineNames":["a","b","c"]}
```

### <font color="#4c548c"> API 2.用于Mock数据库中，需要返回的线条数据 </font>

* url地址 ： [ <font color="#af7d27">http://www.yuanxiaolong.cn:49160/highchartData </font>](http://www.yuanxiaolong.cn:49160/highchartData)
* 参数 ： 无 或 lineName ，当无参传入时，返回所有线条数据
* 请求示例 ： http://www.yuanxiaolong.cn:49160/highchartData?lineName=a
* 返回 ： Map<String, List<String>>
* 返回说明 ：Map-Key是线条的名称，Map-Value是线条，其中列表中的每一个值，代表一个点。用<font color="green"> “#”号</font>分隔，Array[0]代表时间（X轴），Array[1]代表数值（Y轴）
* 格式 ：json
* 返回值示例

``` json
{
    "b": [
        "2014-10-23 03:00:32#223",
        "2014-10-23 06:00:32#623",
        "2014-10-23 09:00:32#323",
        "2014-10-23 12:00:32#643",
        "2014-10-23 15:00:32#1500",
        "2014-10-23 18:00:32#932",
        "2014-10-23 21:00:32#535"
    ],
    "c": [
        "2014-10-23 03:00:32#73",
        "2014-10-23 06:00:32#473",
        "2014-10-23 09:00:32#173",
        "2014-10-23 12:00:32#493",
        "2014-10-23 15:00:32#1350",
        "2014-10-23 18:00:32#782",
        "2014-10-23 21:00:32#385"
    ],
    "a": [
        "2014-10-23 03:00:32#133",
        "2014-10-23 06:00:32#533",
        "2014-10-23 09:00:32#233",
        "2014-10-23 12:00:32#553",
        "2014-10-23 15:00:32#1410",
        "2014-10-23 18:00:32#842",
        "2014-10-23 21:00:32#445"
    ]
}

```
