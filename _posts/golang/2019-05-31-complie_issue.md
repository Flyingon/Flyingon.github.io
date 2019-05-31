---
layout: post
title: 编译踩坑记录
category: golang
tags: [go, complie, 踩坑]
keywords: go, complie, 踩坑
---

## 编译报错

### 链接错误(link issue)

- 问题现象
```shell
go build -o ../bin/app_server -v app_server.go

# command-line-arguments
/usr/local/go/pkg/tool/linux_amd64/link: running g++ failed: exit status 1
/bin/ld: /tmp/go-link-576289596/000026.o: unrecognized relocation (0x2a) in section `.text'
/bin/ld: final link failed: Bad value
collect2: error: ld returned 1 exit status
```

- 环境版本
```shell
go version: go version go1.12.5 linux/amd64
g++ version: g++ (GCC) 4.8.5 20150623 (Red Hat 4.8.5-5) Copyright (C) 2015 Free Software Foundation, Inc. This is free software; see the source for copying conditions. There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
ld version: GNU ld version 2.23.52.0.1-55.el7 20130226
```

- 编译脚本预处理
```shell
#!/bin/bash
source /etc/profile
if [ -f ~/.bash_profile ]; then
  source ~/.bash_profile
fi
if [ -f ~/.bashrc ]; then
  source ~/.bashrc
fi
```
其中 ~/.bashrc最后一句包含切换g++版本环境变量命令:
```shell
source scl_source enable devtoolset-7
```

- go issue链接: [issue_31293](https://github.com/golang/go/issues/31293)

- 问题分析

基本原因是go1.12以上版本需要更高版本的ld支持，g++ 4.8对应的ld版本过低

- 问题原因

1. 仔细检查发现/etc/profile文件中最后一句写死了PATH环境变量定义
```shell
export PATH=/root/anaconda3/bin:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/data/home/go_proj/going_proj/bin:/usr/local/go/bin:/data/home/go_proj/going_proj/tools/FlameGraph:/root/bin:/usr/local/bin:/usr/libexec/git-core
```
2. 当执行source /etc/profile初始化包含该语句的profile文件后，切换g++版本环境变量命令无法生效

以下两句，都无法正确的在PATH中添加新版本g++路径：/opt/rh/devtoolset-7/root/usr/bin
```shell
source scl_source enable devtoolset-7
scl enable devtoolset-7 bash
```

3. 考虑到/etc/profile中这句PATH设置没什么作用，先删除解决问题，但具体profile设置PATH就会导致该问题的原因还有待进一步分析。



