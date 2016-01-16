---
layout: post
title: "js and coffee mixed code on nodejs"
date: 2014-10-18 22:14:22 +0800
comments: true
categories: nodejs
tags: nodejs
share: true
description: js和coffee混合编程
toc: true
---

在node平台下，用原始js和coffeeScript混合编程

<!--more-->

coffeeScript是一种包装过js的语言，可以避免一些js的不友好的语法。
介绍将2种混合起来使用。

---

## 引言

coffee和js本质是一样的，因此在nodejs平台下，可以无缝衔接起来。

为什么会有这样的需求？难道把coffee编译输出成js再使用，难道不好么？

这是因为，如果需要用express这样js框架，那么就有2种选择

* coffee  -> js        太麻烦，每次改动都要重新生成js，再使用，频繁变更。
* js      -> coffee   不科学，需要重写语法，而且不能保证将一些类似express这样的框架，转换后，是否还是工作良好


 如果能够用原生的js框架，然后业务模块，用coffee写，就太好了

 ---

## 例子

```bash app.js

xiaolongyuan@xiaolongdeMacBook-Air coffee_work$ cat app.js

// for coffeescript 1.8.0+
require('coffee-script/register')
var foo = require("./lib/foo");
console.log('the 3 + 5 = ' + foo.add(3,5))

```

```bash lib/foo.coffee
xiaolongyuan@xiaolongdeMacBook-Air coffee_work$ cat lib/foo.coffee
add =(a,b = 2) -> a + b
exports.add = add
```

运行后

```bash
xiaolongyuan@xiaolongdeMacBook-Air coffee_work$ node app.js
the 3 + 5 = 8
```

## 补充

注意先要安装coffee模块，具体方法百度很多。coffee真是给后端开发者，向前又迈了一步，赞！
