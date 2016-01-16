---
layout: post
title: "nginx with docker service other containers"
date: 2014-10-24 23:23:00 +0800
comments: true
categories: docker
tags: docker
share: true
description: nginx的docker容器，处理其他容器
toc: true
---

介绍如何利用nginx的docker，来反向代理、负载均衡其他运行的容器。

<!--more-->

nginx 跟 apache类似，是一种负载均衡和反向代理的软件。当遇见docker，你会觉得一切都“自动化”了。

---

## 前言

可能有些同学初次接触ngnix，或不太清楚做什么的。这里还是啰嗦几句吧

比个例子，如果 请求 http://www.baidu.com ，其实请求的是，http://www.baidu.con:80 即80端口。

而我们知道，如果要形成分布式，那么 80 端口只允许一个 web 应用占用，假如有2个web应用都要使用80端口对外提供相同的服务，
那么，就出现了一个问题，到底给谁呢？

nginx就解决了这个问题，让nginx作为“代理”，独占80端口。并将请求路由转发到，后面的分布式应用上。

就像，nginx:80  =>  [appA(127.0.0.100:8080)] Or [appB(127.0.0.101:8080)]

因此，后面即使appB宕机了，那么也不影响网站的可用性，因为还有appA在对外服务，而且nginx保证不会把请求转发到宕机的机器上。

---

## 步骤

Step 1.下载封装后的nginx镜像（不是docker官方纯净的nginx）[<font color="#2d58bd">地址这里</font>](https://registry.hub.docker.com/u/jwilder/nginx-proxy/)

``` bash
docker pull jwilder/nginx-proxy
```

Step 2.直接运行nginx，把本地80端口映射到ngnix容器的80端口上。再看一下日志，有没有报错什么的。

```bash
[root@yuanxiaolong nginx]# docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock jwilder/nginx-proxy
f7007b2bb4cb37022f7d87bae49682c75aa70e2215e5586fcbd86098b7262d6e

[root@yuanxiaolong nginx]# docker logs f7
forego     | starting nginx.1 on port 5000
forego     | starting dockergen.1 on port 5100
dockergen.1 | 2014/10/23 02:46:18 Generated '/etc/nginx/sites-enabled/default' from 1 containers
dockergen.1 | 2014/10/23 02:46:18 Running 'nginx -s reload'
dockergen.1 | 2014/10/23 02:46:18 Watching docker events
nginx.1    | 61.172.240.228 - - [23/Oct/2014:02:46:28 +0000] "GET / HTTP/1.1" 503 614 "-" "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/38.0.2125.104 Safari/537.36"

```

Step 3.启动几个自己的docker应用，形成分布式。传递虚拟地址变量，<font color="#e07115">(这里就需要你有个DNS域名)</font>，我启动的是我
[<font color="#2d58bd"> 上上一篇 </font>](http://blog.yuanxiaolong.cn/blog/2014/10/22/docker-with-nodejs/)blog的 nodejs-helloworld

```bash
[root@yuanxiaolong nginx]# docker run -d -e VIRTUAL_HOST=www.yuanxiaolong.cn yuanxiaolong/centos-node-hello
496861d3a50d0655acb8ac544e732f06d000e47e88877e11b67877f6849cb3f0

[root@yuanxiaolong nginx]# docker run -d -e VIRTUAL_HOST=www.yuanxiaolong.cn yuanxiaolong/centos-node-hello
64e466932bdd459d6b2db707d118d8c188d2a5dab591483df5d73d59e1f67a76
```

Step 4.查看运行中的容器，是否是刚才我们启动的 1个nginx，2个nodejs

```bash
[root@yuanxiaolong nginx]# docker ps

CONTAINER ID        IMAGE                                   COMMAND              CREATED              STATUS              PORTS                NAMES
64e466932bdd        yuanxiaolong/centos-node-hello:latest   node /src/index.js   About a minute ago   Up About a minute   8088/tcp             prickly_shockley
496861d3a50d        yuanxiaolong/centos-node-hello:latest   node /src/index.js   11 minutes ago       Up 11 minutes       8088/tcp             backstabbing_nobel  
f7007b2bb4cb        jwilder/nginx-proxy:latest              forego start -r      12 minutes ago       Up 12 minutes       0.0.0.0:80->80/tcp   determined_nobel

```

Step 5.在浏览器 输入刚才 <font color="#b834a9"> VIRTUAL_HOST </font>设置的，你的域名，就可以查看到服务已经正常启动了。这时候用<font color="green"> docker logs </font> 看一下ngnix，看一下日志。

``` bash

[root@yuanxiaolong nginx]# docker logs f7
部分日志如下
-------第一部分，刚启动nginx-proxy，未代理任何应用时，只有自己一个容器-------
forego     | starting nginx.1 on port 5000
forego     | starting dockergen.1 on port 5100
dockergen.1 | 2014/10/23 02:46:18 Generated '/etc/nginx/sites-enabled/default' from 1 containers （这句话）
dockergen.1 | 2014/10/23 02:46:18 Running 'nginx -s reload'
dockergen.1 | 2014/10/23 02:46:18 Watching docker events
nginx.1    | 61.172.240.228 - - [23/Oct/2014:02:46:28 +0000] "GET / HTTP/1.1" 503 614 "-" "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/38.0.2125.104 Safari/537.36"

-------第二部分，刚启动第一个应用时，有2个容器-------
dockergen.1 | 2014/10/23 02:47:10 Received event start for container 496861d3a50d        （这句话）
dockergen.1 | 2014/10/23 02:47:10 Generated '/etc/nginx/sites-enabled/default' from 2 containers
dockergen.1 | 2014/10/23 02:47:10 Running 'nginx -s reload'
nginx.1    | 61.172.240.228 - - [23/Oct/2014:02:47:14 +0000] "GET / HTTP/1.1" 200 12 "-" "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/38.0.2125.104 Safari/537.36"

-------第三部分，刚启动第二个应用时，有3个容器-------
dockergen.1 | 2014/10/23 02:57:40 Received event start for container 64e466932bdd         （还有这句）
dockergen.1 | 2014/10/23 02:57:40 Generated '/etc/nginx/sites-enabled/default' from 3 containers
dockergen.1 | 2014/10/23 02:57:40 Running 'nginx -s reload'
nginx.1    | 180.153.81.158 - - [23/Oct/2014:02:57:48 +0000] "GET / HTTP/1.1" 200 12 "-" "DNSPod-Monitor/2.0"
nginx.1    | 61.172.240.228 - - [23/Oct/2014:02:57:54 +0000] "GET / HTTP/1.1" 200 12 "-" "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/38.0.2125.104 Safari/537.36"

```  

Step 6.进入nginx容器，看一下反向代理的配置。（利用[<font color="#2d58bd">上一篇</font>](http://blog.yuanxiaolong.cn/blog/2014/10/23/how-to-enter-a-running-docker-container/)的nsenter方法）

``` bash
[root@yuanxiaolong tmp]# sh docker-enter f7007b2bb4cb

root@f7007b2bb4cb:/etc/nginx# ls -al
total 60
drwxr-xr-x  5 root root 4096 Oct 22 23:53 .
drwxr-xr-x 68 root root 4096 Oct 23 02:46 ..
drwxr-xr-x  2 root root 4096 Sep 26 18:34 conf.d
-rw-r--r--  1 root root 1034 Sep 26 08:06 fastcgi.conf
-rw-r--r--  1 root root  964 Sep 26 08:06 fastcgi_params
-rw-r--r--  1 root root 2837 Sep 26 08:06 koi-utf
-rw-r--r--  1 root root 2223 Sep 26 08:06 koi-win
-rw-r--r--  1 root root 3957 Sep 26 08:06 mime.types
-rw-r--r--  1 root root 1344 Oct 22 23:53 nginx.conf
-rw-r--r--  1 root root  180 Sep 26 08:06 proxy_params
-rw-r--r--  1 root root  596 Sep 26 08:06 scgi_params
drwxr-xr-x  2 root root 4096 Oct 22 23:53 sites-available
drwxr-xr-x  2 root root 4096 Oct 23 02:57 sites-enabled    #注意这里的时间，跟日志里的吻合，应该就是这个文件夹
-rw-r--r--  1 root root  623 Sep 26 08:06 uwsgi_params
-rw-r--r--  1 root root 3071 Sep 26 08:06 win-utf



root@f7007b2bb4cb:/etc/nginx/sites-enabled# cat default

map $http_x_forwarded_proto $proxy_x_forwarded_proto {
  default $http_x_forwarded_proto;
  ''      $scheme;
}

server {
        listen 80 default_server;
        server_name _; # This is just an invalid value which will never trigger on a real hostname.
        error_log /proc/self/fd/2;
        access_log /proc/self/fd/1;
        return 503;
}

upstream www.yuanxiaolong.cn {

    # prickly_shockley         （这里是那2个容器的NAME，可以通过docker ps 看到）
    server 172.17.0.33:8088;

    # backstabbing_nobel
    server 172.17.0.32:8088;

}

server {
        gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

        server_name www.yuanxiaolong.cn;
        proxy_buffering off;
        error_log /proc/self/fd/2;
        access_log /proc/self/fd/1;

        location / {
                proxy_pass http://www.yuanxiaolong.cn;
                proxy_set_header Host $http_host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $proxy_x_forwarded_proto;

                # HTTP 1.1 support
                proxy_http_version 1.1;
                proxy_set_header Connection "";
        }
}


```

我们看到，负载均衡、反向代理已经配置好了。

Step 7.再用 <font color="green">docker inspect </font>看一下运行中的nodejs容器，是不是这2个IP

``` bash

[root@yuanxiaolong tmp]# docker inspect 64
[{
    "Args": [
        "/src/index.js"
    ],
   ……
   "Env": [
            "VIRTUAL_HOST=www.yuanxiaolong.cn",
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ],
   "ExposedPorts": {
            "8088/tcp": {}
        },
    ……
   "NetworkSettings": {
        "Bridge": "docker0",
        "Gateway": "172.17.42.1",
        "IPAddress": "172.17.0.33",
        "IPPrefixLen": 16,
        "PortMapping": null,
        "Ports": {
            "8088/tcp": null
        }
    },
   ……
}]


[root@yuanxiaolong tmp]# docker inspect 49
[{
    "Args": [
        "/src/index.js"
    ],
    ……
    "Env": [
            "VIRTUAL_HOST=www.yuanxiaolong.cn",
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ],
     "ExposedPorts": {
            "8088/tcp": {}
      },
     ……
     "NetworkSettings": {
        "Bridge": "docker0",
        "Gateway": "172.17.42.1",
        "IPAddress": "172.17.0.32",
        "IPPrefixLen": 16,
        "PortMapping": null,
        "Ports": {
            "8088/tcp": null
        }
    },
    ……
}]
```

果然，IPAddress 可以对的上。
