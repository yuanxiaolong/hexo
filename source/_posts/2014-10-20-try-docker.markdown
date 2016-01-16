---
layout: post
title: "try docker"
date: 2014-10-20 22:27:08 +0800
comments: true
categories: docker
tags: docker
share: true
description: docker初体验
toc: true
---
docker 初体验

<!--more-->
感受一下docker，介绍一下如何从无到有的利用docker来，建立自己的镜像大军

---

## 简介

docker是2014年最值得关注的一个东西，跟当年的github一样，呵呵，废话少说。Go ~

docker跟vmware类似，但docker更轻量，感性认识一下

![](/images/docker/20141020/1.png)

引用CSDN中有人回答如下

<font size="2" color="#7f7079">“docker做到了PAAS即平台即服务，docker在64位linux上使用的是lxc内核虚拟化也就是轻量级的虚拟化，与VM相比不需要对硬件进行仿真就可以共享跟主机一样的操作系统，并且有AUFS和lXC来虚拟化，加入一个ubuntu的镜像是265MB，你要再VM主机新建1000个就需要265000MB内存，但是docker共享容量也就需需要256多一点，如果你在linux上跑VMware相信你会看主机内存的消耗是比较大的，一个亚马逊EC2 512MB内存单核的云主机开5个docker无压力，你要是跑5个vmware那可费劲了”</font>

---

## 通俗理解

docker其实就是运行在 linux 下的一个进程，这个进程可以管理很多个“容器”。

如果非要跟java类比一下，这样可能更容易理解

* docker相当于jvm
* docker“镜像”，相当于一组java文件
* 而运行在docker上的“容器”，相当于在 jvm 下运行的 文件组
* 托管镜像的地方，就是docker服务器仓库。类似github托管代码一样。

现在有那么点儿意思了吧！来看个实例吧

---

## 从无到有

1.先在 64 位linux系统下（以centos为例），安装docker前置依赖。

```bash
yum install epel-release
```

2.安装docker

```bash
yum install docker-io
```


3.启动docker服务

```bash
sudo service docker start
```

4.查看状态或版本 docker version 或 docker info

```bash
[root@yuanxiaolong ~]# docker version

Client version: 1.1.2
Client API version: 1.13
Go version (client): go1.2.2
Git commit (client): d84a070/1.1.2
Server version: 1.1.2
Server API version: 1.13
Go version (server): go1.2.2
Git commit (server): d84a070/1.1.2
```

5.获取centos 6的镜像 docker pull centos:centos6

```bash
[root@yuanxiaolong ~]# docker pull centos:centos6

Pulling repository centos
68edf809afe7: Download complete
511136ea3c5a: Download complete
5b12ef8fd570: Download complete
```

6.查看本地镜像 docker images，此时会有一个centos的镜像 tag是centos6

```bash
root@yuanxiaolong ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
centos              centos6             68edf809afe7        2 weeks ago         212.7 MB
```


7.运行一下，看是否正常，docker run 68edf809afe7 /bin/echo hello world 或 docker run centos:centos6 /bin/echo hello world

```bash
[root@yuanxiaolong ~]# docker run 68edf809afe7 /bin/echo hello world
hello world

[root@yuanxiaolong ~]# docker run centos:centos6 /bin/echo hello world
hello world
```

8.再次确认，进入交互模式sudo docker run -i -t centos:centos6 /bin/bash

```bash
[root@yuanxiaolong ~]# docker run -i -t centos:centos6 /bin/bash
bash-4.1# pwd
/
bash-4.1# ls
bin  dev  etc  home  lib  lib64  lost+found  media  mnt  opt  proc  root  sbin  selinux  srv  sys  tmp  usr  var
bash-4.1# echo yes it is
yes it is
bash-4.1# exit
exit
[root@yuanxiaolong ~]#
```

9.（非必须）将想要保存的镜像，上传到 Docker Hub ，首先你需要有一个账户，在 [<font color="#118d6c">这里申请</font>](https://hub.docker.com/account/login/)


上传前，我的账户

![](/images/docker/20141020/2.png)

9.1先对要上传的images，打上个tag

```bash
[root@yuanxiaolong ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
centos              centos6             68edf809afe7        2 weeks ago         212.7 MB

[root@yuanxiaolong ~]# docker tag 68edf809afe7 yuanxiaolong/centos6

[root@yuanxiaolong ~]# docker images
REPOSITORY             TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
centos                 centos6             68edf809afe7        2 weeks ago         212.7 MB
yuanxiaolong/centos6   latest              68edf809afe7        2 weeks ago         212.7 MB
```

9.2上传镜像,输入用户名、密码、邮箱，即到你的账户里

```bash
[root@yuanxiaolong ~]# docker push yuanxiaolong/centos6
The push refers to a repository [yuanxiaolong/centos6] (len: 1)
Sending image list

Please login prior to push:
Username: yuanxiaolong
Password:
Email: 232351936@qq.com
Login Succeeded
The push refers to a repository [yuanxiaolong/centos6] (len: 1)
Sending image list
Pushing repository yuanxiaolong/centos6 (1 tags)
511136ea3c5a: Image already pushed, skipping
5b12ef8fd570: Image already pushed, skipping
68edf809afe7: Image already pushed, skipping
Pushing tag for rev [68edf809afe7] on {https://cdn-registry-1.docker.io/v1/repositories/yuanxiaolong/centos6/tags/latest}
```

上传后
![](/images/docker/20141020/3.png)

---

## 构建自己的环境

重复pull自己想要的镜像，可以在  [<font color="#118d6c">这里搜索</font>](https://registry.hub.docker.com/)

然后 <font color="#1e5d75">docker pull 名称:版本tag </font>，获取到本地，下面就是我个人在VPS上安装的镜像

```bash
[root@yuanxiaolong ~]# docker images
REPOSITORY             TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
mongo                  2.6                 97d8b6dd3b57        4 days ago          391.4 MB
node                   latest              eb23c5dc3891        9 days ago          728.9 MB
redis                  latest              cd8d56009b1f        10 days ago         110.8 MB
centos                 centos6             68edf809afe7        2 weeks ago         212.7 MB
yuanxiaolong/centos6   latest              68edf809afe7        2 weeks ago         212.7 MB
nginx                  1.7.5               d2d79aebd368        3 weeks ago         100.2 MB
```
---

## FAQ

Q: <font color="#c46a1d">如果pull错了版本的镜像怎么办？</font>

A: 不用担心，可以用删除镜像 命令

docker rmi centos:centos6 (如果镜像正在运行，则需要-f 来强制删除)

Q: <font color="#c46a1d">每个在docker上运行的容器，都有自己独立的IP么？</font>

A: 看起来是的，我进入2个不同的镜像，看到ip是不一样的，一个是172.17.0.9，一个是172.17.0.10

```bash
[root@yuanxiaolong ~]# docker run -i -t centos:centos6 /bin/bash
bash-4.1# ip -4 -o addr show eth0
26: eth0    inet 172.17.0.9/16 scope global eth0
bash-4.1# exit
exit
[root@yuanxiaolong ~]# docker run -i -t nginx:1.7.5 /bin/bash
root@de1467756eaf:/# ip -4 -o addr show eth0
29: eth0    inet 172.17.0.10/16 scope global eth0
root@de1467756eaf:/# exit
exit
```

* <font color="#c46a1d"> 更多内容</font>

[<font color="#340ba7">csdn docker资料大全</font>](http://special.csdncms.csdn.net/BeDocker/)
[<font color="#340ba7">docker 的10个小点</font>](http://docker.u.qiniudn.com/15_Docker_Tips_in_5_Minutes.pdf)
