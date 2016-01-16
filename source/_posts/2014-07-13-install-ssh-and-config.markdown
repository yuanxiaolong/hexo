---
layout: post
title: "install ssh and config"
date: 2014-07-13 17:38:10 +0800
comments: true
category: ssh
tags: ssh
share: true
description: 配置linux ssh互信
toc: true
---
安装及配置ssh,是hadoop启动的前置条件
<!--more-->

---

## ssh是什么？
我们不是百度百科的搬运工，拿目前看来SSH是用来解决多台机器之间的互信问题。

例如有2台机器，它们之间可以通过 **口令/密码** 来登录。那么久需要维护密码这样的东西，如果hadoop集群有1000台，而且出于安全考虑3个月换一次随机密码。那么这将是无法估量的维护成本。


这样ssh的 **公钥/私钥** 就显的十分灵巧了。

---

## ssh之间的过程
这里是 [阮一峰的blog](http://www.ruanyifeng.com/blog/2011/12/ssh_remote_login.html)里面有介绍ssh，文章不长，还可以。

---

## ssh配置
1.先看自己是否已经有ssh，如果用下面命令发现有sshd的进程，那么就可以略过，步骤2安装。

``` bash
ps -ef | grep sshd
```

2.安装sshd(拿ubuntu举例)

``` bash
sudo apt-get install openssh-server
```

3.在每台需要跳转，及被跳转的机器上执行下面命令，一路回车（如果要安装hadoop，则每台机器需要建立相同的用户名）

``` bash
ssh-keygen -t rsa
```
这时，在 ~/.ssh目录下有2个文件 其中id_rsa为私钥，id_rsa.pub为公钥。

4.添加信任名单。较为简单稳妥的方式是这样的。

假设3台机器 hadoop1、hadoop2、hadoop3 。我们先在hadoop1上，执行

``` bash
cp id_rsa.pub authorized_keys
chmod 640 authorized_keys
```

再利用 *<font color="green">scp ip:远端路径  本地路径 </font>* 将hadoop2、hadoop3上的公钥文件（.pub文件）拷贝到hadoop1上。（注意：不要重名）

这样hadoop2的公钥文件为2.pub，hadoop3的为3.pub  再将文件，追加合并

``` bash
cat 2.pub >> authorized_keys
cat 3.pub >> authorized_keys
```

再用scp命令 *<font color="green">scp 本地路径 ip:远端路径</font>* ，将名单分发到hadoop2和hadoop3上

5.在/etc/hosts下，分别加上3台机器的密码。再手工试试 ssh remotelocalhost 看看是否成功。
