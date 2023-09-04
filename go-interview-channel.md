

  ```
  channel 底层数据结构
  
  https://segmentfault.com/a/1190000040263156
  
  buf 循环数组，有缓冲的channel 才有
  
  sendx
  
  recqx (发送 和 接受指针)
  
  两个双向链表 （一个send 生产者grooutine list， 一个 consumer 消费者 goroutine list），保证了先进先出。
  
  close 状态
  
  sync.Mutex 互斥锁，防止并发。
  ```





```
channel 的读取

 // 两种单方向的 chan
 var  c  <- chan int
  	  
var  c1  chan <- int
  	
 // 读取数据 	
 c, bool := <-chan
  	
 注意，channel中有元素的时候，不管 channel 是否关闭，都是true， 其实就是标志这个数据是不是init data (默认值)
  	
 就是这个标志位不会实时显示 channel 是否关闭，而是代表读取出来的数据是否是默认零值。
 
 如果关闭了，channel 读取不会阻塞，可以返回值， 否则是阻塞获取数据。
  	
  	
  	
 // 给 nil chan 发送数据和接受数据，都会永远堵塞
  	
  	  
  	
 // 代码中我们经常要做到通知 关闭的功能
 // 之前 我们是向一个 作用关闭的channel 中塞入一个信号，当我们从中获取到 信号的时候，代表功能关闭
 // 其实我们大可不必， 可以直接把这个channel 设置成  <-chan struct{}, 当这个chan 不堵塞的时候，代表这个chan 关闭了 （context 包中的 done 方法就是这样实现的）

 
```



```
channel 是否是线程安全的。

是，结构体中包含了 一个 sync.Mutex 锁。



那 map ，slice 这些呢 ？

不是，属于临界资源，没有加锁。
map 只有一个标志位， 并发读写的时候会panic （throw 错误）,而且map 的并发读写还不能被 recover。
  
slice 也非线程安全
  
string 也非线程安全。
  
int 这些但凡没有 锁的都是线程不安全 （底层不能用一个cpu 原子指令执行），需要用 atomic包。
```



```
channel 是否是引用类型

底层是 *hchan


func 
map   （可以直接在函数内部修改，但是感觉这样写容易出bug， 我们的func 作用是修改map， 感觉容易被忽略，感觉不如把值返回出来）
slice （原则上讲还不是，只是结构体中包含一个指向数组的指针, 扩容会修改地址）
channel  （所以channel 可以在各个func 中传递， 让并发编程更加方便）
 
 
map 如果不初始化， 读取没问题，赋值会报给 nil pointer address 赋值。

slice append 没什么问题， 具体index取值 有问题。

channel 发送会阻塞。
```



```
channel 关闭了接着 send 数据会发生什么，关闭一个已经关闭的 channel 会发生什么。

1.send close 会panic 。（想象一下，send 并没有返回值，不能出现别的情况）

2. close close 的 也会panic。 （没有返回值）


3.获取 close 的channel， 先把数据读取完，再获取到默认零值，通过 comma 去判断。

4.send 或者 获取 没有初始化的 channel， 会阻塞。
```



```
两个goroutine 怎么控制先后顺序


  c := make(chan struct{})
  go func() {
    fmt.Println("a")
    c <- struct{}{}
  })
  
  go func() {
      select {
       case tmp := <- c :
       		fmt.Println("c")
      } 
  }
  
  time.Sleep(1)

```



```
 关闭 channel (平时工作中 大部分定义的都是普通channel， 双向channel)

  func Stop(stop <-chan bool) {
      close(stop)
  }
  不能关闭 读channel
  
  var cc  chan <- int
  close(cc)  // 这是可以的

```



```
go 的select 的default 的作用 (优先级最低)

当 select 中的其他条件分支都没有准备好的时候，`default` 分支会被执行。
为了非阻塞的发送或者接收，可使用 default 分支：
```



```
 channel 被关闭还能读出值么？ 多次读的时候会返回什么


  channel被关闭后是可读的，直到数据读完为止。
  被关闭后，读取的如果是已经存在的值，注意标志位是true(想象一下，如果关闭之后直接就读取到false，那就没法判断读到的0 值到底是 真实数据， 还是 close 后给的默认数据)
  如果继续读数据，得到的是零值(对于int，就是0)， 两个返回值，另一个是 false
  
  
  channel 被关闭 可以一直读取。 比如利用 for 循环，关闭后，for 读取不会被阻塞。
  
  for range 读取不到值，且不会被阻塞。
  
  channel 没有被关闭，读取会被阻塞。
  for  n 循环 和 for range 都会被阻塞。
```





```
 
  channel 的 读取。 常用的操作
  
  1. for range 读取  
  
  channel 如果关闭了，不会阻塞。 如果没有关闭，会阻塞住一直获取数据
  
  2. for i:=0;  i < 10  . select 读取
  
  如果关闭了，也不回阻塞，能读取到数据。如果没有关闭，也是阻塞在获取数据处
  
  
    c13 := make(chan int, 10)
  	close(c13)
  
  	/*for i:=0; i < 3;i++ {
  		fmt.Println(<-c13)
  	}*/
  	/*for  v := range c13 {
  		fmt.Println(v)
  	}*/
  	for {
  		select {
  		case <-c13:
  			fmt.Println("111")
  			
  		}
  	}
  
  1. for i:=0; i < 4; i++  
  
  这种经常也是最安全的读法，channel 。如果close 了，读取到默认值。
  
  2. for tmp := range c   这种如果 channel 没有被close， 会被阻塞住。
  
  如果 close了，不阻塞。 （走不到 for range 中）
  
  
  3. for {
     select
  }
  
  同第一种方案。
  
  
  这种正常情况下没有问题， channel 读取不出来数据会被阻塞，但是当channel close 之后，会一直能不间断的读取到数据，需要注意，一定要判断 value, comma := <- chan  ,判断这个数据是否是默认值。
  
  利用上面 comma 和关闭后不阻塞 的特性，如果我们的channel 只是想用来通知关闭信号， 可以不往里面塞值，直接获取 comma ，如果能获取到了，说明channel 关闭了
  
  
  3. for select 这种形式，break 是跳不出来的， 只能用 return 或者 goto
```



```

 go 的引用类型有哪些



https://blog.csdn.net/love666666shen/article/details/99882528

这篇文章对值类型和引用类型有很好的解释

值类型： int float， bool， string (不可变类型)， 数组， struct, interface （感觉就是能否用nil 表示, interface 应该是结构体，可以用 unsafe.Point 去验证）

引用类型 ：slice, map, , chan （多个函数内部传递, 虽然是值传递，但架不住引用类型）, func

值类型的特点是：变量直接存储值
引用类型的特点是：变量存储的是一个地址


c := 10
var a interface{}
a = c
value := **(**int)(unsafe.Pointer(uintptr(unsafe.Pointer(&a)) + 8))

fmt.Println(value, "----")
```



```
  
 简单谈下 chan 的应用场景及注意事项

 不同协程之间交互数据
 
 常用的有
 1.超时取消
 2.存储结果进行分发，比如singleflight， 连接池获取链接
 
```

