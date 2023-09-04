```
内存泄露， 什么情况下回内存泄露 ？

很出名的一个例子，time.After 时间间隔比较久。

之前同事写了一个例子 


tim := time.NewTimer(3 * time.Second)
res := make(chan interface{})

go func {
   time.Sleep(4 * time.Second)
   res <- interface{}
}

select {
case <-tim: 
    do()
case tmp := <-res:
    do()
}
结果没了 case2， res 一直阻塞状态， go 协程没办法塞入数据，导致go 协程一直存在，内存泄露。



自己也写过一个例子
goArr := make(chan struct{}, 5) // 控制并发数

go func {
  for _, v := range funcArr {
     goArr <- v
  }
}

for _, v := range goArr {
   v := v
   go func() {
      v()
   }
}

没有close goArr, 导致 goArr 读取完数据一直在阻塞， 所以协程一直没有释放。



go time after 协程泄露

res := make(chan int, 1)
res <- 10
select {
 case time.After(3 * time.Second):
    // timer 不能及时被回收
 case <-res:
    fmt.Println("--test--")
}

这个 time.After 其实严格意义上不能算是泄露，只是没有及时关闭。



更可怕的是 for 循环这种
  
  for {
   select {
    case <- res.Chan:
    // work
    case <- time.After(time.Second)
    // do
   }
  }
  
  上面每次time.After会创建一个新的定时器。
  
  怎么解决呢 ？
  
  1.time.NewTimer, 主动释放
  
  c := time.NewTimer(time.Second)
  defer  c.Stop()
  
  select {
    case <- res.Chan:
    // work
    case <- c.C:
    // do
   }
  
  
  
  c := time.NewTimer(time.Second)
  for {
    c.Reset(time.Second) // 主动重置，而不用重新生成
   select {
    case <- res.Chan:
    // work
    case <- c.C
    // do
   }
  }


http 请求的时候 cancel() 的执行，也是为了能及时释放资源。

```



```
  阿b下面的errgroup 更好。协程数量控制死。 通过 channel 取任务
  
  用的 b站封装的 errgroup 函数，有个设置groutine 数量的，GOMAXPROCES。用了这个方法之后，当我们添加 func 的时候，他是塞入 channel 中， 先通过 for 启动固定数量的协程，然后在协程里面通过 for range 去读取对应的方法
  
  ctx := context.Background()
  	res := make(chan func(ctx context.Context) error)
  	for i:= 0; i<10; i++ {
  		go func() {
  			for fun := range res {
  				_ = fun(ctx)
  			}
  		}()
  	}
  
  使用的是 缓冲channel。 
  
  当时没有用他的wait 方法。 他的wait 方法会去close 这个 添加方法的chan
  
  导致 没能关闭这个缓冲channel 。 又因为他的协程中有个 for range channel 的操作， 导致一直for range 阻塞， groutine 得不到释放。
  
```





```
Goroutine 数量控制在多少合适，会影响 GC 和调度

gmp 

p  processor 可以设置，一般和系统核数一样。 gomaxprocs 

m machine, 默认设置 10000， 超过会报错。 GO: runtime: program exceeds 10000-thread limit

g  初始 2k ~ 4k 大小， 所以手内存大小限制。


go working steal 先从 global 队列取，再从其他 本地队列取。


Goroutine 阻塞的话，是不是对应的M也会阻塞?
不一定。
```



```
其实goroutine id是存在的

只是大家不建议使用

goroutine 不能更好的控制和回收
```



```
GMP 模型，为什要有p
1.之前没有p 的时候，都是全局队列上取 goroutine， 锁竞争激烈
2.如果把 g直接挂在m 上， m 会因为 系统调用暂时陷入睡眠状态，这时候会生成新的 m 去执行g 。这时候我们不希望 g 的处理会被阻塞，所以需要把老的 g队列转移到新的 m的g队列上，麻烦
3.g 队列的数量不会随着 m的增加而增加。
```



```
var nums1 []interface{}
 nums2 := []int{1, 3, 4}
 num3 := append(nums1, nums2)
 fmt.Println(len(num3))
 
 
 输出：
 1
 
 这题我感觉并不是很难，主要是 append 最后的元素并没有拆解。
```



```
面向对象 (go 的实现方式)


1.封装。 （go 首字母大写，公共。首字母小写，私有）
2.继承。 （字段嵌套）
3.多态。 （接口类型）
```



```
值传递 和 引用传递 的区别

这种题目感觉对于写了这几年的我来说，感觉不应该记录了，hhh， 但怕蒙了。

我们平时工作中对于结构体类型，大部分传的引用，主要担心互相copy ，消耗内存大。 int 这种类型一般传递的还是值，然后通过返回值来修改。
```



```
进程，线程 和 协程的区别

  https://zhuanlan.zhihu.com/p/70256971
  
  1.进程是程序运行的实体，资源分配的最小单位 (比如限定某个程序最大多少内存)（qq 和 网易云同时 运行）
  进程拥有自己的资源空间，一个进程包含多个线程，多个线程共享同一个进程的资源
  
  2.线程类似于程序中的多个任务，cpu 独立运行和独立调度的最小单位， 内核态 （网易云 播放歌曲 和 歌词滚动同时运行）
  cpu 上跑的任务是线程
  
  3.协程 ,用户态线程，占用内存小，kb，可以自动扩容，  线程栈 几mb, 线程是cpu 调度， goroutine 是 go runtime 进行调度
  对于cpu 来说，他是不知道协程的存在的。
  
  4.协程进行切换代价比线程小。
  
  为什么协程切换代价比线程切换代价小？
  1.协程间任务的切换发生在用户态，程序控制，资源消耗小。线程的切换发生在内核态，所以需要用户态到内核态的切换， 再内核态到用户态切换。
  2、而且对于多核情况下，如果协程的切换都发生在一个cpu 上执行，线程的切换在多个cpu上执行， 消耗大。（协程间不需要加锁，因为本质上协程是在一个线程中切换的）https://www.v2ex.com/t/387596​， 协程不用加锁的解释，少场景 （这是以前刚写go 的时候内心os, 协程不用加锁， 但不代表他执行的 内容不用加锁）
  3.协程的切换只需要把 cpu 的寄存器上内容切换到上次执行的地方，线程包含的资源更多，比如协程切换不用管线程共享的栈内存，但线程间切换就得管。
```



```
单核cpu， 开两个goroutine， 其中一个死循环会怎么样 

1.14版本以下会hang住

包含以及之后的版本会正常，出现了基于信号的抢占式调度。


runtime.sysmon 做的检测的事
1. sysmon线程是无限循环执行。
2. sysmon线程在每个循环中，会进行netpool（获取fd事件）、retake（抢占）、force gc（按时间强制执行gc），scavenge heap（释放自由列表中多余的项减少内存占用）等处理。
3. sysmon线程一开始每次循环后，休眠 20us，50次之后（即1ms后）每次休眠时间倍增，最终每一轮都会休眠 10ms。

sysmon 发信号给 m， m收到信号休眠正在阻塞的goroutine， 调用绑定的信号方法，并进行重新调度。
```



```
panic机制 ？ 

1.只能捕获当前协程的panic， 

2.recover 只能在func 中执行，靠栈存储，先进后执行， 可以参考小米的那篇文章原理。
```





```
Context 平时什么场景使用到？
基本上每个函数都用到。比如 http接口中，例如 gin ，把 req 封装在了 context 中，还有 context 中通过中间件注册了用户uid 这些，从里面拿。


context.WithTimeout 使用现象和实现？
withTimeout 记得没错的话，应该是换算成 withdeadline 方法。
实现了 canceler 接口，通过定时器实现。
会通过 map 存储子context 和 对应的cancel 方法，如果当前方法取消，会遍历的去调用 子的cancel。
并且把自己从父 cancel map 中删除 （如果父是cancel 类型 context 的话）


如果想缩短超时时间。比如总的服务超时250ms， 依赖的其中某个服务只能给50ms。
里面会判断传入的超时是否超过当前已存在的超时，如果是延长，直接不处理返回。




context.WithTimeout 还有 cancel 返回值，如果不去调用是否有问题？
不会有问题，如果想提前cancel，可以调用，如果不调用，到点了函数内部会自己调用。
```



```
 
 package main
     import "fmt"
     const (
         a = iota
         b = iota
     )
     const (
         name = "menglu"
         c    = iota // 这个单纯记录const 里面的行数
         d    = iota
         e    = "tt"
         f    // 延续上一个变量
         g    = iota
     )
     func main() {
         fmt.Println(a) // 0
         fmt.Println(b) // 1
         fmt.Println(c)  // 1
         fmt.Println(d)  // 2
         fmt.Println(f)  // "tt"
         fmt.Println(g) // 5
     }
     
     iota 在一个const 片段中会重置
     
     iota 就当做在 const 中的行数
```



```
const (
	a  = 1
	b  = iota  // 1
	c1 = 5
	d1 = iota  // 3
)
```



```
 // const 常用容易遇到的问题
 const (
      x uint16 = 120
      y
      s = "abc"
      z
      w = iota
      ww
  )
  
  fmt.Println(y,z, w, ww)
  
  // 120  abc  4 5
  // iota 才会递增， 别的不会
  // const 里面除了记录iota 还可以用常量 数值
```



```
  常量和 函数参数未使用，在 go 中是可以被允许的
  
  我们代码中经常发现 const 可能是黑的，没有被使用，或者 func
  
  但引入的包没有被使用
  
  变量没有被使用都是不行的
```



```
  go 中 i++ 是语句，不能直接用来进行赋值
  
  c[i++]
  //  不能这样使用，得分成 2步，如下
  i++
  c[i]
  
  
  
 


  A.
  i := 1
  i++
  
  B.
  i := 1
  j = i++
  
  C.
  i := 1
  ++i
  
  D.
  i := 1
  i--
  
  AD 是正确的，go 中不存在 ++i
  
  i++ 和 i-- 在 Go 语言中是语句，不是表达式，因此不能赋值给另外的变量。
```



```
别名


  type User struct{}
  type User1 User
  type User2 = User
  
  func (i User) m1() {
      fmt.Println("m1")
  }
  func (i User) m2() {
      fmt.Println("m2")
  }
  
  func main() {
      var i1 User1
      var i2 User2
      i1.m1()  // 不能被执行
      i2.m2()
  }
  
  能，输出m1 m2，第 2 行代码基于类型 User 创建了新类型 User1，第 3 行代码是创建了 User 的类型别名 User2，注意使用 = 定义类型别名。因为 User2 是别名，完全等价于 User，所以 User2 具有 User 所有的方法。但是 i1.m2() 是不能执行的，因为 User1 没有定义该方法。
```



```
* for range 中容易遇到的

  func main() {
      x := []string{"a", "b", "c"}
      for v := range x {
          fmt.Print(v)
      }
  }
  
// 注意for range 第一个元素是key， 第二个元素是value

```



```
 断言和 switch type 只能用在 interface身上

  ```
  A.
  type MyInt int
  var i int = 1
  var j MyInt = i
  
  B.
  type MyInt int
  var i int = 1
  var j MyInt = (MyInt)i  // 前面这种写法，会直接报错， MyInt 不是表达式， (MyInt)(i) 经常这样修改
  
  C.
  type MyInt int
  var i int = 1
  var j MyInt = MyInt(i)
  
  D.
  type MyInt int
  var i int = 1
  var j MyInt = i.(MyInt)
  
  
  c是对的。
```



```
* 感觉很沙雕的题目

  
  func main() {
      i := 1
      s := []string{"A", "B", "C"}
      i, s[i-1] = 2, "Z"
      fmt.Printf("s: %v \n", s)
  }
  
  计算等号左边的索引表达式和取址表达式，接着计算等号右边的表达式；
  赋值；
  所以本例，会先计算 s[i-1]，等号右边是两个表达式是常量，所以赋值运算等同于 i, s[0] = 2, "Z"。
  
  // 输出
  // s: [Z,B,C]

```



```
go 怎么设计一个连接池 （https://learnku.com/articles/41137）

1.最大链接数量
2.最大空闲链接数 （db 中主要用来判断释放的链接是放到 free 中，还是close）
3.任务队列
4.链接的健康

go 标准库sql

new pool ， open 并没有创建链接，只是启动了一个协程， 监听 mysql 请求，来一个请求开一个获取一个链接。

空闲链接存放数组中

1.判断数组 freeConn 中是否有空闲链接，如果有， 拿出 0号元素。 通过 freeConn[1:] 去覆盖 freeConn
2.如果数组 freeConn 中没有空闲元素， 但是当前 freeConn 小于 max 链接数，就会new 一个新的链接
3.如果数组 freeConn 中没有空闲元素， 且当前 freeConn 大于 max 链接数，就会通过产生一个唯一 key， 然后用map[唯一key]channel 存储。 通过 select 监听多个 1.当前channel， 2. ctx.Done(), 3. 默认最大超时 。 当有链接数归还的时候 for k, v := range map 随机出一个 等待链接的请求，把链接塞到 channel 中。

拿到链接后要检查
1.链接的使用时间



go-redis 

new pool, 同时启动一个协程，维护最低 idle 链接数的pool, 和 db 中有些不一样 。

通过一个 queue 用来发放令牌，只有通过这个queue 拿到令牌，才能去获取 conn (感觉这个queue 就是任务队列)

1.判断数组 freeConn 中是否有空闲链接，如果有， 如果是 fifo 拿出 0号元素。 通过 freeConn[1:] 去覆盖 freeConn. 否则末尾出链接。

2.如果数组 freeConn 中没有空闲元素， 就会new 一个新的链接， 因为已经通过 令牌拿到了许可证，通过令牌数量限制最大链接数

```





```
 Go happen before 。（其实工作中蛮常见，自己不太注意）

 
  var c = make(chan int, 10)
  var a string
  
  func f() {
   a = "炸煎鱼"   (1)
   c <- 0        (2)
  }
  
  func main() {
   go f()
   <-c           (3)
   print(a)      (4)
  }
  
  // happen before 原则， c的写入开始在 c 输出完成之前， c的写入完成在 c的开始接受之后
  // 所以把 c 感觉可以当作一个串行过程
  // 炸煎鱼
  
  
  
  var c = make(chan int)
  var a string
  
  func f() {
   a = "煎鱼进脑子了"    (1)
   <-c                 (2)
  }
  
  func main() {
   go f()
   c <- 0              (3)
   print(a)            (4)
  }
  
  // 煎鱼进脑子了
  
```



```
go什么时候 内存逃逸从栈上逃逸到堆上 
  
  1.地址类型的返回，（野指针, 闭包函数， map的返回）
  2.大的内存占用 （slice len很大，大于栈的内存空间）
  3.interface类型 （fmt， 编译期间不知道具体的数据类型）

  gc 回收的是堆上的对象。
```



```
首先明确一点：nil是值而非类型。nil值只能赋值给slice、map、chan、interface和指针。

在Go中，任何类型都会有一个初始值。数值类型的初始值为0，slice、map、chan、interface和指针类型的初始值为nil，对于nil值的变量，我们可以简化理解为初始状态变量。

var err error
e := &err
if e != nil {
    fmt.Printf("&err is not nil:%p\n", e)
}
// 输出：&err is not nil:0xc0000301f0

err是一个接口类型的变量，其初始值为nil，然后对err进行取址操作会发现能成功取到地址，这就是Go和C++最大的不同之一。有C++基础的人在刚接触Go的时候，自然而然的会认为nil是个空指针类型值，上面的代码力证在Go中，nil只是一个表示初始状态的值。

```



```
go 中保证数据并发安全的方法

1. mutex

2. atomic

3. channel
```



```
无意间看到 string 并发的坑 (string 底层也是一个结构体， len 和 data 不一致)


https://segmentfault.com/a/1190000023283854
  
万物都是线程不安全的。
  
线程安全的 channel， sync.Map (mutex 护航)
atomic.Value （atomic 实现）
```



```
 线程安全 ： channel， 就是用来协程间同步数据的。
  
 线程不安全：
 string ， 不可修改， 防止 copy on write （上面有个例子，讲的是string 又读又写出现的问题）
 int， float， 都不行 所以出了 atomic 包， 需要我们去自行处理。
 map, slice, 这些 在读写并发时候会有问题， 
 struct 对于相同field 并发读写会有问题，不同field 并发读写没问题.(因为不同的field 存在于不同的内存地址)
 map 有个好处是并发会panic.
```







```
 关于 zero-base

  type People struct {}
  
  func main() {
   a := &People{}
   b := &People{}
   fmt.Println(a == b) // false
  }
  
  
  type People struct {}
  
  func main() {
   a := &People{}
   b := &People{}
   fmt.Printf("%p\n", a)
   fmt.Printf("%p\n", b)
   fmt.Println(a == b) // true, 从栈上逃逸到堆上
  }
  
  
  
  为什么逃逸后相等。
  runtime
  
  var zerobase uintptr  // 每块二进制代码并不一样
  
  变量 zerobase 是所有 0 字节分配的基础地址。更进一步来讲，就是空（0字节）的在进行了逃逸分析后，往堆分配的都会指向 zerobase 这一个地址。
  所以空 struct 在逃逸后本质上指向了 zerobase，其两者比较就是相等的，返回了 true。
  
  为什么没逃逸是相等的。
  
  在没逃逸的场景下，两个空 struct 的比较动作，你以为是真的在比较。实际上已经在代码优化阶段被直接优化掉，转为了 false。
```



```
struct 是能比较的，但如果包含不能比较的元素，就不能比较了

  https://juejin.cn/post/6881912621616857102
  // 这篇文章讲的很好。
  // 注意结构体的field 顺序得一致，要不然也不一样，感觉像是内存对齐的问题
  // 注意 struct 的类型一定要一致呀，但是匿名结构体的比较，只考虑字段
  // 结构体比较所有field 都得能比较，如果包含map 这种不能比较的会报错。
  
  // 下面这种就没法比较大小
  // 下面这种编译都没法通过，需要强制转换成相同类型
  package main
  
  import "fmt"
  
  func main() {
      type A struct {
          a int
      }
      type B struct {
          a int
      }
      a := A{1}
      //b := A{1}
      b := B{1}
      if a == b {
          fmt.Println("a == b")
      }else{
          fmt.Println("a != b")
      }
  } 
  
```



```
 go 中类似 php的语法

 
  type People struct{}
  
  func (p *People) ShowA() {
  	fmt.Println("showA")
  	p.ShowB()
  }
  func (p *People) ShowB() {
  	fmt.Println("showB")
  }
  
  type Teacher struct {
  	People
  }
  
  func (t *Teacher) ShowB() {
  	fmt.Println("teacher showB")
  }
  
  func main() {
  	t := Teacher{}
  	t.ShowA()
  }
  
  输出结果为showA、showB。golang 语言中没有继承概念，只有组合，也没有虚方法，更没有重载。因此，*Teacher 的 ShowB 不会覆写被组合的 People 的方法。
```



```
并发和并行的区别
 
并行是真正意义上的同时执行
  
并发 本质上是轮流执行，速度太快，外界看上去是同时执行 (时间片的切换。轮流执行)
```



```
Golang swith 和 php switch 区别


1.go中加上了默认break，匹配到对应case，在执行完相应代码后就会退出整个,switch 代码块
 2.go中用fallthrough关键字继续执行后续分支的代码
 3.go 中 switch  type 可以用于获取字段类型 (只能是interface 类型)
  
```



```
* 为 sync.waitgroup 中wait 函数 支持 wait timeout 功能


  首先 sync.WaitGroup 对象的 Wait 函数本身是阻塞的，同时，超时用到的time.Timer 对象也需要阻塞的读。
  
  同时阻塞的两个对象肯定要每个启动一个协程,每个协程去处理一个阻塞，难点在于怎么知道哪个阻塞先完成。
  
  目前我用的方式是声明一个没有缓冲的chan，谁先完成谁优先向管道中写入数据。
  
  
  func WaitTimeout(wg *sync.WaitGroup, timeout time.Duration) bool {
  	// 要求手写代码
  	// 要求sync.WaitGroup支持timeout功能
  	// 如果timeout到了超时时间返回true
  	// 如果WaitGroup自然结束返回false
  	ch := make(chan bool, 1)
  
  	go time.AfterFunc(timeout, func() {
  		ch <- true
  	})
  
  	go func() {
  		wg.Wait()
  		ch <- false
  	}()
  	
  	return <- ch
  }

// 上面这样写也行，
// 我想到的就是给 wait 还是主协程，然后给给各个协程方法增加超时

func main() {
	wg := sync.WaitGroup{}

	ctx, _ := context.WithTimeout(context.Background(), time.Second*4)

	wg.Add(2)

	now := time.Now()
	go func() {
		defer func() {
			wg.Done()
		}()

		resChan := make(chan struct{})
		go func() {
			time.Sleep(time.Second * 2)
			close(resChan)
		}()

		select {
		case <-ctx.Done():
			return
		case <-resChan:
			return
		}
	}()

	go func() {
		defer func() {
			wg.Done()
		}()

		resChan := make(chan struct{})
		go func() {
			time.Sleep(time.Second * 2)
			close(resChan)
		}()

		select {
		case <-ctx.Done():
			return
		case <-resChan:
			return
		}
	}()

	wg.Wait()

	fmt.Println(time.Now().Unix()-now.Unix(), "---")
}
```





```
  // 编译错误
  // 日常写代码遇到的问题
  
   func (i int) PrintInt ()  {
      fmt.Println(i)
  }
  
  func main() {
      var i int = 1
      i.PrintInt()
  }
  
  基于类型创建的方法必须定义在同一个包内，上面的代码基于 int 类型创建了 PrintInt() 方法，由于 int 类型和方法 PrintInt() 定义在不同的包内，所以编译出错。
  
  解决的办法可以定义一种新的类型：
  
  type Myint int
  
  func (i Myint) PrintInt ()  {
      fmt.Println(i)
  }
  
  func main() {
      var i Myint = 1
      i.PrintInt()
  }
  
  // type Myint = int ,这是两种相同的类型
```



```
单核cpu. 一个协程 死循环，另一个会怎么样


  核心就是 1.14 之后，go 的调度策略有改变，
  
  从以前协作式，增加了抢占式调度 (协程一般都是协作式调度，go 中的协程是抢占式调度)。
  
  runtime.sysmon会检测：
  
  1.抢占阻塞在系统调用上的 P 。
  2.抢占运行时间过长的 G。
  
  发送信号给 M。M 收到信号后将会休眠正在阻塞的 Goroutine，调用绑定的信号方法，并进行重新调度
  
```



```
 sync.Pool 的使用场景


  sync.Pool 本身是并发安全的.但是New 方法的并发安全性需要自己保证。
  
  
  bufferPool := &sync.Pool{
          New: createBuffer,
      }
  
  func createBuffer() interface{} {
      // 这里要注意下，非常重要的一点。这里必须使用原子加，不然有并发问题；
      // 很少在 new 方法里面会遇到 线程不安全
      atomic.AddInt32(&numCalcsCreated, 1)
      buffer := make([]byte, 1024)
      return &buffer
  }
  
 
```



````
 go 怎么实现 协程一个出错，其他任务都被取消

```
package main

import (
	"context"
	"errors"
	"fmt"
	"sync"
	"time"
)

type ErrGroup struct {
	wg sync.WaitGroup
	ctx context.Context
	cancel context.CancelFunc
}

func DefaultErrGroup(ctx context.Context) *ErrGroup {
	ctx, cancel := context.WithCancel(ctx)
	return &ErrGroup{
		wg:  sync.WaitGroup{},
		ctx: ctx,
		cancel: cancel,
	}
}

func (eg *ErrGroup)Do(f func(ctx context.Context) error)  {
	eg.wg.Add(1)
	go func() {

		defer func() {
			eg.wg.Done()
		}()

		res := make(chan struct{})
		go func() {
			if err := f(eg.ctx); err != nil {
				eg.cancel()
			}
			close(res)
		}()
		
		// 阻塞不一定需要用wait 这种信号量
		// 也可以用 channel
		select {
		case <-eg.ctx.Done():
		case <-res:
		}
	}()
}

func (eg *ErrGroup)Wait()  {
	eg.wg.Wait()
}

func main()  {
	eg := DefaultErrGroup(context.Background())

	eg.Do(func(ctx context.Context) error {
		return  Print()
	})

	eg.Do(func(ctx context.Context) error {
		time.Sleep(time.Second)
		fmt.Println("---fail")
		return nil
	})

	eg.Wait()
	fmt.Println("--success")
}

func Print()  error {
	return errors.New("11")
}

````



```
  内存逃逸分析
  煎鱼大佬从另一个角度的分析
  
  1. 指针 （并不是指针都会逃逸，而是指针才当前作用域内使用，就不会逃逸。所以值类型不会逃逸，因为是copy， 而且会copy on write）（函数返回野指针）
  2. 未确定类型（所以interface 会存在内存逃逸，所以fmt 这种存在内存逃逸）
  3. 泄露参数 （如果指针是从外部传进来，再传出去，就不会存在内存逃逸）
  4. 数据过大
```



```
 GMP 模型


  G groutine (占用2kb起步，可以自动扩容，数量受到内存限制)
  m machine  （执行线程，默认 10000， 内核基本不能达到）
  p process  （逻辑处理器，现在设置都和 机器内核一致）
  
  
  p 出现的原因是。 
  之前没有p, 只有g 和 m 的时候，锁竞争激烈。
  g 的生成，放回都会在global g queue 上产生竞争。
  产生了 local g queue, 但这个如果挂在 m 上，因为m 会随着syscall 调用而阻塞。
  这时候我们不希望g 的处理也被阻塞，如果产生新的 local g queue或者 绑定到新的的m 上，都会伴随着 g queue的管理更加复杂，所以抽象出了 p.
  
  
  
  g m, 是 m:n 模型
  
  （1:1 模型，最简单，但没法实现并发）
   (n:1 模型，协程阻塞，导致工作线程就阻塞)
   (m:n 模型，解决了上述问题)
 
 
 local q queue 出现的一个原因
 还有就是cpu 执行的亲和性，如果是当前 cpu产生的 g， 在localqueue 没有满的情况下回放在当前local queue.
 
 
把内核态的线程管理，转移到 用户态 runtime 的调度。


如果本地队列g 执行完了， 会去全局global g 里面取 （全局剩余g 的一半）， 没有再去 别的 g queue里面取 （偷一半别人的）。
p 找m， 会优先从休眠的m 里面找，没有的话会新生成一个M。
m 如果取不到 g， 会自旋，最大的自旋个数是 p 的数量。


m 的阻塞现在已知的只有 syscall ，比如读取磁盘，生成新的线程，
像 channel 阻塞， 网络阻塞，都不会导致 m的阻塞了。


g的切换直接在用户态
```



```
csp 是个啥


 以通信的方式来共享内存。
 用于描述两个独立的并发实体通过共享的通讯 channel(管道)进行通信的并发模型
  

```







```
go 的init 用过么


  1.会在导入包的时候执行，执行在main 函数之前。
  2.每个包可以有多个init， 但顺序不一定。
  3.执行顺序在表达式之后
  
  
  某个go package 文件执行顺序
  
  1.var 定义的变量或者表达式
  
  2.init 方法
  
  3.main 方法
  
  init（不能被调用）
  
  看了下这个执行顺序，所以我们就会明白一些 var 变量的定义不一定非得按照顺序
  
  https://zhuanlan.zhihu.com/p/34211611
  
  init() 函数是 Go 程序初始化的一部分。Go 程序初始化先于 main 函数，由 runtime 初始化每个导入的包，初始化顺序不是按照从上到下的导入顺序，而是按照解析的依赖关系，没有依赖的包最先初始化。
  
  每个包首先初始化包作用域的常量和变量（常量优先于变量），然后执行包的 init() 函数。同一个包，甚至是同一个源文件可以有多个 init() 函数。init() 函数没有入参和返回值，不能被其他函数调用，同一个包内多个 init() 函数的执行顺序不作保证。
  
  一句话总结： import –> const –> var –> init() –> main()
  
  示例：
  
  package main
  
  import "fmt"
  
  func init()  {
  	fmt.Println("init1:", a)
  }
  
  func init()  {
  	fmt.Println("init2:", a)
  }
  
  var a = 10
  const b = 100
  
  func main() {
  	fmt.Println("main:", a)
  }
  // 执行结果
  // init1: 10
  // init2: 10
  // main: 10
  
```



```
go 怎么调度 goroutine

  
  gmp 调度模型。
  
  最开始是 gm 模型，从global goroutine 队列中拿 goroutine， 抢锁激烈。
  
  而且生成新的goroutine 后放到全局队列，等待cpu执行，失去了cpu 自身的亲和性。
  
  所以出现了 local goroutine queue，如果当前 g 生成新的g ， 如果本地队列没有满，直接塞到本地队列。
  
  local goroutine queue 又不能挂在m 上，因为 m会随着syscall （创建新的线程，读取磁盘）阻塞住，不希望local gouroutine 对垒任务受到影响，如果转移又涉及到 队列的管理，更麻烦，于是产生了逻辑处理器 p。
  
  
  gm 的对应关系是 m :n,  1:1简单，但是不能并发，同理 m:1 也是。
  
  g 内存大小默认 2kb， 数量受到内存总量限制。
  
  m 默认10000，可以调整，但系统内核一般承受也就是10000以下。
  
  p 一般设置和系统内核数量一样。
  
  
  
  channel 堵塞，
  网络堵塞
  现在都不会引起 m 的堵塞
  
  m如果堵塞会带着 g 一起出逃 p， 当可以被调度后 m 会找以前的 p 是否需要m， 如果不需要 m会去休眠，g 存放到全局队列。
```



```
 golang 平滑重启原理


  https://www.tizi365.com/archives/83.html
  
  
  // 当前的goroutine等待信号量
  quit := make(chan os.Signal)
  //通过channel， 监控信号：SIGINT, SIGTERM, SIGQUIT
  signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
  
  // 这里会阻塞当前goroutine等待信号
  <-quit
  
  // 调用Server.Shutdown graceful结束
  timeoutCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
  defer cancel()
  
  if err := server.Shutdown(timeoutCtx); err != nil {
  	log.Fatal("Server Shutdown:", err)
  }
  
  // http 和 grpc 都封装好了 shutdown 方法
  // 我的 server 服务也要封装好 shutdown 方法

```



```
golang 的 context 的 withcancel 用过么，什么场景

  withcancel ，
  withtimeout 
  withdeadline 这些都会返回一个新的context 和一个主动触发的cancel ，我们通过 监听 ctx.Done() 这个chan， 在需要的时候，比如超时设置，获取到 chan 中msg， 主动释放资源 （goroutine 不能通过外力去结束，只能在自己的代码中设置对应的条件去结束）
  
  上面的平滑关闭就是一个场景。
  
  可能有的组件关闭写的不合理，总是close 不了，所以给一个最大超时时间。
```



```
  Golang中常用的并发模型的递进
  
  1.原始版本，通过 go 去生成协程， 通过channel通知实现并发控制 (channel 读取阻塞，防止程序没有执行完)
  2.通过sync包中的WaitGroup实现并发控制， add 和 done 一致后，release 挂起的协程 （waitgroup 的坑，之前遇到过，他本身是一个struct, 所以如果在另一个方法中 done， 一定要传入地址类型，否则不是修改自身）
  3. 但并发的时候，我们不能做到某个方法遇到err， 解放其余的goroutine，于是 在Go 1.7 以后引进的强大的Context上下文，实现并发控制.（errgroup 的接受到一个 错误就返回，用到的就是遇到一个error  sync.Once执行一次 ctx 的 cancel() 方法，我们可以通过各个 goroutine 中 select ctx.Done() 获取取消的channel 信号 ）
  
```



```
go 的 gc 回答

gc 其实有两种

1.引用计数法（PHP使用的，某个变量使用次数为 0， 就可以被回收了）

2.可达性分析法。 go 初版本 
stw -》标记 -》清除 -》stw解除，

后来把 stw 解除移动到 清除之前。因为那些无法达到的元素的修改 不可能会被读到，所以没必要担心读写冲突。 

（stw 感觉就是类似锁，所有的程序停止运行）


三色标记法 + 读写屏障，删除写屏障

1.stw开始， 开启屏障，stw 结束
2.全部元素，白色。层序遍历，第一次灰色， 然后所有灰色上升 成 黑色，黑色到达的元素 标记成灰色， 不断的从白色中拿数据，直到最后
3.屏障中是否记录到元素，如果有，stw 开始，重新标记 。还要re-scan 栈空间，栈上没有屏障机制， stw 结束
4.清除

其实三色只是一个状态，并不一定是三色，灰色是一个中间状态，而黑色 和 白色 代表永远不会关联。


上面stw 时间还是过长了 （re-scan栈空间）， 那我们分析下如果让三色法可以支持并发，并且没有stw， 我们可能遇到的情况

1.本应该被回收的变量，没有被回收。这个没啥关系，在下一次gc能被回收就行。
2.不该被回收的变量，被回收了。这个比较严重。这个出现的情况需要满足的条件是

1.黑色增加了对白色的引用 （读写屏障）
2.白色被引用的链条上所有的灰色都被断开了 （删除写屏障）

满足上述条件的情况下，三色法如果是并发的会有问题。

上述两种情况任意被破坏，都会防止并发问题的产生。



插入屏障，删除屏障-》混合写屏障 

（新增引用，如果是白色，标记成黑色。 删除引用，如果是白色，标记成黑色）

栈上所有元素开始标记成黑色，新增的元素也标记成黑色。防止二次 re-scan （re-scan 的stw 被优化掉了。）



还有gc 回收的是heap 上的元素，栈因为很快，不用gc。


stw 时机。

1.开启混合屏障
2.混合屏障记录到有元素变动，重新标记
```



```
gc 触发时机

1.达到阈值。 分配内存时, 当前已分配内存与上一次GC 结束时存活对象的内存达到某个比例时就触发GC。（默认配置会在堆内存达到上一次垃圾收集的 2 倍时，触发新一轮的垃圾收集，可以通过环境变量 GOGC 调整，在默认情况下它的值为 100，即增长 100% 的堆内存才会触发 GC。

2.人为代码操作。 runtime.GC    {调用 runtime.GC() 强制触发GC }

3.时间限制 。 {sysmon 检测出一段时间内（由 runtime.forcegcperiod 变量控制，默认为 2 分钟）没有触发过 GC，就会触发新的 GC。
```



```
go 内存管理

分配算法：tcmalloc (thread cacheing malloc)
核心是：多级管理，降低锁的力度。

如果是小于16b， 直接是通过mcache 的 tiny分配器分配。

如果 大于 32kb, 直接通过 mheap 分配。

16b ~ 32kb，
1.先通过mcache 申请（每个m包含的本地缓存，所以不用加锁）
2.mcache 中没有合适的 向mcentral 中申请 （全局的会加锁，但每个central 保存一种特定大小的mspan， 多线程共享，需要加锁）
3.mcentral 没有合适的 向mheap 中申请 （全局唯一）

https://juejin.cn/post/6844903795739082760
```



```
epoll


底层是红黑树。

通过 create 方法，创建一个 epfd， 类似root 的存在。

add 方法，添加对应socket 的对应事件到 红黑树中。

wait 方法阻塞，等待 对应socket 的对应事件受到触发。
```



```
go cas 操作的实现
```

