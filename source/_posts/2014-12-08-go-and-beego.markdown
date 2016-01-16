---
layout: post
title: "go and beego"
date: 2014-12-08 22:33:01 +0800
comments: true
categories: go
tags: go
share: true
description: go语言介绍
toc: true
---

golang 简称 go ，是google新推的一种语言，目前火热的docker就是用go语言开发完成的

<!--more-->

---

## GO安装及环境配置(本文说明一下在linux下如何安装)

百度云下载地址:

* [<font color="#323d89">linux版64位</font>](http://pan.baidu.com/s/1ntuoO5N)
* [<font color="#323d89">windows版64位</font>](http://pan.baidu.com/s/1dDndE4x)
* [<font color="#323d89">mac版</font>](http://pan.baidu.com/s/1pJFj1Ej)



Step 1. 下载好后，将文件解压到 `/usr/local/` 下

Step 2. 在 `~/.bash_profile` 配置环境变量( GOPATH 是 GO下载依赖包后存放的地方，有点类似maven的本地仓库)

``` bash
#golang
export GOROOT=/usr/local/go
export GOBIN=$GOROOT/bin
export GOOS=linux
export GOARCH=amd64
export GOPATH=/home/web/xiaolong.yuanxl/software/golang
export PATH=$PATH:$GOBIN
```

Step 3. 生效文件

``` bash
source ~/.bash_profile
```

Step 4.验证

``` bash
[web@wh-9-95 software]$ go version
go version go1.4beta1 linux/amd64
```

---

## beego 轻量的web框架

beego 是用go语言写的一个 web框架 ，[<font color="#323d89">Github地址</font>](https://github.com/astaxie/beego/) ,对付一般的网站此框架足以。

[beego官网文档](http://beego.me/docs/intro/) 已经十分的详细了，这里就是不在赘述。

---

## GOPM 第三方的 GO包管理工具


值得说的是，go 并没有自带的 包管理工具，这对于 java ,ruby ,nodejs这样的同学来说，就感到十分困惑了。

这里推荐一个包管理工具 gopm [地址](https://github.com/GPMGo/gopm)，你可以在开发环境使用它，而到生产环境时 就用 ``` go get ```来获取依赖。

使用gopm有3个好处
 1.  它隔离了GOPATH在当前的工程，以 `.vendor` 文件夹的形式存在
 2.  有一个内容描述文件 `.gopmfile`，但这个文件不像`maven`的 pom.xml ，为什么？因为 go 获取依赖包，<b>最终</b>是根据代码里的 `import`来的,所以此文件仅展示用.
 3.  `gopm list`可以快速展现当前工程依赖了哪些包

当然缺点就是，它没有像 beego 的 `bee run`这样有热加载能力

---

## GO 教程及文档

由于“墙”的问题，golang.org 经常上不去，因此我们可以访问

* [<font color="#2c977e">golang中国——包文档</font>](http://godoc.golangtc.com/pkg/)

* [<font color="#2c977e">在线学习手册</font>](http://www.vaikan.com/go/a-tour-of-go)
