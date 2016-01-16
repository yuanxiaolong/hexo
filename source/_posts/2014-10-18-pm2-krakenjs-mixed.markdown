---
layout: post
title: "pm2 krakenjs mixed"
date: 2014-10-18 23:06:32 +0800
comments: true
categories: nodejs
tags: nodejs
share: true
description: 将pm2应用在krakenjs上
toc: true
---

终于找到一种方法，可以让krakenjs框架的工程，在pm2上托管。

<!--more-->

krakenjs 是paypal 开发的一个mvc的js框架。

pm2 是nodejs 的一个监控、运维托管的模块，可以启动、关闭自己的Node程序。

---

## 整合

1.将kraken自动生成的index.js，改成下面的样子，以便代理模式应用。

```js index.js
'use strict';

var kraken = require('kraken-js'),
    // app = {};
    delegate = require('./delegate');

// 采用代理模式,将pm2和krakenJs 整合

// app.configure = function configure(nconf, next) {
//     // Async method run on startup.
//     next(null);
// };
//
// app.requestStart = function requestStart(server) {
//     // Run before most express middleware has been registered.
// };
//
// app.requestBeforeRoute = function requestBeforeRoute(server) {
//     require('dustjs-linkedin').optimizers.format = function(ctx, node) { return node };
//     // Run before any routes have been added.
// };
//
// app.requestAfterRoute = function requestAfterRoute(server) {
//     // Run after all routes have been added.
// };
//
// if (require.main === module) {
//     kraken.create(app).listen(function (err) {
//         if (err) {
//             console.error(err.stack);
//         }
//     });
// }
// module.exports = app;

kraken.create(delegate).listen(function (err) {
    if (err) {
        console.error(err.stack);
    }
});

```

2.同级目录新建delegate.js

```js delegate.js
'use strict';

/**
 * 用于代理请求,以便使用pm2
 */
module.exports = {

    configure: function configure(nconf, next) {
        // Async method run on startup
        next(null);
    },

    requestStart: function requestStart(server) {
        // Run before most express middleware has been registered
    },

    requestBeforeRoute: function requestBeforeRoute(server) {
        // Run before any routes have been added
        require('dustjs-linkedin').optimizers.format = function(ctx, node) { return node };
    },

    requestAfterRoute:  function requestAfterRoute(server) {
        // Run after all routes have been added
    }
};
```
---

## Issue

[https://github.com/krakenjs/generator-kraken/issues/35](https://github.com/krakenjs/generator-kraken/issues/35)
