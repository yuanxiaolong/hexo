---
layout: post
title: "deploy shadowsocks on VPS"
date: 2015-04-28 21:27:15 +0800
comments: true
categories: ['vps']
tags: vps
share: true
description: 利用vps搭建属于自己的通道
toc: true
---
利用自己的VPS，搭建一个代理通道，以便访问google等资源

<!--more-->

好久没写blog了，最近非工作事情比较忙，才耽搁了一阵，但是保证都是干货。

---

## 环境准备

VPS一台，我用的是 digtal Ocean 的，2015年提供了新加坡的机房，可以买在新加坡。比纽约和旧金山要快一倍访问速度，大约 ping 200ms

CentOS 6.x

``` bash
yum install git openssl-devel -y
yum install build-essential autoconf libtool gcc -y

```

---

## 原理介绍

![](/images/vps/20150428/1.png)

如果浏览器直接访问 google 则由于墙的存在，导致不通。因此需要用代理做请求的转发。

Shadowsocks 是一种安全的 socks5 代理，可以保护你的上网流量。基于多种加密方式，推荐使用 aes-256-cfb 加密。安装和使用需要本地端和服务端

shadowsocks 就是这样的一个代理，在海外vps上装Server，在本机上装Client就可以了


## Server端

将下面脚本，在 `CentOS 6.x ` 上运行

``` bash shadowsocks-libev.sh
#! /bin/bash
PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/bin
export PATH
#===============================================================================================
#   System Required:  CentOS6.x (32bit/64bit)
#   Description:  Install Shadowsocks(libev) for CentOS
#   Author: Teddysun <i@teddysun.com>
#   Intro:  http://teddysun.com/357.html
#===============================================================================================

clear
echo "#############################################################"
echo "# Install Shadowsocks(libev) for CentOS 6 or 7 (32bit/64bit)"
echo "# Intro: http://teddysun.com/357.html"
echo "#"
echo "# Author: Teddysun <i@teddysun.com>"
echo "#"
echo "#############################################################"
echo ""

# Make sure only root can run our script
function rootness(){
if [[ $EUID -ne 0 ]]; then
   echo "Error:This script must be run as root!" 1>&2
   exit 1
fi
}

# Get version
function getversion(){
    if [[ -s /etc/redhat-release ]];then
        grep -oE  "[0-9.]+" /etc/redhat-release
    else
        grep -oE  "[0-9.]+" /etc/issue
    fi
}

# CentOS version
function centosversion(){
    local code=$1
    local version="`getversion`"
    local main_ver=${version%%.*}
    if [ $main_ver == $code ];then
        return 0
    else
        return 1
    fi
}

# Disable selinux
function disable_selinux(){
if [ -s /etc/selinux/config ] && grep 'SELINUX=enforcing' /etc/selinux/config; then
    sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
    setenforce 0
fi
}

# Pre-installation settings
function pre_install(){
    # Not support CentOS 5
    if centosversion 5; then
        echo "Not support CentOS 5, please change to CentOS 6 or 7 and try again."
        exit 1
    fi
    #Set shadowsocks-libev config password
    echo "Please input password for shadowsocks-libev:"
    read -p "(Default password: teddysun.com):" shadowsockspwd
    if [ "$shadowsockspwd" = "" ]; then
        shadowsockspwd="teddysun.com"
    fi
    echo "password:$shadowsockspwd"
    echo "####################################"
    get_char(){
        SAVEDSTTY=`stty -g`
        stty -echo
        stty cbreak
        dd if=/dev/tty bs=1 count=1 2> /dev/null
        stty -raw
        stty echo
        stty $SAVEDSTTY
    }
    echo ""
    echo "Press any key to start...or Press Ctrl+C to cancel"
    char=`get_char`
    #Install necessary dependencies
    yum install -y wget unzip openssl-devel gcc swig python python-devel python-setuptools autoconf libtool libevent
    yum install -y automake make curl curl-devel zlib-devel openssl-devel perl perl-devel cpio expat-devel gettext-devel
    # Get IP address
    echo "Getting Public IP address, Please wait a moment..."
    IP=$(curl -s -4 icanhazip.com)
    if [[ "$IP" = "" ]]; then
        IP=`curl -s -4 ipinfo.io | grep "ip" | awk -F\" '{print $4}'`
    fi
    echo -e "Your main public IP is\t\033[32m$IP\033[0m"
    echo ""
    #Current folder
    cur_dir=`pwd`
    cd $cur_dir
}

# Download latest shadowsocks-libev
function download_files(){
    if [ -f shadowsocks-libev.zip ];then
        echo "shadowsocks-libev.zip [found]"
    else
        if ! wget --no-check-certificate https://github.com/shadowsocks/shadowsocks-libev/archive/master.zip -O shadowsocks-libev.zip;then
            echo "Failed to download shadowsocks-libev.zip"
            exit 1
        fi
    fi
    unzip shadowsocks-libev.zip
    if [ $? -eq 0 ];then
        cd $cur_dir/shadowsocks-libev-master/
    else
        echo ""
        echo "Unzip shadowsocks-libev failed! Please visit http://teddysun.com/357.html and contact."
        exit 1
    fi
    # Download start script
    if ! wget --no-check-certificate https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks-libev; then
        echo "Failed to download shadowsocks-libev start script!"
        exit 1
    fi
}

# Config shadowsocks
function config_shadowsocks(){
    if [ ! -d /etc/shadowsocks-libev ];then
        mkdir /etc/shadowsocks-libev
    fi
    cat > /etc/shadowsocks-libev/config.json<<-EOF
{
    "server":"0.0.0.0",
    "server_port":8989,
    "local_address":"127.0.0.1",
    "local_port":1080,
    "password":"${shadowsockspwd}",
    "timeout":600,
    "method":"aes-256-cfb"
}
EOF
}

# iptables set
function iptables_set(){
    echo "iptables start setting..."
    /sbin/service iptables status 1>/dev/null 2>&1
    if [ $? -eq 0 ]; then
        /etc/init.d/iptables status | grep '8989' | grep 'ACCEPT' >/dev/null 2>&1
        if [ $? -ne 0 ]; then
            /sbin/iptables -I INPUT -m state --state NEW -m tcp -p tcp --dport 8989 -j ACCEPT
            /etc/init.d/iptables save
            /etc/init.d/iptables restart
        else
            echo "port 8989 has been set up."
        fi
    else
        echo "iptables looks like shutdown, please manually set it if necessary."
    fi
}

# Install
function install(){
    # Build and Install shadowsocks-libev
    if [ -s /usr/local/bin/ss-server ];then
        echo "shadowsocks-libev has been installed!"
        exit 0
    else
        ./configure
        make && make install
        if [ $? -eq 0 ]; then
            mv $cur_dir/shadowsocks-libev-master/shadowsocks-libev /etc/init.d/shadowsocks
            chmod +x /etc/init.d/shadowsocks
            # Add run on system start up
            chkconfig --add shadowsocks
            chkconfig shadowsocks on
            # Start shadowsocks
            /etc/init.d/shadowsocks start
            if [ $? -eq 0 ]; then
                echo "Shadowsocks-libev start success!"
            else
                echo "Shadowsocks-libev start failure!"
            fi
        else
            echo ""
            echo "Shadowsocks-libev install failed! Please visit http://teddysun.com/357.html and contact."
            exit 1
        fi
    fi
    cd $cur_dir
    # Delete shadowsocks-libev floder
    rm -rf $cur_dir/shadowsocks-libev-master/
    # Delete shadowsocks-libev zip file
    rm -f shadowsocks-libev.zip
    clear
    echo ""
    echo "Congratulations, shadowsocks-libev install completed!"
    echo -e "Your Server IP: \033[41;37m ${IP} \033[0m"
    echo -e "Your Server Port: \033[41;37m 8989 \033[0m"
    echo -e "Your Password: \033[41;37m ${shadowsockspwd} \033[0m"
    echo -e "Your Local IP: \033[41;37m 127.0.0.1 \033[0m"
    echo -e "Your Local Port: \033[41;37m 1080 \033[0m"
    echo -e "Your Encryption Method: \033[41;37m aes-256-cfb \033[0m"
    echo ""
    echo "Welcome to visit:http://teddysun.com/357.html"
    echo "Enjoy it!"
    echo ""
    exit 0
}

# Uninstall Shadowsocks-libev
function uninstall_shadowsocks_libev(){
    printf "Are you sure uninstall shadowsocks_libev? (y/n) "
    printf "\n"
    read -p "(Default: n):" answer
    if [ -z $answer ]; then
        answer="n"
    fi
    if [ "$answer" = "y" ]; then
        ps -ef | grep -v grep | grep -v ps | grep -i "ss-server" > /dev/null 2>&1
        if [ $? -eq 0 ]; then
            /etc/init.d/shadowsocks stop
        fi
        chkconfig --del shadowsocks
        # delete config file
        rm -rf /etc/shadowsocks-libev
        # delete shadowsocks
        rm -f /usr/local/bin/ss-local
        rm -f /usr/local/bin/ss-tunnel
        rm -f /usr/local/bin/ss-server
        rm -f /usr/local/bin/ss-redir
        rm -f /usr/local/lib/libshadowsocks.a
        rm -f /usr/local/lib/libshadowsocks.la
        rm -f /usr/local/include/shadowsocks.h
        rm -rf /usr/local/lib/pkgconfig
        rm -f /usr/local/share/man/man8/shadowsocks.8
        rm -f /etc/init.d/shadowsocks
        echo "Shadowsocks-libev uninstall success!"
    else
        echo "uninstall cancelled, Nothing to do"
    fi
}

# Install Shadowsocks-libev
function install_shadowsocks_libev(){
    rootness
    disable_selinux
    pre_install
    download_files
    config_shadowsocks
    if ! centosversion 7; then
        iptables_set
    fi
    install
}

# Initialization step
action=$1
[  -z $1 ] && action=install
case "$action" in
install)
    install_shadowsocks_libev
    ;;
uninstall)
    uninstall_shadowsocks_libev
    ;;
*)
    echo "Arguments error! [${action} ]"
    echo "Usage: `basename $0` {install|uninstall}"
    ;;
esac

```

然后执行

``` bash
chmod +x shadowsocks-libev.sh
./shadowsocks-libev.sh 2>&1 | tee shadowsocks-libev.log
```

中间提示输入自己的密码

``` bash
Congratulations, shadowsocks-libev install completed!
Your Server IP:your_server_ip
Your Server Port:8989
Your Password:<你的密码>
Your Local IP:127.0.0.1
Your Local Port:1080
Your Encryption Method:aes-256-cfb

Welcome to visit:http://teddysun.com/357.html
Enjoy it!

```

查看启动后的进程

``` bash
ps -ef | grep ss-server | grep -v ps | grep -v grep

```

卸载命令

``` bash
./shadowsocks-libev.sh uninstall
```

防火墙（如果有，则配置，没亲测，我关闭了防火墙）

``` bash
service iptables status   （查看状态）
service iptables stop     （停止）
service iptables start    （启动）
```

``` bash
-A INPUT -p tcp -m state --state NEW -m tcp --dport 8989 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited                (如果有这条，则需要加在这条的前面)
```

---

## Client 端

百度搜索关键词 `shadowsockx + 你的OS ` 根据自己的操作系统，下载不同的客户端

我的是mac的，共享百度云 [<font color="#22832f"> 点击这里 </font>](http://pan.baidu.com/s/1bnsBOIj)

配置客户端

![](/images/vps/20150428/2.png)

![](/images/vps/20150428/3.png)


至此，就有了一个通道 客户端 `默认1080端口` ，即请求经过1080代理后，转发到服务器上的8989，通过shadowsock代理

---

## 切换请求

在chrome里装一个插件，这个插件就可以当做开关，让请求是直接连接，还是用shadowsock代理

插件叫switchysharp，这里有篇安装教程
[<font color="#22832f"> http://switchysharp.com/install.html </font>](http://switchysharp.com/install.html)


配置switchysharp如下，其中 “不代理的地址” 即不想用shadowsock代理访问的地址，类似 `*.youku.com` ，因为看视频可能流量大，VPS每月是限制流量限额的，跟手机一样。

![](/images/vps/20150428/4.png)



参考资料：

* [<font color="#22832f"> http://teddysun.com/357.html </font>](http://teddysun.com/357.html)
* [<font color="#22832f"> http://switchysharp.com/install.html </font>](http://switchysharp.com/install.html)
