---
layout: post
title: "docker with Nodejs"
date: 2014-10-22 21:59:52 +0800
comments: true
categories: docker
tags: docker
share: true
description: docker部署nodejs应用
toc: true
---

利用docker部署nodejs应用

<!--more-->

将docker官网的example，自己做了一遍。例子很好，[<font color="#2d58bd">地址这里</font>](http://docs.docker.com/examples/nodejs_web_app/)


---

## 原理（个人理解）

[<font color="#2d58bd">上一篇</font>](http://blog.yuanxiaolong.cn/blog/2014/10/20/try-docker/) 介绍了docker的一些概念，
因此，在实践之前，需要先讲一下原理，这样就可以理解了。

<font color="#ca9729">1.docker的核心概念为什么是“容器”而不是“镜像”？</font>

因为“镜像”是无状态的，对同一个镜像 *<font color="green">docker run</font>* 两次，将会产生2个“容器”。而“容器”之间是相互隔离的。

也即是说，第一次 *<font color="green">docker run centos</font>*，得到容器A，然后bash shell进入。*<font color="green">mkdir /app/myDir </font>*后退出。

第二次 *<font color="green">docker run centos</font>*，得到容器B，再bash shell进入后，*<font color="green">/app/myDir</font>* 是不存在的。

因此 A 和 B 是相互隔离的。

<font color="#ca9729">2.如何从“无状态”到“有状态” ？</font>

docker利用一个叫Dockerfile 的文件来描述，从一个“纯净”的镜像，执行哪些步骤（类似脚本的方式），最后产生一个“有状态”的镜像。

有点类似 “程序 = 数据结构 + 算法”的概念。数据结构 就是 “纯净”的镜像，算法就是 Dockerfile。

这样，每个人的Dockerfile 都不一样，因此，就泛化出来了，很多“实例镜像”。再 *<font color="green">docker run 实例镜像</font>*，那么就是有状态的了。

---

## 实践

<font color="#827981" size="2">前置推荐：（非必须）首先在VPS上 建立一个工作目录，这样方便归类，本文将在此目录做所有操作。</font>

``` bash
[root@yuanxiaolong nodejs]# pwd
/root/app/workspace/nodejs
```

Step 1.建立 nodejs的package.json，以便Nodejs利用npm下载依赖的模块。

``` bash
[root@yuanxiaolong nodejs]# cat package.json

{
  "name": "docker-centos-hello",
  "private": true,
  "version": "0.0.1",
  "description": "Node.js Hello world app on CentOS using docker",
  "author": "Daniel Gasienica <daniel@gasienica.ch>",
  "dependencies": {
    "express": "3.2.4"
  }
}
```

Step 2.编写逻辑文件

``` bash
[root@yuanxiaolong nodejs]# cat index.js

var express = require('express');

// Constants
var PORT = 8088;

// App
var app = express();
app.get('/', function (req, res) {
  res.send('Hello world\n');
});

app.listen(PORT);
console.log('Running on http://localhost:' + PORT);
```

Step 3.编写Dockerfile

``` bash
root@yuanxiaolong nodejs]# cat Dockerfile

# DOCKER-VERSION 0.3.4
FROM    centos:centos6

# Enable EPEL for Node.js
RUN     rpm -Uvh http://download.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm
# Install Node.js and npm
RUN     yum install -y npm

# Bundle app source
COPY . /src
# Install app dependencies
RUN cd /src; npm install

EXPOSE  8088
CMD ["node", "/src/index.js"]

```

Step 4.利用Dockerfile构建属于自己的镜像

``` bash
sudo docker build -t  yuanxiaolong/centos-node-hello  /root/app/workspace/nodejs
```

Step 5.检验，看是否产生了自定义的镜像

``` bash
[root@yuanxiaolong nodejs]# docker images

REPOSITORY                       TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
yuanxiaolong/centos-node-hello   latest              3e448b2e9bd5        18 hours ago        464.3 MB
```

Step 6.将宿主主机的一个端口映射到 docker 容器暴露的端口上。这里把本地的49160端口，映射到容器中的8088端口，并运行

``` bash
docker run -p 49160:8088 -d yuanxiaolong/centos-node-hello
```

Step 7.验证

``` bash

[root@yuanxiaolong ~]# docker ps
CONTAINER ID        IMAGE                                   COMMAND              CREATED             STATUS              PORTS                               NAMES
163ee06b3176        yuanxiaolong/centos-node-hello:latest   node /src/index.js   17 hours ago        Up 17 hours         8088/tcp, 0.0.0.0:49160->8080/tcp   mad_shockley

[root@yuanxiaolong nodejs]# curl -i localhost:49160
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 12
Date: Wed, 22 Oct 2014 09:26:46 GMT
Connection: keep-alive

Hello world


```
