---
layout: post
title: "install dns server bind"
date: 2014-09-07 16:18:29 +0800
comments: true
categories: dns
tags: dns
share: true
description: 安装dns服务器,用于解析
toc: true
---

介绍如何安装dns 服务器用于解析

<!--more-->
即使没有一个有效的域名也可以配置dns服务器，解析自定义的域名。
这样在管理hadoop集群时，新增一个节点，就只需要在dns服务器的配置文件里，
新增一套配置记录，而不用在每个节点上hosts文件里添加新机器的映射，以便ssh跳转。

---

## 环境准备

linux 系列，我用的ubuntu，一般线上服务器用的是centos，不过不影响，只是下载dns server的方式不同。

---

## 安装配置

1.下载dns server 软件 bind

``` bash
sudo apt-get install bind9
```

2.修改dns server 的配置 <font color="#7c837f"> (其中 /etc/bind/zones/ 和 yxl.com 是自己选的) </font>

``` bash /etc/bind/named.conf.local
hadoop@hadoop4:/etc/bind$ cat named.conf.local

//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
zone "yxl.com"{
        type master;
        file "/etc/bind/zones/yxl.com.db";
};
zone "1.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/zones/1.168.192.db";
};
```

3.配置 上面的正向域名解析、下面的反向域名解析文件 <font color="#7c837f">(由于我本机这个网段是192.168.1的，因此反向文件照旧这样反正写)。</font>

``` bash /etc/bind/zones/yxl.com.db
hadoop@hadoop4:/etc/bind/zones$ cat yxl.com.db

@       IN      SOA     localhost. root.localhost. (
1         ; Serial
604800         ; Refresh
86400         ; Retry
2419200         ; Expire
604800 )       ; Negative Cache TTL
;
@       IN      NS      localhost.
@       IN      A       127.0.0.1

hadoop4 IN      A       192.168.1.111
hadoop5 IN      A       192.168.1.116
hadoop6 IN      A       192.168.1.117
hadoop7 IN      A       192.168.1.197

```

``` bash /etc/bind/zones/1.168.192.db
hadoop@hadoop4:/etc/bind/zones$ cat 1.168.192.db
N SOA yxl.com. root.yxl.com. (
1 ; Serial
604800 ; Refresh
86400 ; Retry
2419200 ; Expire
604800 ) ; Negative Cache TTL;

@       IN      NS      yxl.com.
111     IN      PTR     hadoop4.yxl.com
116     IN      PTR     hadoop5.yxl.com
117     IN      PTR     hadoop6.yxl.com
197     IN      PTR     hadoop7.yxl.com
```

4.重启，利用nslookup 测试刚才的正向、反向解析。

![](/images/dns/20140907/1.png)
