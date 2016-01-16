---
layout: post
title: "try python fabric"
date: 2015-04-29 22:56:52 +0800
comments: true
categories: ['python']
tags: python
share: true
description: python的自动化运维工具fabric
toc: true
---

试用fabric

<!--more-->

fabric是一个自动化运维工具，简单快捷

---

## 环境准备

fabric能够高效的处理ssh这样运维的事情，类似于 scp 但又强大于它。

1.本机先装python（略）

2.安装python的包管理工具  [<font color="#2551b2">pip 官网</font>](https://pip.pypa.io/en/stable/installing.html)

``` bash
wget https://bootstrap.pypa.io/get-pip.py
sudo python get-pip.py
```

小技巧
 * `pip list | grep fabric` 快速查看到 fabric 的版本
 * `pip show fabric` 查看 fabric 的详情

3.安装fabric包

``` bash
xiaolongyuan@xiaolongdeMacBook-Air workspace_python$ sudo pip install fabric

Downloading Fabric-1.10.1.tar.gz (209kB)
    100% |████████████████████████████████| 212kB 156kB/s
Collecting paramiko>=1.10 (from fabric)
  Downloading paramiko-1.15.2-py2.py3-none-any.whl (165kB)
    100% |████████████████████████████████| 167kB 113kB/s
Collecting ecdsa>=0.11 (from paramiko>=1.10->fabric)
  Downloading ecdsa-0.13-py2.py3-none-any.whl (86kB)
    100% |████████████████████████████████| 90kB 75kB/s
Collecting pycrypto!=2.4,>=2.1 (from paramiko>=1.10->fabric)
  Downloading pycrypto-2.6.1.tar.gz (446kB)
    100% |████████████████████████████████| 446kB 82kB/s
Installing collected packages: ecdsa, pycrypto, paramiko, fabric
  Running setup.py install for pycrypto
  Running setup.py install for fabric
Successfully installed ecdsa-0.13 fabric-1.10.1 paramiko-1.15.2 pycrypto-2.6.1
```
---

## 简单使用

1.进入 [<font color="#2551b2">fabric 官网</font>](http://www.fabfile.org/) ，照着demo做，创建名字为 `fabfile.py` 的文件

``` python fabfile.py
from fabric.api import run

def host_type():
    run('uname -s')
```


2.执行命令 (其中 `root@com.x.logservernode2` 是远端机器的名字，根据需要修改需要ssh到的机器)


``` bash
xiaolongyuan@xiaolongdeMacBook-Air workspace_python$ fab -H localhost,root@com.x.logservernode2 host_type
Traceback (most recent call last):
  File "/usr/local/bin/fab", line 5, in <module>
    from pkg_resources import load_entry_point
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/Extras/lib/python/pkg_resources.py", line 2603, in <module>
    working_set.require(__requires__)
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/Extras/lib/python/pkg_resources.py", line 666, in require
    needed = self.resolve(parse_requirements(requirements))
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/Extras/lib/python/pkg_resources.py", line 565, in resolve
    raise DistributionNotFound(req)  # XXX put more info here
pkg_resources.DistributionNotFound: paramiko>=1.10

```

报错了，但是我们能看到刚才安装的 `paramiko`是大于1.10的

``` bash
xiaolongyuan@xiaolongdeMacBook-Air workspace_python$ pip show paramiko
---
Metadata-Version: 2.0
Name: paramiko
Version: 1.15.2
Summary: SSH2 protocol library
Home-page: https://github.com/paramiko/paramiko/
Author: Jeff Forcier
Author-email: jeff@bitprophet.org
License: LGPL
Location: /Library/Python/2.7/site-packages
Requires: ecdsa, pycrypto
```

解决方法：

``` bash
xiaolongyuan@xiaolongdeMacBook-Air workspace_python$ sudo pip install -U setuptools

Downloading setuptools-15.2-py2.py3-none-any.whl (501kB)
    100% |████████████████████████████████| 503kB 150kB/s
Installing collected packages: setuptools
  Found existing installation: setuptools 0.6c12dev-r88846
    Uninstalling setuptools-0.6c12dev-r88846:
      Successfully uninstalled setuptools-0.6c12dev-r88846
Successfully installed setuptools-15.2

```

而升级 `setptools` 之前是mac系统自带的旧版本，所以有这个错误

``` bash
xiaolongyuan@xiaolongdeMacBook-Air workspace_python$ pip list | grep setuptools

setuptools (0.6c12dev-r88846)
```

3.再次执行，成功！

``` bash
xiaolongyuan@xiaolongdeMacBook-Air workspace_python$ fab -H localhost,root@com.x.logservernode2 host_type
[localhost] Executing task 'host_type'
[localhost] run: uname -s
[localhost] out: Darwin
[localhost] out:

[root@com.x.logservernode2] Executing task 'host_type'
[root@com.x.logservernode2] run: uname -s
[root@com.x.logservernode2] out: Linux
[root@com.x.logservernode2] out:


Done.
Disconnecting from root@com.x.logservernode2... done.
Disconnecting from localhost... done.

```
