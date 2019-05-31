---
layout: post
title: 代码踩坑记录
category: golang
tags: [go, code, 踩坑]
keywords: go, code, 踩坑
---

## for循环中地址

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