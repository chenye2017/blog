---
title: go-反射
date: 2020-07-02 16:13:01
tags: [go, 反射]
categories: go
---

反射在php 中的应用场景主要是依赖注入的时候，通过控制器在调用函数的时候对于传入参数 class 类名的限制，自动从di容器中取出对应class 的单例类实体，方便我们在使用的时候不需要主动传入。

<!--more-->

go中反射的使用当然包含上述依赖注入的场景，目前我接触到的场景就是struct 中tag 的使用，比如struct 在json string 中 对于json属性名的自动转小写。

```
type Name struct{
   name string
}

a := Name{"111"}

r := reflect.TypeOf(a)  // 返回一个type 类型
r.NumField()  // 返回属性的个数
r.Field(i) // 返回第i 个属性
r.Name() // 获取属性的名称
r.Kind() // 获取属性的类别 ，比如自定义结构体叫myStruct, kind 返回 struct, name 返回myStruct

// 对于地址类型
r := reflect.TypeOf(a).Elem() // 返回一个type类型，我们就能愉快的使用接下来的那些方法了


v := reflect.ValueOf(a)  // 返回一个value 类型 ，（把类型和值分开了，虽然感觉很奇怪，也可以用来包装一个值成为 value 类型）
v.Field(i) 这个返回的类型和上面type 的类型还不一样，上面的那个可以获取tag等属性


// 还有个很重要的就是函数的反射

func Sum(a int, b int) int {
   return a + b
}

p := reflect.ValueOf(Sum)

// 注意call 调用的时候必须是[]value, 然后单个value 可以用ValueOf 来返回
p.Call(reflect.Value{reflect.ValueOf(10), reflect.ValueOf(20)})
```



写的一个类似 json反序列的demo

```
package main

import (
	"encoding/json"
	"fmt"
	"reflect"
)

type R1 struct {
	Name string `redis:"name"`
	Age string `redis:"age"`
}

func main()  {
	m1 := map[string]string{
		"name": "cy",
		"age" : "12",
	}

	r := R1{}

	Unmar(&r, m1)
}

func Unmar(o interface{}, m map[string]string) {
	v := reflect.ValueOf(o)
	t := reflect.TypeOf(o)

	v = v.Elem()  // 这个地方需要注意的就是下面的这些方法直接用上面指针类型调用不通，原因我也没太深究
	t = t.Elem()
	filedNum := v.NumField()

	for i:= 0; i< filedNum; i++ {
		tagname := t.Field(i).Tag.Get("redis")
		//fmt.Println(tagname, "====")
		v.FieldByName("Name").SetString(m[tagname])
		// v.Field(i).SetString(m[tagname])
	}

	s, _ := json.Marshal(o)
	fmt.Println(string(s))
}

```



go-redis 中有大量的源码是关于interface 转成我们需要的类型，下次如果es 或者其他nosql 中有这种需求，可以借鉴下

今天在看写 

```
switch  t := a.(type)

case int:
	如果进入了这个case， 我们可以直接把t 当做 int 类型使用，真的是太方便了
```



