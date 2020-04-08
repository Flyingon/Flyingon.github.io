---
layout: post
title: FOR循环中的协程
category: golang
tags: [go, golang, for, coroutine]
keywords: go, golang, for, coroutine
---

## FOR循环中启动协程


代码示例:

```go
package main

import (
	"fmt"
	"sync"
)

type Elem struct {
	Data string
}

// ClosureInForV1 不符合预期的使用
func ClosureInForV1(dataList []*Elem) {
	wg := sync.WaitGroup{}
	wg.Add(len(dataList))
	for i, v := range dataList {
		go func() {
			defer wg.Done()
			fmt.Printf("cycle %d time, index addr: %p, data addr: %p, value: %s\n", i, &i, &(v.Data), v.Data)
		}()
	}
	wg.Wait()
}

// ClosureInForV2 符合预期饿的使用
func ClosureInForV2(dataList []*Elem) {
	wg := sync.WaitGroup{}
	wg.Add(len(dataList))
	for i, v := range dataList {
		go func(index int, elem *Elem) {
			defer wg.Done()
			fmt.Printf("cycle %d time, index addr: %p, data addr: %p, value: %s\n", index, &index, &(elem.Data), elem.Data)
		}(i, v)
	}
	wg.Wait()
}

// ClosureInForV3 符合预期饿的使用
func ClosureInForV3(dataList []*Elem) {
	wg := sync.WaitGroup{}
	wg.Add(len(dataList))
	for i, v := range dataList {
		DataPrint(&wg, i, v)
	}
	wg.Wait()
}

func DataPrint(wg *sync.WaitGroup, index int, elem *Elem) {
	defer wg.Done()
	fmt.Printf("cycle %d time, index addr: %p, data addr: %p, value: %s\n", index, &index, &(elem.Data), elem.Data)
}

func main() {
	var tArray []*Elem
	tArray = append(tArray, &Elem{"value1"})
	tArray = append(tArray, &Elem{"value2"})
	tArray = append(tArray, &Elem{"value3"})
	tArray = append(tArray, &Elem{"value4"})
	tArray = append(tArray, &Elem{"value5"})
	fmt.Printf("------v1-------\n")
	ClosureInForV1(tArray)
	fmt.Printf("------v2-------\n")
	ClosureInForV2(tArray)
	fmt.Printf("------v3-------\n")
	ClosureInForV3(tArray)
}

```

返回结果:

```go
------v1-------
cycle 4 time, index addr: 0xc000088020, data addr: 0xc000062230, value: value5
cycle 4 time, index addr: 0xc000088020, data addr: 0xc000062230, value: value5
cycle 4 time, index addr: 0xc000088020, data addr: 0xc000062230, value: value5
cycle 4 time, index addr: 0xc000088020, data addr: 0xc000062230, value: value5
cycle 4 time, index addr: 0xc000088020, data addr: 0xc000062230, value: value5
------v2-------
cycle 0 time, index addr: 0xc000092010, data addr: 0xc0000621e0, value: value1
cycle 4 time, index addr: 0xc0000a8020, data addr: 0xc000062230, value: value5
cycle 3 time, index addr: 0xc0000a8030, data addr: 0xc000062220, value: value4
cycle 1 time, index addr: 0xc0000a8040, data addr: 0xc0000621f0, value: value2
cycle 2 time, index addr: 0xc000092018, data addr: 0xc000062210, value: value3
------v3-------
cycle 0 time, index addr: 0xc000092028, data addr: 0xc0000621e0, value: value1
cycle 1 time, index addr: 0xc000092040, data addr: 0xc0000621f0, value: value2
cycle 2 time, index addr: 0xc000092050, data addr: 0xc000062210, value: value3
cycle 3 time, index addr: 0xc000092060, data addr: 0xc000062220, value: value4
cycle 4 time, index addr: 0xc000092070, data addr: 0xc000062230, value: value5

```