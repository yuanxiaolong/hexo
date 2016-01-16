---
layout: post
title: "install apache james"
date: 2014-07-20 22:34:30 +0800
comments: true
category: ['apache james']
tags: 'apache james'
share: true
description: 安装邮件服务器apache james
toc: true
---
介绍一下利用apache james来安装一个邮件服务器，发送邮件。

<!--more-->

去年搭建了一个，把过程整理一下，分享给大家(由于文章较长，因此跟本文无关的都精简掉)

---

## 准备工作

1.  由于Apache James邮件服务需要用到1024以下的端口，因此请用root用户登录进行部署
2.  需要先安装JDK1.5或以上版本，部署前请确保您的JDK环境变量如JAVA_HOME等已经设置好
3.  James 启动时，其SMTP 服务默认在 25 端口启动，POP3 服务默认在 110 端口启动， NNTP 服务默认在 119 端口启动, 请确保这些端口未被占用。

  我们可以通过下面命令来查看端口

``` bash
lsof -i:25
```

如果出现了listen状态的，说明这些端口被占用了。例如，我本地开启4000端口服务，如果4000端口被占用，则显示如下：

``` bash
xiaolongyuan@xiaolongdeMacBook-Air ~$ lsof -i:4000
COMMAND    PID    USER           FD   TYPE     DEVICE                 SIZE/OFF NODE  NAME
ruby       7828   xiaolongyuan   9u   IPv4     0x51aae4d89be38d57     0t0      TCP   *:terabase (LISTEN)
```

这时就要根据自己的情况，到底kill掉进程还是怎么做。一般可能会遇到sendmail服务占用25端口的情况，
关闭sendmail服务，百度有很多，由于篇幅原因，请自行查找。

---

## 正式部署

1.下载 [<font color="#1f7698">apache-james-2.3.2.tar.gz</font>](http://pan.baidu.com/s/1h57fQ)

2.解压

``` bash
tar zxvf apache-james-2.3.2.tar.gz
```

3.进入james-2.3.2/bin目录，添加执行权限并运行run.sh，生成james的配置文件config.xml

``` bash
chmod +x run.sh phoenix.sh
sh ./run.sh

Using PHOENIX_HOME:   /usr/local/james-2.3.2
Using PHOENIX_TMPDIR: /usr/local/james-2.3.2/temp
Using JAVA_HOME:      /usr/java/jdk1.5.0
Running Phoenix:
Phoenix 4.2
James Mail Server 2.3.2
Remote Manager Service started plain:4555
POP3 Service started plain:110
SMTP Service started plain:25
NNTP Service started plain:119
FetchMail Disabled
```

说明james启动成功，并占用3个端口

---

## 自定义信息

1.按Ctrl + C退出James，编辑config.xml文件( *<font color="#6a117b">/james-2.3.2/apps/james/SAR-INF/config.xml </font>* )

<font color="green" size="4">主要的一些点：</font>

 修改成下面的配置 <font color="#76747a">(示例:发件人地址abc.com)</font>

``` xml

<config>
    <James>
      <postmaster>Postmaster@abc.com</postmaster>
      <servernames autodetect="true" autodetectIP="true">
        <servername>abc.com</servername>
      </servernames>
    <James>
<config>

```

  同时将/etc/hosts里面添加 abc.com 的ip
  <font color="#76747a">( autodetct设为true会自动侦测你的主机名,设成false会用你指定的server name</font>

 inbox的存储位置，修改成下面配置，并注释掉一段（用叫maildb的数据源）

``` xml
<inboxRepository>
  <repository destinationURL="db://maildb/inbox/" type="MAIL"/>
</inboxRepository>
……
<!-- <mailet match="RemoteAddrNotInNetwork=127.0.0.1" class="ToProcessor">
            <processor> relay-denied </processor>
            <notice>550 - Requested action not taken: relaying denied</notice>
</mailet> -->
……
<outgoing> db://maildb/spool/outgoing </outgoing>
……
```

 同理修改spool的配置也如上

 修改dnsserver，由于邮件服务器发送邮件的时候肯定是往外网发，所以需要dns来做发送时，投递给对面接受服务器域名时的解析工作

  1.先查找dns服务器

``` bash
xiaolongyuan@xiaolongdeMacBook-Air ~$ cat /etc/resolv.conf
nameserver 10.203.102.215
nameserver 10.203.101.215
```

  2.修改james的dns配置

``` xml

  <dnsserver>
    <servers>
      <server>10.203.102.215</server>
      <server>10.203.101.215</server>
    </servers>
  </dnsserver>
  ……
  <autodiscover>false</autodiscover>
  <authoritative>false</authoritative>
  ……
  <smtpserver enabled="true">
    <handler>
      <helloName autodetect="false">myMailServer</helloName>
      ……
      <authRequired>false</authRequired>
      ……
      <authorizedAddresses>*</authorizedAddresses>
      ……
    </handler>
  </smtpserver>  

```

 修改存储，修改为jdbc模式

``` xml
<users-store>
  <!--<repository name="LocalUsers" class="org.apache.james.userrepository.UsersFileRepository">
       <destination URL="file://var/users/"/>
    </repository>-->
  ……
  <repository name="LocalUsers" class="org.apache.james.userrepository.JamesUsersJdbcRepository" destinationURL="db://maildb/users">
    <sqlFile>file://conf/sqlResources.xml</sqlFile>
  </repository>
  ……
</users-store>
```

 新建jdbc数据源(先在mysql中新建一个数据库，这里是james_mail。同时为james新建个用户密码)

  同时连接mysql的驱动 ([<font color="#3756b2">这里下载</font>](http://pan.baidu.com/s/1dDnaw05) 并把它放到james-2.3.2的lib目录下)

``` xml
<data-source name="maildb" class="org.apache.james.util.dbcp.JdbcDataSource" >
  <driver>com.mysql.jdbc.Driver</driver>
  <dburl>jdbc:mysql://DB服务器ip:3306/james_mail?autoReconnect=true</dburl>
  <user>james</user>
  <password>james</password>
  <max>20</max>
</data-source>
```

 账号管理，help查看使用（默认的登陆id 为root 密码也为 root 可以在config.xml中修改）

``` bash
telnet 邮件服务器ip 4555
help
```

 再次启动james就OK了。
