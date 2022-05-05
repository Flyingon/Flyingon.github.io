---
layout: post
title: JavaScript
category: 前端
tags: [前端，js]
keywords: web，js
---

### 资料
- 重新介绍 JavaScript: [https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/A_re-introduction_to_JavaScript](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/A_re-introduction_to_JavaScript)

- 现代 JavaScript 教程: [https://zh.javascript.info/](https://zh.javascript.info/)

### 问题记录
#### npm ERR! code ELIFECYCLE
```shell
npm cache clean --force
delete node_modules folder
delete package-lock.json file
npm install
```

### 基本语法

#### for循环
```
for (let i = 0; i < arr.length; ++i)
arr.forEach((v, i) => { /* ... */ })
for (let i in arr)
for (const v of arr)
```