---
layout: post
title: IT基础资源
category: 计算机
tags: [IT, CS, Knowledge]
keywords: IT, CS, 计算机, 面试
---

####  算法可视化: [https://www.cs.usfca.edu/~galles/visualization/Algorithms.html](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html)

### ascii码表
ascii code table: [http://www.yuanzhaoyi.cn/resource/ascii.html](http://www.yuanzhaoyi.cn/resource/ascii.html)

### 字节序（Byte Order）
我们一般把字节（byte）看作是数据的最小单位。当然，其实一个字节中还包含8个bit (bit = binary digit)。 在一个32位的CPU中“字长”为32个bit，也就是4个byte。在这样的CPU中，总是以4字节对齐的方式来读取或写入内存， 那么同样这4个字节的数据是以什么顺序保存在内存中的呢？
#### 字节序包括：大端序(big endian)和小端序(little endian)
##### 不错的说明博客： [大端序与小端序](https://www.cnblogs.com/graphics/archive/2011/04/22/2010662.html)
字节和地址高端底端说明：
```
int a = 0x12345678 ; 那么左边12就是高位字节，右边的78就是低位字节，从左到右，由高到低，（注意，高低乃相对而言，比如56相对于78是高字节，相对于34是低字节）
地址的高端与低端
0x00000001
0x00000002
0x00000003
0x00000004
从上倒下，由低到高，地址值小的为低端，地址值大的为高端。
```
- 大端序: 数据的高位字节存放在地址的低端 低位字节存放在地址高端
```
0x00000001           -- 12
0x00000002           -- 34
0x00000003           -- 56
0x00000004           -- 78
```
- 小端序: 数据的高位字节存放在地址的高端 低位字节存放在地址低端
```
0x00000001           -- 78
0x00000002           -- 56
0x00000003           -- 34
0x00000004           -- 12
```