---
layout: post
title: coroutine
category: golang
tags: [go, golang, coroutine]
keywords: go, golang, coroutine
---

## 多协程安全

### 加锁不加锁代码验证:

- map: [use_map](https://github.com/Flyingon/code_tpl_go/blob/master/for_coroutine/use_map/use_map.go)

- slice: [use_slice](https://github.com/Flyingon/code_tpl_go/blob/master/for_coroutine/use_slice/use_slice.go)

### 用channel解决

- 用chan解决多协程结果汇总: [use_chan](https://github.com/Flyingon/code_tpl_go/blob/master/for_coroutine/use_chan/use_chan.go)