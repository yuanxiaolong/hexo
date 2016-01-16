---
layout: post
title: "redis proxy for ip change"
date: 2014-11-11 22:12:42 +0800
comments: true
categories: docker
tags: docker
share: true
description: 利用代理模式，缓解master重启后slave主从不同步的问题
toc: true
---

redis 当master重启后，则docker ip 变化，slave就无法主从同步。利用代理模式可以缓解这个。

<!--more-->

由于每次run、restart 容器后，容器的ip就会变化。因此 redis的 slaveOf=< old master ip> 就会引起主从不同步的问题。

所以，这里引入代理模式。可以缓解这个问题，思路是这样的，在不稳定的机器上部署代理。本质是这样的：master->proxy<-slave
因此，slave永远获取的是proxy的ip，而master ip change会被proxy监听到，对slave 透明。

---

## 步骤

### 启动master

```
docker run --name redis-master -v /root/app/workspace/redis/data:/data -d --expose 6379 redis redis-server --appendonly yes
```

### 启动代理

利用 <font color="#9f4074"> cpuguy83/docker-grand-ambassador </font>

```
docker run -d -v /var/run/docker.sock:/var/run/docker.sock  --name redis_ambassador  cpuguy83/docker-grand-ambassador -name redis-master
```

### 构建slave

还是利用上一篇Blog的Dockerfile，但这次slave启动的时候link的是proxy

```

[root@yuanxiaolong redis]# docker build -t yuanxiaolong/redis-slave .

[root@yuanxiaolong redis]# docker run -d -P --name=redis_slave --link=redis_ambassador:redis_ambassador yuanxiaolong/redis-slave
cbaeabd5d49f87660d4b53ae812f2ea04ba21cfe54b304e37d478137f6f5cb88
[root@yuanxiaolong redis]# docker logs cb
redis_ambassador's ip ->  172.17.0.76
[1] 05 Nov 09:58:31.666 # You requested maxclients of 10000 requiring at least 10032 max file descriptors.
[1] 05 Nov 09:58:31.666 # Redis can't set maximum open files to 10032 because of OS error: Operation not permitted.
[1] 05 Nov 09:58:31.667 # Current maximum open files is 1024. maxclients has been reduced to 4064 to compensate for low ulimit. If you need higher maxclients increase 'ulimit -n'.
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

[1] 05 Nov 09:58:31.668 # Server started, Redis version 2.8.17
[1] 05 Nov 09:58:31.668 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
[1] 05 Nov 09:58:31.668 * The server is now ready to accept connections on port 6379
[1] 05 Nov 09:58:31.668 * Connecting to MASTER 172.17.0.76:6379
[1] 05 Nov 09:58:31.668 * MASTER <-> SLAVE sync started
[1] 05 Nov 09:58:31.670 * Non blocking connect for SYNC fired the event.
[1] 05 Nov 09:58:31.671 * Master replied to PING, replication can continue...
[1] 05 Nov 09:58:31.671 * Partial resynchronization not possible (no cached master)
[1] 05 Nov 09:58:31.673 * Full resync from master: 0694696a66ef4e6dce92dbbf66c03cb46cc44922:1
[1] 05 Nov 09:58:31.689 * MASTER <-> SLAVE sync: receiving 48 bytes from master
[1] 05 Nov 09:58:31.689 * MASTER <-> SLAVE sync: Flushing old data
[1] 05 Nov 09:58:31.689 * MASTER <-> SLAVE sync: Loading DB in memory
[1] 05 Nov 09:58:31.689 * MASTER <-> SLAVE sync: Finished with success

```

### 验证
验证一下现在的数据，现在的ip，重启，重启后的ip

```

[root@yuanxiaolong redis]# sh ~/docker-enter.sh cb
root@cbaeabd5d49f:~# redis-cli
127.0.0.1:6379> keys *
1) "hello"
2) "yuan"

[root@yuanxiaolong redis]# docker-ip 26
172.17.0.97
[root@yuanxiaolong redis]# docker restart 26
26
[root@yuanxiaolong redis]# docker-ip 26
172.17.0.106
```

### 重启master
重启master后，给master放置数据

```

[root@yuanxiaolong redis]# sh ~/docker-enter.sh 26
root@26d8e94cc2b1:~# redis-cli keys *
1) "yuan"
2) "hello"
root@26d8e94cc2b1:~# redis-cli
127.0.0.1:6379> set good job
OK
127.0.0.1:6379> keys *
1) "good"
2) "yuan"
3) "hello"
```

### 查看slave
进入slave，发现主从已同步。（之前不引用代理，重启master，slave里是没有新数据的，而且slave的log是找不到old master ip的）

```

[root@yuanxiaolong redis]# sh ~/docker-enter.sh cb
root@cbaeabd5d49f:~# redis-cli
127.0.0.1:6379> keys *
1) "hello"
2) "good"
3) "yuan"
127.0.0.1:6379> get good
"job"
```

我们看一下 proxy 的日志，就能发现 master重启后，proxy监听并代理的新的ip。（我之前重启了master 2次，所以ip从 75 -> 97，再 97 -> 106 ）

```
[root@yuanxiaolong redis]# docker ps
CONTAINER ID        IMAGE                                     COMMAND                CREATED             STATUS              PORTS                     NAMES
cbaeabd5d49f        yuanxiaolong/redis-slave:latest           /entrypoint.sh ./sta   12 minutes ago      Up 12 minutes       0.0.0.0:49168->6379/tcp   redis_slave
fb86c9b5d007        cpuguy83/docker-grand-ambassador:latest   /usr/bin/grand-ambas   About an hour ago   Up About an hour                              redis_ambassador,redis_slave/redis_ambassador  
26d8e94cc2b1        redis:latest                              /entrypoint.sh redis   About an hour ago   Up 10 minutes       6379/tcp                  redis-master  


[root@yuanxiaolong redis]# docker logs fb
time="2014-11-05T08:17:39Z" level="info" msg="Initializing proxy"
time="2014-11-05T08:17:39Z" level="info" msg="Proxying 172.17.0.75:6379/tcp"
time="2014-11-05T08:17:39Z" level="info" msg="Handling Events for: 26d8e94cc2b1c39ac4df7a0e7c8bc45abc4af01da6a684093b9070c2e7ecc6db: /redis-master"
time="2014-11-05T09:03:10Z" level="info" msg="Received event: &{26d8e94cc2b1c39ac4df7a0e7c8bc45abc4af01da6a684093b9070c2e7ecc6db stop}"
time="2014-11-05T09:03:10Z" level="info" msg="Stopping proxy on tcp/[::]:6379 for tcp/172.17.0.75:6379 (accept tcp [::]:6379: use of closed network connection)"
time="2014-11-05T09:03:10Z" level="info" msg="Received event: &{26d8e94cc2b1c39ac4df7a0e7c8bc45abc4af01da6a684093b9070c2e7ecc6db die}"
time="2014-11-05T09:03:44Z" level="info" msg="Received event: &{26d8e94cc2b1c39ac4df7a0e7c8bc45abc4af01da6a684093b9070c2e7ecc6db start}"
time="2014-11-05T09:03:44Z" level="info" msg="Closing old servers"
time="2014-11-05T09:03:44Z" level="info" msg="Servers closed"
time="2014-11-05T09:03:44Z" level="info" msg="Proxying 172.17.0.97:6379/tcp"
time="2014-11-05T10:00:13Z" level="info" msg="Received event: &{26d8e94cc2b1c39ac4df7a0e7c8bc45abc4af01da6a684093b9070c2e7ecc6db die}"
time="2014-11-05T10:00:13Z" level="info" msg="Received event: &{26d8e94cc2b1c39ac4df7a0e7c8bc45abc4af01da6a684093b9070c2e7ecc6db restart}"
time="2014-11-05T10:00:13Z" level="info" msg="Closing old servers"
time="2014-11-05T10:00:13Z" level="info" msg="Stopping proxy on tcp/[::]:6379 for tcp/172.17.0.97:6379 (accept tcp [::]:6379: use of closed network connection)"
time="2014-11-05T10:00:13Z" level="info" msg="Servers closed"
time="2014-11-05T10:00:13Z" level="info" msg="Proxying 172.17.0.106:6379/tcp"
time="2014-11-05T10:01:15Z" level="info" msg="Can't forward traffic to backend tcp/172.17.0.97:6379: dial tcp 172.17.0.97:6379: connection timed out\n"
```
