---
layout: post
title: golang和python互相调用
category: go语言
tags: [go, golang, python]
keywords: go, golang, python, 互相调用
---


### 网站

python3-ctypes: [https://docs.python.org/3.5/library/ctypes.html#ctypes.c_wchar_p](https://docs.python.org/3.5/library/ctypes.html#ctypes.c_wchar_p)

golang-cgo: [https://golang.org/cmd/cgo/#hdr-Go_references_to_C](https://golang.org/cmd/cgo/#hdr-Go_references_to_C)

#### 综述
golang和python之间，当前可以通过golang的cgo和python的ctypes，把golang对象和python对象分别转换为C对象，从而通过编译和调用c的动态连接库，完成交互。

#### python调用golang:
go 函数实现：
```
//libadd.go
package main

import "C"

//export add
func add(left, right int) int {
    return left + right
}

func main() {
}
```
通过c-shared模式编译成so:
```
go build -buildmode=c-shared -o libadd.so libadd.go
```
python调用so:
```
from ctypes import cdll
lib = cdll.LoadLibrary('./libadd.so')
print("Loaded go generated SO library")
result = lib.add(2, 3)
print(result)
```
`注意`: 只有int可以不需要转换，直接在go和C直接互相调用

对于不同的类型，需要使用cgo中定义的方法转换，具体可以参考golang-cgo文档。

比如string需要用*C.char来传递，C.GoString(s)可以将*C.char类型转换为string，反之C.CString可以把string类型转为 *C.char

使用举例：
```
package main

import "C"
import (
	"github.com/ppmoon/gbt2260"
	"strings"
)

//export parsecode
func parsecode(s *C.char) (*C.char){
	code := C.GoString(s)
	region := gbt2260.NewGBT2260()
	localCode := region.SearchGBT2260(code)
	if len(localCode) < 3 {
		return C.CString("没有匹配上,,")
	}
	ret := strings.Join(localCode, ",")
	return C.CString(ret)
}


//export add
func add(left, right int) int {
	return left + right
}

func main() {

}
```
编译.so:
```
go build -buildmode=c-shared -o ~/Develop/law_dev/law_service/util/parseareacode.so test_other.go
```
python调用:

用.argtype和.restype可以定义调用动态连接库函数的传入和传出参数
```
# 通过golang的"github.com/ppmoon/gbt2260"包解析area code
import os
from ctypes import cdll, c_char_p

current_path = os.path.dirname(os.path.realpath(__file__))
lib = cdll.LoadLibrary(current_path + '/parseareacode.so')
parsecode = lib.parsecode

def get_list(area_code):
    parsecode.argtype = c_char_p
    parsecode.restype = c_char_p
    result = lib.parsecode(area_code.encode("utf-8"))
    return result.decode("utf-8").split(",")
```
简单总结
```
Python与Go之间的参数传递, 处理非INT型时需要都转为对应的C类型
ctypes需要显式地声明DLL函数的参数和返回期望的数据类型
注意在Python3中字符串bytes和string的区别
Go模块需要//export 声明外部可调用
Go处理C的类型是需要显式转换
```