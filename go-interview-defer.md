关于 defer 的总结

### 如果defer 函数中的参数是作为参数传进去的， 定义的时候就能确定参数值。defer 直接调用 定义好的参数(比如内部函数)，也是这样的效果

```

func test() {
   t := 1
   defer func(i int) {
     fmt.println(i)
   }(t)
   t = 2
}
// 输出 1


func test(){
  i:=0
  defer fmt.println(i) // 输出 0
  i = 1
}
```

### 如果是defer body 依赖外部的参数，那defer 最后真正执行的时候才能确定参数值。

这种情况就等同于闭包，只有执行的时候才能确定值。

```
func test() {
  t := 1
  defer func() {
     fmt.println(t)
   }()
   t = 2
}
// 输出 2
```

**题外话： 闭包这样用函数外的变量一定要注意，以下几个场景。 造成下面的场景的核心都是闭包的执行用的是真正执行时候的那个时候变量的值**

```
for k := range []int{1,2} {
  go func () {
     fmt.println(k)
  }
}
// k 的输出不满足预期，因为协程的执行不是顺序按照 for range 的顺序

var func1 xxx (某种类型)
var func2 xxx
for k := range []int{1,2} {
   if k == 1 {
      func1 = func() {
          fmt.println(k)
      }
   }
   if k == 2 {
      func1 = func() {
         fmt.println(k)
      }
   }
}

// 执行出来都是 k= 1 这个index
func1()
func2()
```

### 多个defer的执行顺序是栈，先进后出。还有必须代表执行到这个defer， 这个defer 才能被加载到栈里面，否则defer 不被执行

```
package main

import "fmt"

func main() {
	defer func() {
		fmt.Println(2)
	}()
	defer func() {
		if err := recover(); err != nil {
			fmt.Println(err)
		}

	}()

	panic("---11")
	fmt.Println(3) // 执行不到
}

// 输出
---11
2
遇到panic， 就会依次执行defer，然后顺序退出
```

### 关于defer 和 return 的执行顺序。1.是先执行return 函数，如果return 后面有指令的话，2.然后执行 defer 3.最后函数结束，也就是 return。 这个经常会结合 go 函数入参 返回值参数申明一起考察。

```
func test(i int) (r int) {
 defer func() {
    r =  r + 3
 }()
  return 4
}

// 这个return 4 可以拆解如下
r = 4
return 
// 为啥是 r = 4,而不是 r:= 4 , 因为函数定义的时候有返回值定义，类似 var r 
// 所以上面的函数最后返回的是 7

func test1(i int) (r int) {
	defer func() {
		r := r + 3
		fmt.Println(r)
	}()
	return 4
}

// 但如果是上面这种，最后函数输出是4，就是因为 r 被重新定义了，只有函数内部能读取到。上面这种情况经常在 return err 的时候遇到，会有shadow err 的嫌疑

func test2() (err error) {
	if _, err := strconv.Atoi("2ss"); err != nil {
		return
	}
	return nil
}

 // err is shadowed during return
 // 这个 err 只能在 if 作用域内被使用
```

### defer函数参数中是func 的时候，会先执行这个参数 func

```
func test3() int {
	i := 1
	defer func(b int) {
		fmt.Println(b)
	}(func() int {
		return i
	}())

	i = 4
	return i
}
```

