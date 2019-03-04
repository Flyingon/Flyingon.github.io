---
layout: post
title: golang timer task
category: golang
tags: [go, golang, timer, cron, crontab]
keywords: go, golang, timer, cron, crontab
---

### 网址

Cron官网: [https://godoc.org/github.com/robfig/cron](https://godoc.org/github.com/robfig/cron)

Timer官网: [https://golang.org/pkg/time/#pkg-examples](https://golang.org/pkg/time/#pkg-examples)

### Cron
#### 格式

Field name   | Mandatory? | Allowed values  | Allowed special characters
----------   | ---------- | --------------  | --------------------------
Seconds      | Yes        | 0-59            | * / , -
Minutes      | Yes        | 0-59            | * / , -
Hours        | Yes        | 0-23            | * / , -
Day of month | Yes        | 1-31            | * / , - ?
Month        | Yes        | 1-12 or JAN-DEC | * / , -
Day of week  | Yes        | 0-6 or SUN-SAT  | * / , - ?

#### 说明
```
1）星号(*)
表示 cron 表达式能匹配该字段的所有值。如在第5个字段使用星号(month)，表示每个月

2）斜线(/)
表示增长间隔，如第1个字段(minutes) 值是 3-59/15，表示每小时的第3分钟开始执行一次，之后每隔 15 分钟执行一次（即 3、18、33、48 这些时间点执行），这里也可以表示为：3/15

3）逗号(,)
用于枚举值，如第6个字段值是 MON,WED,FRI，表示 星期一、三、五 执行

4）连字号(-)
表示一个范围，如第3个字段的值为 9-17 表示 9am 到 5pm 直接每个小时（包括9和17）

5）问号(?)
只用于日(Day of month)和星期(Day of week)，\表示不指定值，可以用于代替 *
```
#### 配置示例

Entry                  | Description                                | Equivalent To
-----                  | -----------                                | -------------
@yearly (or @annually) | Run once a year, midnight, Jan. 1st        | 0 0 0 1 1 *
@monthly               | Run once a month, midnight, first of month | 0 0 0 1 * *
@weekly                | Run once a week, midnight between Sat/Sun  | 0 0 0 * * 0
@daily (or @midnight)  | Run once a day, midnight                   | 0 0 0 * * *
@hourly                | Run once an hour, beginning of hour        | 0 0 * * * *

```
每隔5秒执行一次：*/5 * * * * ?

每隔1分钟执行一次：0 */1 * * * ?

每天23点执行一次：0 0 23 * * ?

每天凌晨1点执行一次：0 0 1 * * ?

每月1号凌晨1点执行一次：0 0 1 1 * ?

在26分、29分、33分执行一次：0 26,29,33 * * * ?

每天的0点、13点、18点、21点都执行一次：0 0 0,13,18,21 * * ?
```
#### 代码示例
- 简单crontab任务

```
package main

import (
    "github.com/robfig/cron"
    "log"
)

func main() {
    i := 0
    c := cron.New()
    spec := "*/5 * * * * ?"
    c.AddFunc(spec, func() {
        i++
        log.Println("cron running:", i)
    })
    c.Start()
    
    select{}
}
```
输出:
```
cron running : 1
cron running : 2
cron running : 3
cron running : 4
cron running : 5
...
```

- 多个定时crontab任务

```
package main

import (
    "github.com/robfig/cron"
    "log"
    "fmt"
)

type TestJob struct {
}

func (this TestJob)Run() {
    fmt.Println("testJob1...")
}

type Test2Job struct {
}

func (this Test2Job)Run() {
    fmt.Println("testJob2...")
}

//启动多个任务
func main() {
    i := 0
    c := cron.New()
    
    // AddFunc
    spec := "*/5 * * * * ?"
    c.AddFunc(spec, func() {
        i++
        log.Println("cron running:", i)
    })
    
    // AddJob方法
    c.AddJob(spec, TestJob{})
    c.AddJob(spec, Test2Job{})
    
    // 启动计划任务
    c.Start()
    
    // 关闭着计划任务, 但是不能关闭已经在执行中的任务.
    defer c.Stop()
    
    select{}
}
```
输出:
```
testJob1...
2017/07/07 18:46:40 cron running: 1
testJob2...
2017/07/07 18:46:45 cron running: 2
testJob1...
testJob2...
2017/07/07 18:46:50 cron running: 3
testJob1...
testJob2...
2017/07/07 18:46:55 cron running: 4
testJob1...
testJob2...
testJob2...
testJob1...
2017/07/07 18:47:00 cron running: 5
...
```

### timer定时

代码示例:
- 按月定时

```
func StartMonthlyTimer(f func()) {
	go func() {
		for {
			f()
			now := time.Now()
			// 计算下个月零点
			year := now.Year()
			month := now.Month()
			if month == time.December {
				year += 1
				month = time.January
			} else {
				month = time.Month(int(month) + 1)
			}
			next := time.Date(year, month, 1, 0, 0, 0, 0, time.Now().Location())
			t := time.NewTimer(next.Sub(now))
			<-t.C
		}
	}()
}
```

- 按分定时

```
func StartMinuteTimer(intMin uint, f func(interface{}), i interface{}) {
	go func() {
		for {
			f(i)
			now := time.Now()
			next := now.Add(time.Duration(intMin) * time.Minute)
			next = time.Date(next.Year(), next.Month(), next.Day(), next.Hour(), next.Minute(), 0, 0, next.Location())
			t := time.NewTimer(next.Sub(now))
			<-t.C
		}
	}()

```