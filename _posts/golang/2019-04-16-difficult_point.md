---
layout: post
title: 代码踩坑记录
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


### 遍历指针数组，赋值到新数组中
- 异常代码

遍历结构体数组tArray，复制该数组中每个元素Data的指针到新的指针数组retArray中

```go
type TArray struct {
    Data string
}

func main() {
    data1 := "value1"
    data2 := "value2"
    tArray := [] TArray{
        {
            data1,
        }, {
            data2,
        },
    }
    var retArray []*string
    for i, v := range tArray {
        fmt.Printf("cycle %d time, v addr: %p\n", i, &v)
        fmt.Printf("cycle %d time, current tArray instance data addr: %p, value: %s\n", i, &(v.Data), v.Data)
        retArray = append(retArray, &(v.Data))
    }
    fmt.Printf("ret array: %+v\n", retArray)
    for _, r := range retArray {
        fmt.Printf("%s ", *r)
    }
}
```

- 输出结果

结果不符合预期，新数组中两个元素是同一个地址，即同一个元素

```go
cycle 0 time, v addr: 0xc0420481d0
cycle 0 time, current tArray instance data addr: 0xc0420481d0, value: value1
cycle 1 time, v addr: 0xc0420481d0
cycle 1 time, current tArray instance data addr: 0xc0420481d0, value: value2
ret array: [0xc0420481d0 0xc0420481d0]
value2 value2 
```

- 正确方式

1. 遍历指针数组不会有该问题，每个元素都有独立地址

```go
type TArray struct {
    Data string
}

func main() {
    data1 := "value1"
    data2 := "value2"
    tArray := [] *TArray{
        {
            data1,
        }, {
            data2,
        },
    }
    var retArray []*string
    for i, v := range tArray {
        fmt.Printf("cycle %d time, v addr: %p\n", i, &v)
        fmt.Printf("cycle %d time, current tArray instance data addr: %p, value: %s\n", i, &(v.Data), v.Data)
        retArray = append(retArray, &(v.Data))
    }
    fmt.Printf("ret array: %+v\n", retArray)
    for _, r := range retArray {
        fmt.Printf("%s ", *r)
    }
}
```

2. 给遍历的结构体Data重新分配地址，赋值到newData

```go
type TArray struct {
    Data string
}

func main() {
    data1 := "value1"
    data2 := "value2"
    tArray := [] TArray{
        {
            data1,
        }, {
            data2,
        },
    }
    var retArray []*string
    for i, v := range tArray {
        fmt.Printf("cycle %d time, v addr: %p\n", i, &v)
        fmt.Printf("cycle %d time, current tArray instance data addr: %p, value: %s\n", i, &(v.Data), v.Data)
        newData := v.Data
        retArray = append(retArray, &newData)
    }
    fmt.Printf("ret array: %+v\n", retArray)
    for _, r := range retArray {
        fmt.Printf("%s ", *r)
    }
}
```