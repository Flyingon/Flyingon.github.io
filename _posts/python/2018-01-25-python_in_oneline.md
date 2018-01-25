---
layout: post
title: python一句话可以做什么
category: python
tags: [python, 一句话, one line]
keywords: python, oneline

> 本文收集整理自网络, 觉得比较有趣, 所以记录一下

> 参考: https://zhuanlan.zhihu.com/p/23321351

#### 启动web服务
```
python -m SimpleHTTPServer 8080  # python2
python3 -m http.server 8080  # python3
```

#### 变量值互换
```
a, b = 1, 2; a, b = b, a
```

#### FizzBuzz问题
> FizzBuzz问题：打印数字1到100, 3的倍数打印“Fizz”, 5的倍数打印“Buzz”, 既是3又是5的倍数的打印“FizzBuzz”
```
print(' '.join(["fizz"[x % 3 * 4:]+"buzz"[x % 5 * 4:] or str(x) for x in range(1, 101)]))
```

#### 特定字符"Love"拼成的心形
```
print('\n'.join([''.join([('Love'[(x-y) % len('Love')] if ((x*0.05)**2+(y*0.1)**2-1)**3-(x*0.05)**2*(y*0.1)**3 <= 0 else ' ') for x in range(-30, 30)]) for y in range(30, -30, -1)]))

                veLoveLov           veLoveLov
            eLoveLoveLoveLove   eLoveLoveLoveLove
          veLoveLoveLoveLoveLoveLoveLoveLoveLoveLov
         veLoveLoveLoveLoveLoveLoveLoveLoveLoveLoveL
        veLoveLoveLoveLoveLoveLoveLoveLoveLoveLoveLov
        eLoveLoveLoveLoveLoveLoveLoveLoveLoveLoveLove
        LoveLoveLoveLoveLoveLoveLoveLoveLoveLoveLoveL
        oveLoveLoveLoveLoveLoveLoveLoveLoveLoveLoveLo
        veLoveLoveLoveLoveLoveLoveLoveLoveLoveLoveLov
        eLoveLoveLoveLoveLoveLoveLoveLoveLoveLoveLove
         oveLoveLoveLoveLoveLoveLoveLoveLoveLoveLove
          eLoveLoveLoveLoveLoveLoveLoveLoveLoveLove
          LoveLoveLoveLoveLoveLoveLoveLoveLoveLoveL
            eLoveLoveLoveLoveLoveLoveLoveLoveLove
             oveLoveLoveLoveLoveLoveLoveLoveLove
              eLoveLoveLoveLoveLoveLoveLoveLove
                veLoveLoveLoveLoveLoveLoveLov
                  oveLoveLoveLoveLoveLoveLo
                    LoveLoveLoveLoveLoveL
                       LoveLoveLoveLov
                          LoveLoveL
                             Lov
                              v
```

#### 九九乘法表
```
print('\n'.join([' '.join(['%s*%s=%-2s' % (y, x, x*y) for y in range(1, x+1)]) for x in range(1, 10)]))
```

#### 计算出1-100之间的素数
```
print(' '.join([str(item) for item in filter(lambda x: not [x % i for i in range(2, x) if x % i == 0], range(2, 101))]))
print(' '.join([str(item) for item in filter(lambda x: all(map(lambda p: x % p != 0, range(2, x))), range(2, 101))]))
```

#### 斐波那契数列
```
print([x[0] for x in [(a[i][0], a.append([a[i][1], a[i][0]+a[i][1]])) for a in ([[1, 1]], ) for i in range(30)]])
```
#### 一行代码实现快排算法
```
qsort = lambda arr: len(arr) > 1 and qsort(list(filter(lambda x: x <= arr[0], arr[1:]))) + arr[0:1] + qsort(list(filter(lambda x: x > arr[0], arr[1:]))) or arr
```

#### 解决八皇后问题
```
[__import__('sys').stdout.write('\n'.join('.' * i + 'Q' + '.' * (8-i-1) for i in vec) + "\n========\n") for vec in __import__('itertools').permutations(range(8)) if 8 == len(set(vec[i]+i for i in range(8))) == len(set(vec[i]-i for i in range(8)))]
```

#### 有趣的 python oneline 网站:
[https://arunrocks.com/python-one-liner-games/](https://arunrocks.com/python-one-liner-games/)