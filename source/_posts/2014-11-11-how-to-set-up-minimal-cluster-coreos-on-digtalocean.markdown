---
layout: post
title: "how to set up minimal cluster coreos on digtalocean"
date: 2014-11-11 22:58:38 +0800
comments: true
categories: coreos
tags: CoreOS
share: true
description: 如何在digtalOcean 搭建CoreOS 最小集群
toc: true
---

如何在 digtalocean 搭建一个最小的，coreos 集群（3台主机）。

<!--more-->

digtalocean 是一个云主机提供商，我用了1年多了，感觉还是不错的。

打个广告，弹性配置，最低端配置，只要30元1个月，平均1天1元。现在 [<font color="#22832f"> 点击这里 </font>](https://www.digitalocean.com/?refcode=78e09919d062) 还可以获取60元，即免费使用2个月。

---

## 配置过程


官网教程 : [<font color="#a95bae"> 点击这里 </font>](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-coreos-cluster-on-digitalocean)


### 申请token
Step 1.首先打开浏览器，输入 <code>https://discovery.etcd.io/new </code>，后你会获得一串英文。


这串英文叫 token，一定！一定！要保存好，它是一个世界全局唯一的一个 集群id ，要搭建集群就靠它了。

![](/images/coreos/20141110/5.png)



### 保存模板
Step 2. 保存一个模板文件，这个模板文件，用来初始化CoreOS的安装。注意，这个文件一定要高度一致，即3个主机所用的文件是一模一样的，我就是由于文件不一致，导致1台主机没加入到集群中，又重新删掉掉队的主机，又重新生成了一次……


``` bash
#cloud-config


coreos:
  etcd:
    # generate a new token for each unique cluster from https://discovery.etcd.io/new
    discovery: https://discovery.etcd.io/你的集群id串
    # multi-region deployments, multi-cloud deployments, and droplets without
    # private networking need to use $public_ipv4
    addr: $private_ipv4:4001
    peer-addr: $private_ipv4:7001
  fleet:
    public-ip: $private_ipv4   # used for fleetctl ssh command
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
```

### 选择系统
Step 3. 操作系统选择“CoreOS”，并把上面文件的内容，在 <font color="#1e68ad">"Settings"</font>的<font color="#1e68ad">"Enable User Data"</font>选项卡中黏贴。

### 设置ssh
Step 4. 由于CoreOS是需要初始化用 ssh key来登录的，因此，你需要先在digtalocean里，添加自己的ssh-key。然后此处勾选上
你希望使用的ssh-key。当CoreOS安装完成后，它会把你这里勾选的ssh-key，放到 <code>~/.ssh/authorized_keys</code>
里。

### 部署
Step 5.点击完成，等30秒，CoreOS就安装完毕了，反复把3台主机都建好。
![](/images/coreos/20141110/6.png)

### 登录
Step 6.利用SercurtCRT，登录刚才初始化好的CoreOS。SercurtCRT的使用就不说了，说一下跟平时登录其他linux不同的地方。
CoreOS是不支持用户名密码登录的（起码我没有发现...），是需要用ssh-key登录，因此需要在 SercurtCRT 的 “session options" 配置里，

将"publicKey"放到首位，而不是 "Password"。
![](/images/coreos/20141110/7.png)

同时，要 Edit，选择自己的 public key
![](/images/coreos/20141110/8.png)


PS：怎么验证集群已经搭建好了呢？下一篇blog会讲，CoreOS的2大组件，etcd和fleet，到时候就知道了。

---

## 小技巧

1.由于CoreOS是只读的ROOT，默认的 .bash_profile 和 .bashrc 都是 ln 到 root上的，因此需要在~下执行

```
cp $(readlink .bashrc) .bashrc.new && mv .bashrc.new .bashrc
```

然后编辑 .bashrc

```
alias ll='ls -alF'

```


2.CoreOS的 nsenter 跟CentOS 的脚本不一样，所以要进入docker 容器，需要用下面的脚本。在Docker 1.3中有 exec功能，可以直接进入容器，简单方便。


``` bash
#!/bin/sh

if [ -e $(dirname "$0")/nsenter ]; then
    # with boot2docker, nsenter is not in the PATH but it is in the same folder
    NSENTER=$(dirname "$0")/nsenter
else
    NSENTER=nsenter
fi

if [ -z "$1" ]; then
    echo "Usage: docker-enter CONTAINER [COMMAND [ARG]...]"
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
        sudo "$NSENTER" $OPTS su - root
    else
        # Use env to clear all host environment variables.
        sudo "$NSENTER" $OPTS env --ignore-environment -- "$@"
    fi
fi
```
