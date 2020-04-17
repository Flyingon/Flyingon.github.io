---
layout: post
title: 踩坑记录
category: golang
tags: [go, code, 踩坑]
keywords: go, code, 踩坑
---

## for循环中地址问题

### 遍历有缓冲的chan, 赋值结果到指针数组
测试代码: [chan2Slice](https://github.com/Flyingon/code_tpl_go/blob/master/for_channel/chan2slice/chan2Slice.go) 

缓冲chan通常用作多协程的结果返回

- 异常代码

```go
// 循环读取channel，取地址append到数组
type ChanStruct struct {
	Num  int
	Data string
}

func main() {
	// 定义chan
	ptrChan := make(chan ChanStruct, 10)
	for i:= 1;i < 10;i++ {
		cs := ChanStruct {
			Num: i,
			Data: strconv.Itoa(i),
		}
		ptrChan <- cs
	}
	close(ptrChan)
	var csList []*ChanStruct
	// 从chan取数据，放入csList中
	for c := range ptrChan {
		csList = append(csList, &c)
	}
	for _, cs := range csList {
		fmt.Printf("cs: %d, %s\n", cs.Num, cs.Data)
	}
}
```

- 输出结果

结果不符合预期，新数组中所有元素都是同一个地址

```go
cs: 0xc000086000, 9, 9
cs: 0xc000086000, 9, 9
cs: 0xc000086000, 9, 9
...
```

- 正确方式

_for循环的局部变量地址是不会变的，如果要赋值地址给别人，需要新生成对象，再取地址_

```go
// 循环读取channel，取地址append到数组
type ChanStruct struct {
	Num  int
	Data string
}

func main() {
	// 定义chan
	ptrChan := make(chan ChanStruct, 10)
	for i:= 1;i < 10;i++ {
		cs := ChanStruct {
			Num: i,
			Data: strconv.Itoa(i),
		}
		ptrChan <- cs
	}
	close(ptrChan)
	var csList []*ChanStruct
	// 从chan取数据，放入csList中
	for c := range ptrChan {
		temp := c
		csList = append(csList, &temp)
	}
	for _, cs := range csList {
		fmt.Printf("cs: %d, %s\n", cs.Num, cs.Data)
	}
}
```

### 遍历指针数组，使用遍历时的临时变量

测试代码: [for_with_coroutine](https://github.com/Flyingon/code_tpl_go/blob/master/for_closure/for_with_coroutine.go) 

- 错误示例1: 

代码片段:
```go
	for i, v := range dataList {
		go func() {
			defer wg.Done()
			fmt.Printf("cycle %d time, index addr: %p, data addr: %p, value: %s\n", i, &i, &(v.Data), v.Data)
		}()
	}
```
返回: 
```go
cycle 4 time, index addr: 0xc000088020, data addr: 0xc000062230, value: value5
cycle 4 time, index addr: 0xc000088020, data addr: 0xc000062230, value: value5
cycle 4 time, index addr: 0xc000088020, data addr: 0xc000062230, value: value5
cycle 4 time, index addr: 0xc000088020, data addr: 0xc000062230, value: value5
cycle 4 time, index addr: 0xc000088020, data addr: 0xc000062230, value: value5
```

- 正确示例1：

代码片段:
```go
for i, v := range dataList {
		go func(index int, elem *Elem) {
			defer wg.Done()
			fmt.Printf("cycle %d time, index addr: %p, data addr: %p, value: %s\n", index, &index, &(elem.Data), elem.Data)
		}(i, v)
	}
```
返回: 
```go
cycle 0 time, index addr: 0xc000092010, data addr: 0xc0000621e0, value: value1
cycle 4 time, index addr: 0xc000096020, data addr: 0xc000062230, value: value5
cycle 3 time, index addr: 0xc000096030, data addr: 0xc000062220, value: value4
cycle 1 time, index addr: 0xc000018098, data addr: 0xc0000621f0, value: value2
cycle 2 time, index addr: 0xc000092018, data addr: 0xc000062210, value: value3
```

- 错误示例2：

代码片段:
```go
for i, v := range dataList {
		go DataPrintV3(&wg, &i, v)
	}
```
返回: 
```go
cycle 4 time, index addr: 0xc000092028, data addr: 0xc0000621e0, value: value1
cycle 4 time, index addr: 0xc000092028, data addr: 0xc000062230, value: value5
cycle 4 time, index addr: 0xc000092028, data addr: 0xc0000621f0, value: value2
cycle 4 time, index addr: 0xc000092028, data addr: 0xc000062210, value: value3
cycle 4 time, index addr: 0xc000092028, data addr: 0xc000062220, value: value4
```

- 正确示例2：

代码片段:
```go
	for i, v := range dataList {
		go DataPrintV4(&wg, i, v)
	}
```
返回: 
```go
cycle 4 time, index addr: 0xc000088050, data addr: 0xc000062230, value: value5
cycle 2 time, index addr: 0xc000088060, data addr: 0xc000062210, value: value3
cycle 3 time, index addr: 0xc000088070, data addr: 0xc000062220, value: value4
cycle 0 time, index addr: 0xc000088080, data addr: 0xc0000621e0, value: value1
cycle 1 time, index addr: 0xc0000180b0, data addr: 0xc0000621f0, value: value2
```

## map并发读写问题