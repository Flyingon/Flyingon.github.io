---
layout: post
title: TypeScript
category: 前端
tags: [前端，ts]
keywords: web，ts
---

### 资料
- TypeScript 中文手册: [https://typescript.bootcss.com/basic-types.html](https://typescript.bootcss.com/basic-types.html)

### for loop

```
for (let i = 0; i < 3; i++) {
  console.log ("Block statement execution no." + i);
}
```
```
Block statement execution no.0
Block statement execution no.1
Block statement execution no.2
```

```
let arr = [10, 20, 30, 40];

for (var val of arr) {
  console.log(val); // prints values: 10, 20, 30, 40
}
```
```
let str = "Hello World";

for (var char of str) {
  console.log(char); // prints chars: H e l l o  W o r l d
}
```
```
let arr = [10, 20, 30, 40];

for (var index in arr) {
  console.log(index); // prints indexes: 0, 1, 2, 3

  console.log(arr[index]); // prints elements: 10, 20, 30, 40
}
```
```
let arr = [10, 20, 30, 40];

for (var index1 in arr) {
  console.log(index1); // prints indexes: 0, 1, 2, 3
}
console.log(index1); //OK, prints 3 

for (let index2 in arr) {
  console.log(index2); // prints elements: 0, 1, 2, 3
}
console.log(index2); //Compiler Error: Cannot find index2
```

### 引用

```
npm i --save-dev @types/ws
```