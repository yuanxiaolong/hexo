---
layout: post
title: "docker redis master slave"
date: 2014-10-29 22:37:59 +0800
comments: true
categories: docker
tags: docker
share: true
description: 利用docker配置redis的master-slave
toc: true
---

介绍如何构建一个 redis2.x 的 master-slave 环境。

<!--more-->

redis是一个轻量级的nosql数据库，一般会视为缓存来使用。

redis3里新增了cluster功能，最小 3master - 3slave ，因此至少要用6个docker来运行，VPS上只有512M内存，

已经跑了不少容器了，没必要搞cluster，因此就没有用redis3 。感兴趣的同学可以参考[<font color="#2d58bd"> docker-redis-cluster </font>](https://github.com/Grokzen/docker-redis-cluster)

---

## 步骤

### 获取redis 镜像

``` bash
docker pull redis
```

### 数据文件
本地建立一个目录，以便保存 redis的数据文件。本文是 /root/app/workspace/redis/data

### 运行redis
以官网 redis 镜像为准，运行一个 redis 服务，命名为 redis-master。

| **命令**                                     |    **含义**                                                    |
| :------------------------------------------ | :-------------------------------------------           |
| <font color="#982c79">--name redis-master                      </font>   | <font color="#982c79">表示将这个容器命名为 redis-master </font>                            |
| <font color="#728621">-v /root/app/workspace/redis/data:/data  </font>   | <font color="#728621">表示容器内 /data 映射到 本地目录/root/app/workspace/redis/data </font> |
| <font color="#982c79">-d                                       </font>   | <font color="#982c79"> 表示后台运行                                               </font> |
| <font color="#728621">redis                                    </font>   | <font color="#728621">  表示 redis:lastest  是image的名称                         </font> |
| <font color="#982c79">redis-server --appendonly yes            </font>   | <font color="#982c79"> 表示容器启动后，需要执行的命令                </font> |


``` bash
[root@yuanxiaolong data]# docker run --name redis-master -v /root/app/workspace/redis/data:/data -d redis redis-server --appendonly yes
17073302366ef5ab3b634bc23b9029712de228f0713d640e0c1bb200f098cc96

[root@yuanxiaolong data]# docker logs 17
[1] 28 Oct 15:06:37.159 # You requested maxclients of 10000 requiring at least 10032 max file descriptors.
[1] 28 Oct 15:06:37.162 # Redis can't set maximum open files to 10032 because of OS error: Operation not permitted.
[1] 28 Oct 15:06:37.162 # Current maximum open files is 1024. maxclients has been reduced to 4064 to compensate for low ulimit. If you need higher maxclients increase 'ulimit -n'.
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 2.8.17 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in stand alone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 1
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[1] 28 Oct 15:06:37.169 # Server started, Redis version 2.8.17
[1] 28 Oct 15:06:37.169 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
[1] 28 Oct 15:06:37.170 * DB loaded from append only file: 0.000 seconds
[1] 28 Oct 15:06:37.170 * The server is now ready to accept connections on port 6379

```

### 构建slave
手动构建 自己的redis slave 镜像 Dockerfile 和 start-slave.sh


``` bash Dockerfile
FROM redis:latest
EXPOSE 6379
VOLUME ["/data"]


# RUN sysctl vm.overcommit_memory=1

ADD start-slave.sh /src/start-slave.sh

WORKDIR /src

RUN cd /src

# ENTRYPOINT ["./start-slave.sh", "--dir", "/data"]

CMD ["./start-slave.sh"]

#
# build:  docker build -t yuanxiaolong/redis-slave .
# run:    docker run -d -P --name=redis_slave --link=redis-master:redis_master yuanxiaolong/redis-slave
#
```


``` bash start-slave.sh (chmod 777)
#!/bin/bash
#
if [ -z "$REDIS_MASTER_PORT_6379_TCP_ADDR" ]; then
    echo "REDIS_MASTER_PORT_6379_TCP_ADDR not defined. Did you run with -link?";
    exit 7;
fi
# exec allows redis-server to receive signals for clean shutdown
#

echo "REDIS_MASTER_PORT_6379_TCP_ADDR -> " $REDIS_MASTER_PORT_6379_TCP_ADDR
echo "REDIS_MASTER_PORT_6379_TCP_PORT -> " $REDIS_MASTER_PORT_6379_TCP_PORT

exec /usr/local/bin/redis-server --slaveof $REDIS_MASTER_PORT_6379_TCP_ADDR $REDIS_MASTER_PORT_6379_TCP_PORT
````

``` bash
docker build -t yuanxiaolong/redis-slave .
```

看一下是否已经存在

``` bash
[root@yuanxiaolong redis]# docker images
REPOSITORY                       TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
yuanxiaolong/redis-slave         latest              fd02ab8e3953        58 seconds ago      110.8 MB
yuanxiaolong/java-service        latest              2873af70208c        4 days ago          356.3 MB
jwilder/nginx-proxy              latest              29e1c2ae84a5        5 days ago          261.6 MB
jwilder/docker-gen               0.3.4               9edd4b39cea0        6 days ago          116.9 MB
yuanxiaolong/centos-node-hello   latest              3e448b2e9bd5        7 days ago          464.3 MB
java                             6                   a6fb766b92ed        7 days ago          352.9 MB
mongo                            2.6                 97d8b6dd3b57        12 days ago         391.4 MB
node                             latest              eb23c5dc3891        2 weeks ago         728.9 MB
redis                            latest              cd8d56009b1f        2 weeks ago         110.8 MB
nginx                            1.7.5               d2d79aebd368        4 weeks ago         100.2 MB
```

### 查看主从同步
进入 redis-master ，放一些数据进去，以便观察主从同步的效果。（docker-enter.sh 可以参考我的[<font color="#2d58bd"> 这篇 </font>](http://blog.yuanxiaolong.cn/blog/2014/10/23/how-to-enter-a-running-docker-container/)blog）

``` bash
[root@yuanxiaolong redis]# sh ~/docker-enter.sh 17

root@17073302366e:~# redis-cli
127.0.0.1:6379> set hello world
OK
127.0.0.1:6379> get hello
"world"

```

运行 一个redis-slave容器，再看一下log。

其中 `--link` 表示，关联上 redis-master 容器，并关联后给它起个别名 叫 redis_master

``` bash
docker run -d -P --name=redis_slave --link=redis-master:redis_master yuanxiaolong/redis-slave
```

``` bash
[root@yuanxiaolong redis]# docker logs b8
REDIS_MASTER_PORT_6379_TCP_ADDR ->  172.17.0.47
REDIS_MASTER_PORT_6379_TCP_PORT ->  6379
[1] 28 Oct 16:29:38.442 # You requested maxclients of 10000 requiring at least 10032 max file descriptors.
[1] 28 Oct 16:29:38.443 # Redis can't set maximum open files to 10032 because of OS error: Operation not permitted.
[1] 28 Oct 16:29:38.443 # Current maximum open files is 1024. maxclients has been reduced to 4064 to compensate for low ulimit. If you need higher maxclients increase 'ulimit -n'.
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 2.8.17 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
(    '      ,       .-`  | `,    )     Running in stand alone mode
|`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
|    `-._   `._    /     _.-'    |     PID: 1
  `-._    `-._  `-./  _.-'    _.-'
|`-._`-._    `-.__.-'    _.-'_.-'|
|    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
|`-._`-._    `-.__.-'    _.-'_.-'|
|    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[1] 28 Oct 16:29:38.447 # Server started, Redis version 2.8.17
[1] 28 Oct 16:29:38.447 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
[1] 28 Oct 16:29:38.447 * The server is now ready to accept connections on port 6379
[1] 28 Oct 16:29:38.447 * Connecting to MASTER 172.17.0.47:6379
[1] 28 Oct 16:29:38.447 * MASTER <-> SLAVE sync started
[1] 28 Oct 16:29:38.448 * Non blocking connect for SYNC fired the event.
[1] 28 Oct 16:29:38.448 * Master replied to PING, replication can continue...
[1] 28 Oct 16:29:38.449 * Partial resynchronization not possible (no cached master)
[1] 28 Oct 16:29:38.450 * Full resync from master: 86c69ea08b60d1157f2997926143a1971455295f:1
[1] 28 Oct 16:29:38.556 * MASTER <-> SLAVE sync: receiving 33 bytes from master
[1] 28 Oct 16:29:38.556 * MASTER <-> SLAVE sync: Flushing old data
[1] 28 Oct 16:29:38.556 * MASTER <-> SLAVE sync: Loading DB in memory
[1] 28 Oct 16:29:38.556 * MASTER <-> SLAVE sync: Finished with success
```
可以看到，主从已经同步了。此时，进入redis-slave容器，可以发现已经有在 master上放置的数据了。

``` bash
[root@yuanxiaolong redis]# sh ~/docker-enter.sh b8

root@b800227c5a99:~# redis-cli
127.0.0.1:6379> keys *
1) "hello"
127.0.0.1:6379> get hello
"world"

```
进入我们映射保存数据的文件夹，发现数据已经产生。

``` bash
[root@yuanxiaolong data]# ll
total 16
drwxr-xr-x 2  999 root 4096 Oct 28 12:29 .
drwxr-xr-x 3 root root 4096 Oct 28 12:28 ..
-rw-r--r-- 1  999  999   58 Oct 28 10:42 appendonly.aof
-rw-r--r-- 1  999  999   33 Oct 28 12:29 dump.rdb
[root@yuanxiaolong data]# cat appendonly.aof
*2
$6
SELECT
$1
0
*3
$3
set
$5
hello
$5
world
```

---

## 后记

Q：<font color="#c46a1d">为什么slave能找到master的 host、ip 呢？</font>

A：在回答这个问题前,我们先来看一下，如果再 link 一下master，会有什么东东，我们可以使用的

``` bash
[root@yuanxiaolong data]# docker run --link=redis-master:redis_master -i -t --name=redis-cli redis /bin/bash

root@617c83cfbd64:/data# env
HOSTNAME=617c83cfbd64
REDIS_MASTER_ENV_REDIS_DOWNLOAD_SHA1=913479f9d2a283bfaadd1444e17e7bab560e5d1e
TERM=xterm
REDIS_DOWNLOAD_URL=http://download.redis.io/releases/redis-2.8.17.tar.gz
REDIS_MASTER_ENV_REDIS_DOWNLOAD_URL=http://download.redis.io/releases/redis-2.8.17.tar.gz
REDIS_MASTER_ENV_REDIS_VERSION=2.8.17
REDIS_MASTER_PORT_6379_TCP_ADDR=172.17.0.47
REDIS_MASTER_NAME=/redis-cli/redis_master
REDIS_MASTER_PORT_6379_TCP=tcp://172.17.0.47:6379
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PWD=/data
HOME=/
SHLVL=1
REDIS_VERSION=2.8.17
REDIS_DOWNLOAD_SHA1=913479f9d2a283bfaadd1444e17e7bab560e5d1e
REDIS_MASTER_PORT_6379_TCP_PORT=6379
REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
REDIS_MASTER_PORT=tcp://172.17.0.47:6379
_=/usr/bin/env
root@617c83cfbd64:/data#
```

对！是用的环境变量。由于<font color="green"> `--link=redis-master:redis_master` </font> 后，
指定了别名为 redis_master，所以环境变量里，跟master有关的都以 REDIS_MASTER 开头了，这样能找到host、port。做主从只是一个命令而已。

Q：<font color="#c46a1d">为什么不直接连主从，而需要用 start-slave.sh 做“代理”呢？</font>

A：是的，可以直接连主从，因为 docker inspect master 完全可以获取master的ip、port。直接 docker run slave ... --slaveof masterIp masterPort 就可以。

主要是因为，如果把逻辑封装到脚本里，那么以后有任何变化，则对 “使用者” 透明，只需要run就可以，命令不变，改动的只是脚本。个人建议，以后构建自己的镜像时，
都这样做，这种设计是通用的。
