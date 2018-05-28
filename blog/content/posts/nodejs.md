---
title: "Nodejs"
date: 2018-05-28T10:42:25+08:00
draft: true
---
Node.js是一个事件驱动io服务端js环境，基于google的v8引擎。

Node.js使用事件驱动模型，当web server接收到请求，就把它关闭然后进行处理，然后去服务下一个web请求。 当这个请求完成，它被放回处理队列，当到达队列开头，这个结果被返回给用户。这也被称之为非阻塞式IO或者事件驱动IO。有点类似于观察者模式，事件相当于一个主题(Subject)，而所有注册到这个事件上的处理函数相当于观察者(Observer)。 

模块是Node.js 应用程序的基本组成部分，文件和模块是一一对应的。换言之，一个 Node.js 文件就是一个模块，这个文件可能是JavaScript 代码、JSON 或者编译过的C/C++ 扩展。 

package.json 位于模块的目录下，用于定义包的属性。使用npm init创建模块。

#### npm
```
npm install <Module Name>  /  npm install express
var express = require('express');
npm config set proxy null
npm list -g / npm list grunt
npm install -g cnpm --registry=https://registry.npm.taobao.org
cnpm install [name]
```
[npm命令使用](https://npmjs.org/doc/)

#### react
```
cnpm install -g create-react-app
create-react-app my-app
```


#### ECMAScript 6

[ECMAScript 6 简介](http://es6.ruanyifeng.com/#docs/intro)
