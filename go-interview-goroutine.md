```
for  range 执行 goroutine, 变量问题 （这个应该各个语言都有，闭包变量作用域问题）（黑金宝盒问题类似，闭包中使用的变量是外部改变的变量，而没有用传入的方法，类似
defer func() {} 的执行，

但之前有个同事 defer log.Print("%+v", res). 本想着打印最后的结果集，结果因为参数开始执行的时候已经确定了。并不会真的在打印出最终的res
  
defer f(). 这个 f 是初始传进去的样子，后期再修改这个 f 并不会改变 defer 的执行
```



```
 For 循环


  案例一：
  
  var all []*Item
  for _, item := range items {
   all = append(all, &item)
  }
  
  // 存储的都是item 这个变量的地址，也就是 items 最后一个变量
  // 如果是普通变量，值传递那种，就直接给过去了
  
  案例二:
  
  var prints []func()
  for _, v := range []int{1, 2, 3} {
   prints = append(prints, func() { fmt.Println(v) })
  }
  for _, print := range prints {
   print()
  }
  
  // 匿名函数，变量还不是传进去的，最后打印的是 v最后的值
  // 解决方案一是作为参数传进去
  // 官方的解决方案是 v := v， 生成一个新的临时变量
  
```



```
 下面这代码是真的恶心，就是为了考验 for range 的时候 v := v, 作用域

  
  type query func(string) string
  
  func exec(name string, vs ...query) string {
  	ch := make(chan string)
  	fn := func(i int) {
  		ch <- vs[i](name)
  	}
  	for i, _ := range vs {
  		go fn(i)
  	}
  	return <-ch
  }
  
  func main() {
  	ret := exec("111", func(n string) string {
  		return n + "func1"
  	}, func(n string) string {
  		return n + "func2"
  	}, func(n string) string {
  		return n + "func3"
  	}, func(n string) string {
  		return n + "func4"
  	})
  	fmt.Println(ret)
  }
  
  // 最后输出
  // 可能 111func1 或者 111func2 或者 111func3 或者 111func4
  //  最后只能输出一个， 所以不确定是第几个协程输出，也不能确定此时n 遍历到第几个了
```







```
米哈游面试题 （https://learnku.com/go/t/60933）


m := make(map[int]int, 10)
for i := 1; i<= 10; i++ {
    m[i] = i
}

for k, v := range(m) {
    go func() {
        fmt.Println("k ->", k, "v ->", v)
    }()
}

核心问题就是 k v 引用了外部变量，需要用  k := k, v := v 替换。

解决方法一
for k, v := range(m) {
    k := k
    v := v
    go func() {
        fmt.Println("k ->", k, "v ->", v)
    }()
}

// 上面的k, v 是个迭代的临时变量


解决方法二：
for k, v := range(m) {
    go func(k, v int) {
        fmt.Println("k ->", k, "v ->", v)
    }(k, v )
}

```



```
for range 遍历数组的时候，产生的是同一个临时变量， 

for _, v := range arr {
 
}

v 对应的是同一个变量的地址。

所以

var all []*Item
for _, item := range items {
 all = append(all, &item)
}
此时 输出的all 中所有变量对应 item 数组中最后一个元素

上面的例子验证了，即使不用协程，但凡用到了临时变量的地址，都会遇到问题。



改造方案：

var all []*Item
for _, item := range items {
 item := item
 all = append(all, &item)
}


又或者
var prints []func()
for _, v := range []int{1, 2, 3} {
 prints = append(prints, func() { fmt.Println(v) })
}
for _, print := range prints {
 print()
}

改造方案：

for _, v := range []int{1, 2, 3} {
  v := v
  prints = append(prints, func() { fmt.Println(v) })
 }


```



```
如何并发100个任务，但是同一时间最多运行的10个任务（waitgroup + channel）
errgroup ，设置最大协程数，for range  通过channel 拿任务。


func main() {

	ch := make(chan struct{}, 10)

	for i := 0; i < 10; i++ {
		ch <- struct{}{}
	}

	wg := sync.WaitGroup{}

	for i := 0; i < 100; i++ {
		wg.Add(1)
		select {
		case t := <-ch:
			go func() {
				defer func() {
					wg.Done()
				}()
				fmt.Println(runtime.NumGoroutine(), "----")
				ch <- t
			}()
		}
	}

	wg.Wait()
	fmt.Println("---success")
}

```





```
30 个并发打印 0-99

ch := make(chan int)
	go func() {
		for i := 0; i < 30; i++ {
			ch <- i
		}
	}()

	wg := sync.WaitGroup{}
	for i := 0; i < 30; i++ {
	wg.Add(1) // 这个千万别写到 go func 里面，这样wait 会停不下来
		go func() {
			defer wg.Done()
			fmt.Println(<-ch)
		}()
	}

	wg.Wait()
```



```
怎么查看goroutine 的数量

 
  runtime 里面有个变量
  
  goroutine id ， 黑科技，go 官方不希望给出来。（不希望 goroutine 级别的信息存储，不容易回收。goroutine 数据不能主动回收 和 结束， 是通过 gc 来进行回收）
```



```
 怎么 借用外力 主动停止  一直执行的 goroutine 


  https://mp.weixin.qq.com/s/tN8Q1GRmphZyAuaHrkYFEg
  
  goroutine如果本身在主动执行某个任务是不能主动停止的，煎鱼上面文章的介绍主要是场景：轮训的定时任务，怎么通知这个协程去主动结束
  
  1. select 多通道，有个关闭通道， 关闭之后能读取到内容。 （
     1. ctx.done() 是一种类型，只读channel， close 之后不在阻塞。 
     2. 普通有数据channel， data, ok := <- c, ok = false 关闭
     
  2. for range 通道，关闭通道之后不再阻塞。
 
```

