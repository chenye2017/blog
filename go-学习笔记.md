---
title: go 基础变量
date: 2019-11-17 15:44:14
tags: [go,php]
categories: go
---

php 活不下去了，┭┮﹏┭┮，开始学习go了，其实还好，多一门语言，多一个视角，~希望2020年识货给我的礼物就是go 和 node 的编写技能

<!--more-->

#### 变量

slice 切片的申明

```
slice  底层是指向数组的指针,所以如果我们用 var 去声明的话，默认得到的是一个nil
相比较map 的nil， slice 的 nil 我们是可以append，map的nil 之后我们是没法属性或者给属性赋值的。

一般 arr := make(a, 0, 10)
arr := []int{1,2}  // 这种方式也用的多，容易被忽略

ps: 在go中经常有 type cy int, 这种别名
[]cy{}, 这样给切片值其实很正常，类似 []int{}
```



map 的申明

```
map
map[string]string{"namme": "ccc"}

```







变量申明多种方式

```
var num int  // 没给初始值，默认 0
num := 1  // 这样更简洁更方便， 函数外不给使用

int, float64, bool 这都是和别的语言一样的，array , slice, map

arr1 := [...]int{1,2}  // 因为不能扩展，用的比较少
slice1 := []int{1,2}  // 关联数组
map1 := map1[string]string{} // 索引数组

// 换一种看法看go， 下面两种都代表数组，只是填充的对象是byte 或者是rune
[]byte{}
[]rune{}
(不管是byte 还是 rune，对应的字符序列都是 十进制，要想转换成 十六进制需要通过 hex.encodetostring ,或者 printf , 第一个只能对 slice 处理，第二个可以对数组)


// 结构体，类似
type cy struct {
		name string `json:Name` // json 的时候生成key 的名称
	}

cy1 := new(cy)
cy1.name = "haha"

fmt.Println(cy1.name)
//cy 类似类名，所以我们还需要变量名

// 别名
type

// map value 定义成某个字段太局限了，可以 map[string]interface{} 这样，更灵活
// :=  map[string]interface{}{} ,一定要给初始值

// go 中 printf %v  struct 这种不够，因为 名称打印不出来，需要 %+v 打印

```

一次性给多个值

```
arr1, arr2 := []int{1,2}, []int{2,3}
```

因为go 中的异常机制和php还是有点区别的，很多函数在使用之后会有error 的返回，我们根据 error != nil ,就能知道是否错误

```

if _,error := test(x , y ); error != nil {
  错误处理
} 
:=  ,只有多个变量中有一个是新的，:= 不会报错

真实因为go 的这种语言机制，导致 if 才能这样用，最多有两个表达式，第二个 值的内容是布尔 （上面 _ 属于匿名）

还有go 中和 js let 一样，都有块状作用域，比如那个 if 里面赋值的变量，只能在 {} 中使用
```

var 和 const 一次性声明多个值

```
var (
	test1 int
	test2 int
)

// 枚举，内部可以用iota
const (
	test2 iota
	test3
)
```



#### 闭包

go 中经常用到goroutine

```
for _, v := range []int{1,2,3} {
   // 这里打印v 会出问题
   // 不是我们想要的
    v := v // 临时变量
    go func() {
  	 fmt.Println(v)
}  
}
```

https://mp.weixin.qq.com/s/OLgsdhXGEMltmjcpTW2ICw 

小知识点: 

函数是高等公民： https://mp.weixin.qq.com/s?__biz=MzAxNzY0NDE3NA==&mid=2247485997&idx=1&sn=e8e966ea60fe337fb9caec61532da332&scene=21#wechat_redirect

```
package main

import "fmt"

func app() func(string) string {
 t := "Hi"
 c := func(b string) string {
  t = t + " " + b
  return t
 }
 return c
}

func main() {
 a := app()
 b := app()
 a("go")
 fmt.Println(b("All"))  // hi All
 fmt.Println(b("All"))  // hi go all 
}

说实话对于这类问题没有深究，只能大致猜到一个原因，这个t 的作用域在同一个闭包中是互通的
```



零值可用：