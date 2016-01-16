---
layout: post
title: "graphite and cabot"
date: 2015-12-08 17:38:57 +0800
comments: true
categories: graphite
tags: [graphite,cabot]
description: graphite 及 cabot 部署 及 配置
toc: true
---

介绍如何 部署及配置 启动 graphite 和 cabot

<!--more-->

graphite 是一个 指标数据收集系统，一般作为监控系统中 「监控数据收集存储」使用，它有3个组件 Carbon Whisper 和 Graphite-web 这3个组件都是 python pip 安装的，它们三个结合起来就是 graphite 了

cabot 是一个根据 graphite Whisper 里的 指标数据，进行规制监控，并报警的一个软件

与 zabbix 、ganglia 有何不同呢? 一般系统级 用 它们就足够了，如果是业务数据，可能就需要 graphite 和 cabot 了

---

## 环境准备

python 2.7 及以上（因为cabot需要）
centos 或其他系统主机 或许基础软件不一样


一般来说，或许有人喜欢用 virtualenv 但是，我不太熟悉 python 因此，直接用了 python 2.7

---

## 搭建 graphite

安装基础

``` bash
pip install whisper
pip install carbon
pip install graphite-web
pip install django==1.6.8
pip install django-tagging==0.3.6
pip install uWSGI==2.0.11.1
pip install MySQL-python==1.2.5
pip install daemonize
```

最后安装下来 python2.7.3 其中组件版本 whisper(0.9.15) daemonize(2.4.1)

引用
* graphite使用cairo进行绘图，由于系统自带的cairo版本较低（需要cairo1.10以上），使用pip安装cairo会出错，所以采用编译安装。


``` bash
wget http://cairographics.org/releases/pycairo-1.8.8.tar.gz
tar zxvf pycairo-1.8.8.tar.gz
python -c "import sys; print sys.prefix"
cd pycairo-1.8.8
./configure --prefix=/data/server/python-envs/graphite
make
make install
```

目录说明

``` bash
bin -- 数据收集相关工具
conf -- 数据存储相关配置文件
    carbon.conf -- 数据收集carbon进程涉及的配置
    dashboard.conf -- Dashboard UI相关配置
    graphite.wsgi -- wsgi相关配置
    storage-schemas.conf -- Schema definitions for Whisper files
    whitelist.conf -- 定义允许存储的metrics白名单
    graphTemplates.conf -- 图形化展示数据时使用的模板
examples -- 示例脚本
lib -- carbon和twisted库
storage -- 数据文件存储目录
webapp -- 数据前端展示涉及程序
```

配置 graphite-web

初始化配置文件

``` bash
cd /opt/graphite/webapp/graphite
cp local_settings.py.example local_settings.py
cp /opt/graphite/conf/graphite.wsgi.example /opt/graphite/conf/graphite.wsgi
cp /opt/graphite/conf/graphTemplates.conf.example /opt/graphite/conf/graphTemplates.conf
cp /opt/graphite/conf/dashboard.conf.example /opt/graphite/conf/dashboard.conf
```

修改 `local_settings.py` ，其中修改自己的环境

``` java
TIME_ZONE = 'Asia/Shanghai'
DATABASES = {
    'default': {
        'NAME': 'graphitedb', #自己定义名字
        'ENGINE': 'django.db.backends.mysql',
        'USER': 'graphite',
        'PASSWORD': 'graphite',
        'HOST': '192.168.7.11',
        'PORT': '3306'
    }
}
```

初始化数据库，期间会让你指定 用户名 和 密码

``` bash
python manage.py syncdb
```

启动 graphite-web

``` bash
nohup uwsgi --http 192.168.7.12:8050 --master --processes 1 --pythonpath /opt/graphite/webapp/graphite --wsgi-file=/opt/graphite/conf/graphite.wsgi --enable-threads --thunder-lock &
```

访问 `http://192.168.7.12:8050` 看是否有界面，如果没有界面 则首先看 nohup.out 日志是否报错，如果未报错，则估计上面的 `cairo` 绘图版本有问题，安装一下就好了

配置收集服务

``` bash
cp /opt/graphite/conf/carbon.conf.example /opt/graphite/conf/carbon.conf
cp /opt/graphite/conf/storage-schemas.conf.example /opt/graphite/conf/storage-schemas.conf
cp /opt/graphite/conf/whitelist.conf.example /opt/graphite/conf/whitelist.conf
```

其中 `/opt/graphite/conf/whitelist.conf` 是白名单，可以添加类似这样的正则，就让 graphite 只收集 test 和 server 的数据了

``` bash
^test\..*
^server\..*
```

其中 `/opt/graphite/conf/storage-schemas.conf` 是配置存储策略的文件，可以添加类似如下的规则

``` bash
[server]
pattern = ^server\..*
retentions = 60s:1d,5m:7d,15m:3y

[default]
pattern = ^test\..*
retentions = 60s:1d,5m:7d
```

上面的配置，会对于test.开头的metrics，以60秒为精度存储一天，以5分钟为精度存储7天。即查询一天内的数据时，可以精确到1分钟，查询7天内的数据时，只能精确到5分钟。

启动 ` nohup python /opt/graphite/bin/carbon-cache.py --config=/opt/graphite/conf/carbon.conf --debug start & `

引用
 * 造收集数据，利用 crontab 添加定时任务，往 2003 端口发送 指标数据

``` bash
#!/bin/sh

HOST=$(hostname | awk -F'.' '{print $1}')
IDC="local"

SYSTEM_LOAD=$(awk '{print $1}' /proc/loadavg)
SYSTEM_MEMORY_FREE=$(free -m | grep 'buffers/cache' | awk '{print $NF}')
SYSTEM_SWAP_USE=$(free -m | grep 'Swap' | awk '{print $(NF-1)}')
SYSTEM_DISK_USED=$(df -h | grep '/' | awk 'BEGIN{_max=0}{len=length($5);i=substr($5,0,len-1);if(_max<i){_max=i}}END{print _max}')

TIMESTAMP=$(date +%s)

### send to garphite through udp port 2003 ########
echo -n "server.$IDC.$HOST.system.load $SYSTEM_LOAD $TIMESTAMP" > /dev/udp/127.0.0.1/2003
echo -n "server.$IDC.$HOST.system.memory_free $SYSTEM_MEMORY_FREE $TIMESTAMP" > /dev/udp/127.0.0.1/2003
echo -n "server.$IDC.$HOST.system.swap_used $SYSTEM_SWAP_USED $TIMESTAMP" > /dev/udp/127.0.0.1/2003
echo -n "server.$IDC.$HOST.system.disk_used $SYSTEM_DISK_USED $TIMESTAMP" > /dev/udp/127.0.0.1/2003
```

---

## 搭建Cabot

* ruby 1.9.3 以上
* 安装 nodejs（内嵌了npm）
* 同时通过 npm 安装 coffee-script `npm install -g coffee-script less@1.3 --registry http://registry.npmjs.org/` 以便 UI 渲染
* redis

安装即 git clone 下 Cabot 源码，并修改配置文件

``` bash
git clone https://github.com/arachnys/cabot.git
cd cabot
cp conf/development.env.example conf/development.env
```

根据个人配置修改环境变量，及是否转成私网ip

``` bash development.env
DATABASE_URL=mysql://cabot:cabot@192.168.7.11:3306/cabot?characterEncoding=utf8
TIME_ZONE=Asia/Shanghai
ADMIN_EMAIL=your@qq.com
CABOT_FROM_EMAIL=your@126.com

# 这里指向了一个 reids实例
CELERY_BROKER_URL=redis://192.168.7.12:6379/1

# graphite
GRAPHITE_API=http://192.168.7.12:8050/
GRAPHITE_USER=admin
GRAPHITE_PASS=admin

# smtp
SES_HOST=smtp.126.com
SES_USER=yourname
SES_PASS=yourpassword
SES_PORT=465          # 用SSL 端口，因为python的组件需要

# 回调查看报警url
WWW_HTTP_HOST=test.cabot.yourdomain.com

```

端口绑定私网地址

``` bash Procfile.dev
web:       python manage.py runserver 192.168.7.12:8051
celery:    celery -A cabot worker --loglevel=DEBUG -B -c 8 -Ofair
```



修改 `setup.py` 添加依赖

``` bash
'MySQL-python==1.2.5',
```

后安装依赖，并初始化数据库，在初始化时，会让你指定一个用户名和密码

``` bash
python setup.py install
/bin/sh ./setup_dev.sh
```

一系列完成后，则你可以在数据库看到很多张表，如果中间报错，多半是因为环境问题，缺少python 包，可以通过 pip 安装
在官网 [<font color="#2798a2">https://pypi.python.org/pypi</font>](https://pypi.python.org/pypi)查找，并通过命令行 安装（pip安装需要前置安装setuptools，请自行百度）。

启动 `nohup foreman start & ` ，如果启动不成功，需要仔细阅读 nohup.out 日志内容定位


参考：
[<font color="#2798a2">http://blog.gaoyuan.xyz/2014/10/01/use-graphite-and-alter-build-monitor-system/?utm_source=tuicool&utm_medium=referral</font>](http://blog.gaoyuan.xyz/2014/10/01/use-graphite-and-alter-build-monitor-system/?utm_source=tuicool&utm_medium=referral)
