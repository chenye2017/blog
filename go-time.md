---
title: go time
date: 2020-06-09 19:47:50
tags: [go,time]
categories: go
---

go 中时间包的使用

<!--more-->

```
time.Now() // time 包的 Time 类型

// Time
// 2018-05-31 09:22:19 +0800 CST Time 类型打印出来长这样
time.Now() // 获取当前时间的Time 类型
time.Now().Unix()  Time 类型装 int64
time.Now().Unix()Formate("2006-01-02 15:04:05") // 格式化
time.Now().in(a)  // 使用某个时区，返回的是Time 类型，所以可以用

// location
time.Location() // 返回时区信息
a,_ := time.LoadLocation("UTC")  // time 包的 location,如果想用时区，就得用这个


// 数字转Time
time.Unix(11111)

// 字符串转Time
time.Parse() 和 Time.Format() 是互逆的两个函数
time.Parse() 解析字符串成Time 的时候需要时区信息，我们可以用Loadlocation获取时区信息，然后作为第三个参数传进去

// Duration 
这个在sleep 的时候经常用到
const (
    Nanosecond  Duration = 1
    Microsecond          = 1000 * Nanosecond
    Millisecond          = 1000 * Microsecond
    Second               = 1000 * Millisecond
    Minute               = 60 * Second
    Hour                 = 60 * Minute
)


```

今天遇到一个问题，比如sleep() ,这个参数需要是 time.duration, 但是我需要可控的 整数， 6 * time.Second， 这样执行不了，需要把 6转换成 time.Duration(6) * time.Second, 这样就能运算了。

这个其实并不是包的特殊性，还是golang语法自身问题自己了解的不足

```
type c int

var cparam int
cparam = 1

o := 1

fmt.Println(cparam == o)  // 必然报错，虽然是别名，但还是不同类型

time.Sleep(6 * time.Second) // 这是因为 6可以当做 time.Duration 类型

可是
r := rand.Int() // int

time.Sleep(r * time.Second) // 不同的数据类型不能相乘
```

```
今天还发现 time 包的一个很牛逼的方法，涉及channl， 所以只有 goroutine 的时候用得到。 time.After() 是一个channl 在规定时间后可以读取一个数据，利用select 机制，可以控制超时问题。
```







今天 发现go 去修改时间也很方便，比如



time.Add() 比如 + 1天， -1 天，

Sub() 计算两天的差值（不是用来算负数天的哦）

