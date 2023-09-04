```
https://segmentfault.com/a/1190000006823652 
（核心就是defer 的传入参数是 定义函数的时候决定，defer 内部的时候参数 是执行决定）

https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.4.html (这个有个需要注意的是变量的作用域，最近的作用域 ， 比如 for range 中 给 goroutine 添加参数， v:=v 这样) （很简单的讲解）

https://learnku.com/articles/42255。 这篇文章讲解的很全面。

defer 的执行要学会转换

核心

1.return 定义的内容 和  函数内部内容是同一个函数空间 （如果定义了返回参数的名称，则这个变量应用空间是整个函数）。 defer 的参数如果是 外部传入的，变量值维持定义时候的状态，但是他也不会影响外部变量的状态。
2.return 的赋值先执行，再执行defer ， 再执行 函数结束指令 (return 的执行不是原子性)
3. defer 如果是一个方法，那就是匿名方法，匿名方法的变量如果传递进去，就走传递进去的值。 如果是引用外面的，就走实际执行到那会的值。
4. fmt.println() 看做一个func 的执行，参数是传进去的
5. 注意一些shadow 问题，函数内部定义的同名变量的修改，是不会映射到函数外部定义的变量
   
example 

例1：

// 如果defer 函数中的参数是作为参数传进去的， 定义的时候就能确定参数值。defer 直接调用 定义好的参数(比如内部函数)，也是这样的效果
i := 1
defer fmt.Println("Deferred print:", i)  // 1
i++
fmt.Println("Normal print:", i)  // 2

// println 传入了函数
Normal print: 2
Deferred print: 1  


例2：
func main() {
 var whatever [6]struct{}
 for i := range whatever {
  defer func() {
   fmt.Println(i)
  }()
 }
}

输出
5
5
5
5
5
5


   例3：
    func main() {
     var whatever [6]struct{}
     for i := range whatever {
      defer func(i int) {
       fmt.Println(i)
      }(i)
     }
    }
    
    
 输出
5
4
3
2
1
0


 例4：
    func f1() (r int) {
        r = 1
        defer func() {
            r++
            fmt.Println(r) // 3
        }()
        r = 2
        return
    }
    
 func main() {
    f1() // r 的返回值也是3
}   


例5
func f() (r int) {
     t := 5
     defer func() {
       t = t + 5 
     }()
     return t
}

输出：
5


 例6：
    func f() (r int) {
        defer func(r int) {
              r = r + 5  // 局部变量
        }(r)
        return 1
    }  // 1
    
    
例8：

type Person struct {
    age int
}

func main() {
    person := &Person{28}

    // 1. 
    defer fmt.Println(person.age) // 28

    // 2.
    defer func(p *Person) {
        fmt.Println(p.age)  // 引用传递，执行的时候，值被改变。 29
    }(person)  

    // 3.
    defer func() {
        fmt.Println(person.age)  // 29
    }()

    person.age = 29
}


例9：
type Person struct {
    age int
}

func main() {
    person := &Person{28}

    // 1.
    defer fmt.Println(person.age) //28, 这个地方感觉是 age 这个变量被传进去了

    // 2.
    defer func(p *Person) {
        fmt.Println(p.age) // 28, 这个地方28很重要，因为 person 的地址被改变了，之前 p 引用类型传进去的地址值还是 28
    }(person)

    // 3.
    defer func() {
        fmt.Println(person.age) // 29
    }()

    person = &Person{29}
}


例10：

很难的一道题 （fn , map 都是地址类型， slice 也包含地址类型）
func f1(n int) (r int) {
	defer func() {
		r += n  // 4 + 3 = 7
	}()

	var f func()

	defer f()
	f = func() {
		r += 2 // 压根没有执行， 感觉也是因为 f 是被换地址了
		fmt.Println(r, "-==")
	}
	
		return n + 1  // r = 3 + 1 = 4
}

func main() {
	fmt.Println(f1(3))
}

// 感觉地址一般不会变，除非重新赋值了。



例11 ：
   // defer函数参数中是func 的时候，会先执行这个参数 func
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
   
 // 输出 1   
```



```
defer 配合 recover 恢复 panic

有的paninc 不能恢复，比如 throw 的那种，目前了解到的情况只有, map 的并发读写。


recover 只能在 defer 定义的函数中才能生效

defer func() {
   recover()
}



recover 生效的要求比较严格

1.defer recover() // 不能生效

defer func() {
  recover()
}

2.defer 嵌套 recover 不能生效。

3.recover 只能捕获最近的一个panic， panic 会被覆盖。

4. defer 函数不能整体在别的函数中

func dd() {
  defer func() {
     if err := recover(); err != nil {
     
     }
  }
}

// 这样不能捕获，可以理解成执行完 dd() 就退出了。
func main() {
   dd()
   panic(11)
}



recover 不能跨协程，只能在当前协程中捕捉，但是把这个捕捉函数独立出去是没问题的, 如下，就是不能在协程之外捕获

  // 这种方法独立出去是没问题的
  func T1() {
  	if err := recover(); err != nil {
  		fmt.Println("recover: ", err)
  	}
  }
  
  func main() {
   defer T1()
  
  }
  
```



```
func calc(index string, a, b int) int {
  	ret := a + b
  	fmt.Println(index, a, b, ret)
  	return ret
  }
  
  func main() {
  	a := 1
  	b := 2
  	defer calc("1", a, calc("10", a, b)) // 执行到这里的时候需要确定defer 的参数，所以会先执行
  	a = 0
  	defer calc("2", a, calc("20", a, b)) // 确定defer 的参数，先执行
  	b = 1
  }
  
// 输出， 核心就是 defer func 参数传入的时候，要提前算清楚  
10 1 2 3
20 0 2 2
2 0 2 2
1 1 3 4
 
```





