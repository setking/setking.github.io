---
title: ES6模块与CommonJS模块的差异
author: setKing
date: 2023-11-06 11:33:00 +0800
categories: [学习]
tags: [学习]
pin: true
math: true
mermaid: true
---

## 传统加载

- 默认情况下，浏览器是同步加载 Javascript 脚本，即渲染引擎遇到`<script>`标签就会停下来，等到执行完脚本，在向下继续渲染，如果是外部脚本还要加入脚本下载时间
- 如果脚本过大，下载时间会很长，容易造成浏览器 I/O 堵塞，于是浏览器允许脚本异步加载，以下是两种加载方法：
  - `defer`：defer 要等到整个页面在内存中正常渲染结束（DOM 结构完全生成，以及其他脚本执行完成），才会执行
  - `async`：async 一旦下载完成，渲染引擎就会中断渲染，执行这个脚本以后，再继续渲染

> 总结就是 defer 是”渲染完再执行“，async 是“下载完就执行”。如果有多个 defer 脚本，会顺序加载，而多个 async 脚本不能保证加载顺序

## ES6 和 CommonJS 模块

### 差异

- `CommonJS`模块输出的是一个值的拷贝，ES6 模块输出的是值的引用

- `CommonJS`模块是运行时加载、ES6 模块是编译时输出接口

  > 这个差异是因为`CommonJS`加载的是一个对象（即`module.export`属性），该对象只有在脚本运行完才会生成。而 ES6 模块不是对象，它的对外接口只是一种静态定义，在代码静态解析阶段就会生成。

- `CommonJS`模块的`require()`是同步加载模块，ES6 模块的`import()`命令是异步加载，有一个独立模块依赖的解析阶段

### 互操作

- nodejs14 以上版本 ESM 模块能够通过`default import`、`name import`、`namespace import`等方式导入 CJS 模块，但反过来 CJS 模块只能通过`dynamic import`即`import()`导入 ESM 模块

## Node.js 模块加载方法

- JavaScript 现在有两种模块。一种是 ES6 模块，简称 ESM；另一种是 CommonJS 模块，简称 CJS。
  CJS 是 Node.js 专用，与 ESM 不兼容，语法上有明显差异。CJS 使用`require()`和`module.exports`，ESM 使用`import`和`export`

- node.js 要求 ESM 采用.mjs 后缀文件名。node.js 遇到.mjs 文件，就默认它是 ESM 模块，默认启用严格模式。

  > `package.json的type`字段指定为`module`也能打到同等作用

- CommonJS 脚本的后缀名都改成`.cjs`。如果没有`type`字段，或者`type`字段为`commonjs`，则`.js`脚本会被解释成 CommonJS 模块。
