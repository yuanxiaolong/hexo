---
layout: post
title: "How to enter a running docker container"
date: 2014-10-23 22:31:19 +0800
comments: true
categories: docker
tags: docker
share: true
description: 一种进入docker容器内部的方法
toc: true
---


介绍如何进入一个正在运行的 docker 容器

<!--more-->
利用 nsenter 来进入一个正在运行的docker容器，可能系统没有自带 <font color="#ca672d">nsenter</font>这个执行程序，需要编译源码获取。

编译依赖C编译器，建议 yum install gcc 安装一个gcc，如果已有则忽略。

---

## 步骤

Step 1.建立一个文件夹，准备编译源码，获取可执行程序

``` bash
[root@yuanxiaolong app]# mkdir -p /root/app/tmp
[root@yuanxiaolong app]# cd /root/app/tmp
```

Step 2.获取源码，解压，进入解压后文件夹。

``` bash
[root@yuanxiaolong tmp]# wget https://www.kernel.org/pub/linux/utils/util-linux/v2.24/util-linux-2.24.tar.gz
--2014-10-22 23:08:50--  https://www.kernel.org/pub/linux/utils/util-linux/v2.24/util-linux-2.24.tar.gz
Resolving www.kernel.org... 199.204.44.194, 198.145.20.140, 149.20.4.69, ...
Connecting to www.kernel.org|199.204.44.194|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7630306 (7.3M) [application/x-gzip]
Saving to: “util-linux-2.24.tar.gz”

100%[=========================================================>] 7,630,306   3.36M/s   in 2.2s

2014-10-22 23:08:53 (3.36 MB/s) - “util-linux-2.24.tar.gz” saved [7630306/7630306]

[root@yuanxiaolong tmp]# ls
util-linux-2.24.tar.gz
[root@yuanxiaolong tmp]# tar -zxf util-linux-2.24.tar.gz
[root@yuanxiaolong tmp]# ls
util-linux-2.24  util-linux-2.24.tar.gz
[root@yuanxiaolong tmp]# cd util-linux-2.24

```

Step 3.检测，编译。

``` bash
[root@yuanxiaolong util-linux-2.24]# ./configure --without-ncurses

## 如果没有C编译器会报错如下：
## checking for gcc... no
## checking for cc... no
## checking for cl.exe... no
## configure: error: in `/root/app/tmp/util-linux-2.24':
## configure: error: no acceptable C compiler found in $PATH
## See `config.log' for more details

[root@yuanxiaolong util-linux-2.24]# make

```

Step 4.拷贝到可执行程序 到/usr/local/bin

``` bash
[root@yuanxiaolong util-linux-2.24]# cp nsenter /usr/local/bin
```

Step 5.编写脚本 docker-enter，以便利用nsenter进入docker 容器

``` bash
[root@yuanxiaolong util-linux-2.24]# cd ..
[root@yuanxiaolong tmp]# vi docker-enter

#!/bin/sh

if [ -e $(dirname "$0")/nsenter ]; then
    # with boot2docker, nsenter is not in the PATH but it is in the same folder
    NSENTER=$(dirname "$0")/nsenter
else
    NSENTER=nsenter
fi

if [ -z "$1" ]; then
    echo "Usage: `basename "$0"` CONTAINER [COMMAND [ARG]...]"
    echo ""
    echo "Enters the Docker CONTAINER and executes the specified COMMAND."
    echo "If COMMAND is not specified, runs an interactive shell in CONTAINER."
else
    PID=$(docker inspect --format "{{.State.Pid}}" "$1")
    if [ -z "$PID" ]; then
        exit 1
    fi
    shift

    OPTS="--target $PID --mount --uts --ipc --net --pid --"

    if [ -z "$1" ]; then
        # No command given.
        # Use su to clear all host environment variables except for TERM,
        # initialize the environment variables HOME, SHELL, USER, LOGNAME, PATH,
        # and start a login shell.
        "$NSENTER" $OPTS su - root
    else
        # Use env to clear all host environment variables.
        "$NSENTER" $OPTS env --ignore-environment -- "$@"
    fi
fi

[root@yuanxiaolong tmp]# chmod +x docker-enter

```

Step 6.查看正在运行的容器，然后利用脚本进入

``` bash
[root@yuanxiaolong ~]# docker ps
CONTAINER ID        IMAGE                                   COMMAND              CREATED             STATUS              PORTS                NAMES
64e466932bdd        yuanxiaolong/centos-node-hello:latest   node /src/index.js   3 hours ago         Up 3 hours          8088/tcp             prickly_shockley
496861d3a50d        yuanxiaolong/centos-node-hello:latest   node /src/index.js   3 hours ago         Up 3 hours          8088/tcp             backstabbing_nobel  
f7007b2bb4cb        jwilder/nginx-proxy:latest              forego start -r      3 hours ago         Up 3 hours          0.0.0.0:80->80/tcp   determined_nobel

[root@yuanxiaolong tmp]# sh docker-enter f7007b2bb4cb
root@f7007b2bb4cb:~#

```

PS: 个人喜欢把 docker-enter 这个脚本 放到 ~ 目录下，这样每次用 <font color="green"> sh ~/docker-enter 容器ID </font>就可以直接进入了
