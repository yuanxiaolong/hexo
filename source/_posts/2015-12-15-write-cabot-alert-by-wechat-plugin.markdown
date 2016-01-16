---
layout: post
title: "write cabot alert by wechat plugin"
date: 2015-12-15 23:40:35 +0800
comments: true
categories: cabot
tags: cabot
description: cabot 微信报警插件
toc: true
---

介绍如何利用 微信 和 Cabot 结合进行报警

<!--more-->

有了上一篇文章的 自定义Cabot扩展点插件 的基础后，我们就可以实现 自己的「微信告警插件了」

---

## 简介

利用「微信企业号」进行报警 ，微信企业号的注册过程，请参考这篇文章 [<font color="#2798a2">zabbix如何实现微信报警</font>](http://wuhf2015.blog.51cto.com/8213008/1688614)

微信发送原理：
  1. 根据微信企业号属性 CorpId 和 Secret 向服务接口，获取本次请求 Token
  2. 携带刚才返回的 Token ，向消息接口发送 post 请求

效果：

![](/images/cabot/20151215/1.png)

---

## 插件安装

前置：

* 已进入cabot的安装目录，例如 ` /usr/local/datacenter/cabot `
* 已停止 cabot 相关进程，有2组进程。
* 消息队列处理进程 `ps -ef | grep python | grep celery` 10个进程
* UI 进程 ` ps -ef | grep python | grep manage.py ` 1个进程（或许你有其他 django 应用在运行，请自己通过端口区分）


如果你实在记不清，也可以通过启动日志查看，类似这样的日志，其中 celery 有10个进程，注意需要逐一关闭

``` bash
14:04:00 web.1 | started with pid 36652
14:04:00 celery.1 | started with pid 36653
```

* 已注册了一个 「微信企业号」，并有 「CorpID」和「Secret」及「应用ID」和「接收组ID」。如果记不住 「CorpID」和「Secret」可以在 微信公共号后台
`「设置」-> 「权限管理」->「你自己建的组名」` 最下面有
* 已通过 [<font color="#2798a2">微信企业号接口调试工具</font>](http://qydev.weixin.qq.com/debug) 调试OK了微信账号，附上 [<font color="#2798a2">发送接口说明</font>](http://qydev.weixin.qq.com/wiki/index.php?title=消息类型及数据格式)

1.编写配置文件,添加 插件名称（注意是下划线，不是工程名，而是里面的 模块名，由setup.py里指定的）, 并修改 `setup.py` 内的相关信息

``` bash
vi conf/development.env

# Plugins to be loaded at launch
CABOT_PLUGINS_ENABLED=cabot_alert_hipchat==1.7.0,cabot_alert_twilio==1.6.1,cabot_alert_email==1.3.1,cabot_alert_wechat==0.0.1

```

2.通过 pip 安装自定义插件

``` bash
pip install git+git://github.com/yuanxiaolong/cabot-alert-wechat.git
```

3.初始化数据库

``` bash
sh setup_dev.sh
```

4.无误后，启动 cabot

``` bash
nohup foreman start &
```

---

## 插件编写

官方更详细的 文档 [<font color="#2798a2"> http://cabotapp.com/dev/writing-alert-plugins.html</font>](http://cabotapp.com/dev/writing-alert-plugins.html)
