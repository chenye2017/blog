



```
工作中印象比较深的事

背景 : go routine panic 没有捕获到 ，导致 pod 重启，  而且还没有打印 panic 堆栈位置，很难排查问题。 (一般容易发生在 job 任务中 ， web server 框架是 gin 会打印出堆栈)
解决 :  
1.消息走先提交，后执行的模式，有问题消息不会被重试， 不会一直导致服务重启 。 
2，利用 addr2line, 查询是哪一行代码出了问题。


 背景： 佩洛西访台， 单redis 流出过大。（100m）
  排查：
  1.观察流出陡增，和链接数增长趋势一致，因为是池化，所以判断 client 增长，此时有扩容，判断是扩容机器上的定时任务。
  2.运维找出大key。（通过key 不能直接推断出业务是因为 接受的老代码）


  背景： 压测，mysql 链接数告警。
  解决：
  先重启
  排查：
  1.机器没有扩容。
  2.链接数之前也有增长比较快，只是没到阈值，没有告警，一段时间之后回归正常， 怀疑自然释放，查询数据库配置 idle， 2h ，和监控上链接数回落整时间一致，确定是业务增长导致连接池链接数增长，之前数据库的active 链接数一直没打满。


  背景：error != nil, 但是打印出来的 %+v error 是nil。 还有panic

  排查：
  1.error 是interface， 可能 type 不等于 nil, data 等于 nil， 但这%+v 也不应该打印出nil， 本地测试会执行Error() 相关
  2.发现 errgroup 中并发修改 err， 所以导致了一系列奇怪的问题。 err != nil, 但是打印出来的err 是 nil

  3. err 本是 interface， 这个赋值公用同一个地址

  err := errors.New("11")
	err1 := err
	
  addr := (*Iface1)((unsafe.Pointer(&err)))
	fmt.Println(addr.D)

	addr1 := (*Iface1)((unsafe.Pointer(&err1)))
	fmt.Println(addr1.D)
```









  ```
  
关于 Atmoic （看看常用方法和原理）

atomic 包含的有 int  ,bool , interface

add  增加

load 加载 

store 存储

compare and swap  参数 addr， old， new  返回是否swap 成功。

之前看到一篇文章关于 cas 和 sync.Once 的区别。 
  
sync.Once 会阻塞 通过 sync.Mutx, 

cas 操作并不会， 直接返回结果。所以得通过 for 循环去自 旋。 

感觉如果不阻塞 住， 后面代码 的变更如果 依赖 那个只用执行一次的func， 但那个func 又比较耗时，结果会不符合预期。

  ```

 



 



  

  



  

  



  ```
 Go 的 sync 包都用过哪些
 
 
 
  sync.Mutex

  sync.Map

  sync.WaitGroup

  sync.Once

  sync.Pool


  sync.Cond 不好用，随机唤醒协程，容易饥饿，一般通过 channel 去解决特定唤醒的问题。


  ```











* go 小对象 大对象， 为什么小对象多了会造成gc 压力

  ```
  https://i6448038.github.io/2019/05/18/golang-mem/
  
  tiny 对象 小于 16b （直接分配在 class 1 的span 上）
  
  小对象  小于32kb
  
  大对象  大于 32kb (mheap 分配，span 0 上，大小不固定)
  
  （大小对象纯属根据size 大小来区分）


  通常小对象过多会导致GC三色法消耗过多的 cpu。优化思路是，减少对象分配.

  优化
  1.利用 sync.pool , 这里面都是无状态的变量 ，不能用于 conn 这种连接信息，而且个数不确定，不适用 连接池固定最小数量，空闲数量 idle

  2.利用 数组生成一匹空对象，从中取值，

  for _, v := range []int{1,2,3} {
     tmp := People{Name: v}
  }

  优化后 // 连续空间的读取，比无序的效率更高

  objectArr := make([]People, 3)

  for k, v := range []int{1,2,3} {
     tmp :=  objectArr[k]
     tmp.Name = v
  }

  // 变种,哪个效率高 （第一个，连续的地址空间）

  for i:=0; i< 10;i++ {
    for j:=0; j < 10; j++ {
        tmp := arr[i][j]
    }
  }
  for i:=0; i< 10;i++ {
    for j:=0; j < 10; j++ {
        tmp := arr[j][i]
    }
  }

  ```

* go 内存管理

  ````
  https://zhuanlan.zhihu.com/p/266496735

  对象切割，多级缓存，位图管理。

  mcache， 依附于 p， 没有并发，所以不用加锁。

  mcache (各种级别的 span, 不同级别的span 上基本单元大小不一样。span 上的内存分配大小是按照page 分的。 span 有两种类型，一种地址类型，一种值的类型，主要是为了方便gc)

  mcentral (被mcache 共享，所以需要加锁，每个mcentral 管理一个有空闲对象的 span, 一个没有空闲对象的span。寻找 分配空间会对上面两个链都进行遍历，因为可能有的空间被gc 了还没有直接放回到空闲对象的span上)

  mheap （管理mcentral, 管理大对象的分配，通过位图标记知道哪些span 被使用，哪些没有被使用）

  mheap 基数树的形式管理span， 如果没有合适的span ，向内存中申请页大小。

  span 上分配内存

  span 有很多种级别，67 个 。 0级别（不在 67个之内） 大小不固定。 剩下级别的span 大小逐渐递增，（个数不固定，导致span 大小不一定）。对齐内存到指定大小的span 上。

  proccer 进行内存分配的时候，先找缓存 mcache 中 （包含了所有的 mspan），找不到大小空间合适的 mcentrl  中查找 （被共享要加锁，后面更高的肯定要加锁），再找不到 mheap  位图 中查找， mheap 二叉搜索树中查找 ，再左后是虚拟内存 （假装连续空间）分配。



```



微服务相关，包括微服务的注册与发现，微服务的监控，微服务的限流相关等等，还有微服务怎么守护进程，我回答的是 supervisor，也不知道对不对。
具体业务的实现，兑换码的实现，如何批量生成几十万或者上百万的兑换码，（这个我回答的是用雪花算法实现），高并发下，一个兑换码如何保证只能被一个人兑换一次，以及一个兑换码如何可以被多个人兑换的实现。（这道题前前后后回答了有半个小时吧，因为之前做过相关的业务，所以心里有点底）

三个算法问题。
写一个方法，解决：输入 4 个数字，通过加减乘除，输出一个期望值。
广度优先算法：怎么输出各层的值。
台阶问题，假如对于上台阶，可以一次上一阶，也可以一次上两阶，写一个方法，实现输入台阶数，输出可以有多少种上法。




```



```







CAS 具体是怎么实现的呢？












Redis 哪些场景在使用？
1.k-v 类型配置
2.zset 排行榜，延迟队列
3.set 去重，最近购买记录
4.hash。用户和主播对应关系。


说一下分布式锁的实现？
1.set ex 过期时间
2.set nx 不存在才设置。
3.go func 定期续约。
4.lua compare and delete 删除数据，防止删错了


基于 Redis 的分布式锁会有什么问题？
我们后端用的redis cluster。主从节点切换 会存在数据丢失问题。redlock 会出现时钟偏移可以重复加锁问题。

Redis 分布式锁超时可以超时时间设长一点可以吗？不可以的话需要怎么解决？
可以，也可以启动一个类似看门狗 watch dog 的协程，定时续约，到点了 close time ticker 和  删除锁。


日常什么项目会用到缓存，用 Redis 做缓存有遇到什么问题吗？
排行榜。之前某个主播下守护者存成一个string， 用pb序列化，每次前端读取部分数据，服务端都要拉全量数据。qps 高，带宽飙高，超100m/s.
大key， bitmap 序列化成二进制存储到redis， 0.6m， 每台机器每秒拉到内存，扩机器，带宽飙高，redis 集群开始不稳定。


平时在使用 Kafka 吗，具体做哪些业务使用到？
异步场景，发奖励。


为 SELECT e FROM a WHERE b = 1 AND c > 1 ORDER BY d; 建立索引。
b，c 联合索引。


有用过 EXPLAIN 吗，结果中有哪些字段值得关注？

type  是不是 const range 这种比较优秀的
key 用到了什么索引
key_len 用到了索引的全部还是部分
extra 里面是否有 file_sort 这些坏东西，是否有 using index 覆盖索引，using index condition 索引下推。






```









  ````

  ![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/img/20211012202547.png)

* g0

  ```
  在Go中 g0作为一个特殊的goroutine，为 runtime 执行调度循环提供了场地（栈）。对于一个线程来说，g0 总是它第一个创建的 goroutine。
  
  之后，它会不断地寻找其他普通的 goroutine 来执行，直到进程退出。
  
  当需要执行一些任务，且不想扩栈时，就可以用到 g0 了，因为 g0 的栈比较大。
  
  g0 其他的一些“职责”有：创建 goroutine、deferproc 函数里新建 _defer、垃圾回收相关的工作（例如 stw、扫描 goroutine 的执行栈、一些标识清扫的工作、栈增长）等等。
  ```

  

* go 数据竞争怎么解决

  ```
  检测方法
  
  $ go test -race mypkg    // 测试包
  $ go run -race mysrc.go  // 编译和运行程序
  $ go build -race mycmd   // 构建程序
  $ go install -race mypkg // 安装程序
  

  ```

* go sync.mutex 和 atomic 的底层实现

  ```
  atomic cpu 的原子性指令去实现。
  
  //sync.Mutex 中 state 4种状态
  // locked
  // woken
  // starving
  // waitersCount
  
  sync.mutex 底层也是依赖这 atomic。通过cas 指令去修改锁是否被lock
  
  如果获取不到锁，通过 runtime 函数（汇编的指令）让这个goroutine 休眠。
  
  加锁的时候，判断当前状态是饥饿状态（ goroutine 等待 1ms以上）还是正常状态，饥饿状态新生成的等待竞争锁的goroutine 直接放到队列尾部 ， 不和唤醒的goroutine 竞争 （具有cpu 亲和性，优先级更高），当竞争到锁的goroutine 等待时间小于 1ms 的时候， 饥饿状态就会解除， 又会让 新生成的 goroutine 和 唤醒的goroutine竞争。
  
  加锁的同时引入了自旋策略，一般是4次的尝试，失败了就进入睡眠状态。防止太多失败，所有的goroutine都进入park，浪费 runtime 的调度。（当然自旋有条件的，比如多核，没有等待的goroutine）
  ```

* Go 抢占式调度原理

  ```
  以前如果调用函数，会判断是否执行时间过长，如果过长，就会主动释放这个goroutine.
  
  但类似for循环这种不能退出，于是现在通过一个后台goroutine 主动监听，如果发现了，通故给p 发出信号，p 主动去停掉对应的goroutine.
  ```

* 都是加锁，为什么用 channel （通信）去代替共享内存

  ```
  channel 来数据的时候 ， recvq 链表中有等待goroutine， 会把来的数据直接copy 到对应goroutine的的栈中，而不需要加锁操作。（感觉是因为队列这个属性已经满足了fifo，所以不用加锁）
  ```



  ```



* go 中的锁有哪些

  ```
  1. 互斥锁 （多次 unlock 造成的 panic 竟然 锁不住。因为也是计数型型号量，所以到了 -1 会报错）
  2. 读写锁 （这两种都是 悲观锁）。读锁可以多次lock， 用的是计数型型号量。到了-1会报错
  ```

* Goroutine 的状态

  ```
  runtime.NumGoroutine函数在被调用后，会返回系统中的处于特定状态的Goroutine的数量。这里的特指是指 Grunnable\Gruning\Gsyscall\Gwaition。处于这些状态的Groutine即被看做是活跃的或者说正在被调度。
  ```

* 实现一个协程池

  ```
  业界 有 ants。
  
  1.b站开源的 errgroup 的处理中包含了最简单的携程池的实现， 通过 for 循环生成对应数量的gorouine， 然后并发的获取 channel （安全）
  2.通过信号量，也就是数字的num 去控制channel 的数量。 通过 load 获取大小，大于0 才执行。compare and swap 去修改大小。因为非阻塞，所以要一直循环。
  ```


  
 

  

* go 的gc机制

  ```
  计数法和标记染色法。
  
  从root节点到达的第一批节点标记为灰色，灰色上升黑色，黑色节点到达的节点标记为灰色。重复上述步骤。
  
  因为存在并发，所以引入了stw。
  
  为了减少stw，在生成新的节点和 节点修改引用的时候都会标记成灰色。
  ```

  



* http 包实现原理

  ```
  Golang中http包中处理 HTTP 请求主要跟两个东西相关：
  ServeMux (路由匹配) （平时我们注册路由，绑定具体的方法）和 
  Handler （请求处理）（方法中写具体的业务逻辑）。
  
  ServeMux 本质上是一个 HTTP 请求路由器，它把收到的请求与一组预先定义的 URL 路径列表做对比，然后在匹配到路径的时候调用关联的处理器（Handler）。
  
  处理器（Handler）负责输出HTTP响应的头和正文。任何满足了http.Handler接口的对象都可作为一个处理器。通俗的说，对象只要有个如下签名的ServeHTTP方法即可：
  
  ServeHTTP(http.ResponseWriter, *http.Request)。
  
  
  具体的 http server
  
  监听 端口（比如8080） =》 
       for 循环，通过端口获取链接  =》 
       协程出去，处理链接上请求，如果是https, 先进行tls 握手🤝。因为现在一般是长链接，所以需要 for 循环处理。通过读取链接上数据，进行路由匹配，找到具体的处理发放。
  
  ```

  





  





* go的并发模型 （golang 如何调度goroutine）

  ```
  GMP （runtime 怎么调度 goroutine 的执行的）
  
  CSP (描述并发系统中交互模式的形式化语言，通过共享内存去通信，而不是通信去共享内存 。go 语言参考了三种， c的语法， csp 的并发 channel 通信， Pascal 的 import package)
  
  CAS （原子性操作， compare and swap）
  
  ```
  



*

  

* 半连接是什么

  ```
  全双工链接没有建立成功。三次握手，只是握了2次
  ```
  
* 粘包是什么，怎么发送的

  ```
  1.tcp 本质上是流的存在，不存在粘包，粘包，是上一层协议的解析导致
  2.由于存在缓冲区（发送缓冲区， 凑足整个buf）一次性发送，可以通过关闭这个设定。但由于接受缓冲区的存在，还是有粘包问题，我们可以通过类似content-length, 获取到整个包的长度。或者通过某个特殊字符进行区分 （但会存在二进制不安全问题）。
  ```

  



  

* 怎么解决缓存击穿，还有其他什么方法

  ```
  1.雪崩（ttl + random key）
  2.穿透 （空值 也埋一个cache）
  3.击穿（某一个时间点某个key 大量的查询，都打到db上，用singleflight 这种挡住）
  ```

  

* Go 的mutx 怎么使用，乐观锁和悲观锁分别怎么实现，使用场景是什么

  ```
  1. 直接使用， 不需要初始化。
  2. 悲观锁，mutex 互斥锁，或者读写锁
  3. 乐观锁感觉 类似csp 操作。（es）
  
  并发度高用悲观锁，并发度低用乐观锁
  ```

* 服务器受到攻击怎么定位服务器问题

  ```
  1.查看流量来源，是否来自同一个ip
  2.查看哪个接口来源，是否有活动，是否流量正常
  ```

* rpc的具体实现

  ```
  编写clientstub(主要用来序列化请求参数，获取链接，获取请求服务的ip+port,) 
  
  serverstub （接受链接，反序列化请求参数，处理相应请求）
  
  复杂均衡：轮训，随机，根据url hash，根绝来源ip hash, 加权
  
  注册中心：接受服务注册，提供服务发现 consule
  
  过滤器： 包含鉴权，熔断，降级，限流
  ```

* 怎么反转树的左右节点

  ```
  ```

* 谈谈epoll 和 select

  ```
  epoll 遍历fd 文件，fd 文件的数组大小个数有限制。 轮训去查询 read write 事件
  
  select fd 事件集合大小没有限制，有read write 事件主动通知主进程
  
  ```

* go怎么读取大文件

  ```
  https://learnku.com/articles/23559/two-schemes-for-reading-golang-super-large-files
  
  1.行读取文件 分隔符
  2.slice 读取文件
  ```







* 往一个对象里写入 10w 条数据，怎样保证数据的准确性（chan、mutex 之类的胡扯就对了）

  ```
  atomic 自增
  sync.Mutex 互斥锁
  ```







* 空结构体

  ```
  没有field 的结构体， 生成的时候空间地址一定  struct{} 
  
  空结构体添加到结构体中，只要不是在最后一个field， 都不占用空间，如果是最后一个field 会需要进行内存对齐，占用空间
  
  !!!!notice, 别人问你
  
  var 是否占用内存空间的时候， 一般都会占用，但是 如果声明的类型是 ，struct{}， 这种空结构体不占用内存空间 
  ```

* 内存对齐是什么

  ```
  1.cpu 读取数据不是一个字节一个字节读取的，比如一次性读取4 个字节或者8个字节， 减少类似粘包，分包的那种问题。32位机器和64位机器。这样读取也能保证原子性操作。
  2.不同数据类型对齐长度不一样，会让之前的字段填充符合自己被读取时候整数倍，（比如struct）进行padding。
  ```

  




 
  
  



* CAS  compare and swap 是非阻塞的，所以要想实现增长，必须还得 for 循环去调用 (牺牲cpu 去不加锁，兑换上下文切换的开销) （乐观锁，compare and swap， 比较两个数是否一样，一样再替换）

  ```
  CAS算法（Compare And Swap）,是原子操作的一种, CAS算法是一种有名的无锁算法。无锁编程，即不使用锁的情况下实现多线程之间的变量同步，也就是在没有线程被阻塞的情况下实现变量的同步，所以也叫非阻塞同步!!!!!非阻塞！！！非阻塞！！！（Non-blocking Synchronization）。可用于在多线程编程中实现不被打断的数据交换操作，从而避免多线程同时改写某一数据时由于执行顺序不确定性以及中断的不可预知性产生的数据不一致问题。
  
  该操作通过将内存中的值与指定数据进行比较，当数值一样时将内存中的数据替换为新的值。
  
  Go中的CAS操作是借用了CPU提供的原子性指令来实现。CAS操作修改共享变量时候不需要对共享变量加锁，而是通过类似乐观锁的方式进行检查，本质还是不断的占用CPU ！！！非阻塞带来的cpu 空转 资源换取加锁带来的开销（比如上下文切换开销）。
  
  
  package main
  
  import (
  	"fmt"
  	"sync"
  	"sync/atomic"
  )
  
  var (
  	counter int32          //计数器
  	wg      sync.WaitGroup //信号量
  )
  
  func main() {
  	threadNum := 5
  	wg.Add(threadNum)
  	for i := 0; i < threadNum; i++ {
  		go incCounter(i)
  	}
  	wg.Wait()
  }
  
  func incCounter(index int) {
  	defer wg.Done()
  
  	spinNum := 0
  	for {
  		// 原子操作
  		old := counter
  		ok := atomic.CompareAndSwapInt32(&counter, old, old+1) // 这个地方其实直接加就行了，因为大小边界没有限制
  		if ok {
  			break
  		} else {
  			spinNum++
  		}
  	}
  	fmt.Printf("thread,%d,spinnum,%d\n", index, spinNum)
  }
  
  
  当主函数main首先创建了5个信号量，然后开启五个线程执行incCounter方法,incCounter内部执行, 使用cas操作递增counter的值，atomic.CompareAndSwapInt32具有三个参数，第一个是变量的地址，第二个是变量当前值，第三个是要修改变量为多少，该函数如果发现传递的old值等于当前变量的值，则使用第三个变量替换变量的值并返回true，否则返回false。
  
  这里之所以使用无限循环是因为在高并发下每个线程执行CAS并不是每次都成功，失败了的线程需要重写获取变量当前的值，然后重新执行CAS操作。读者可以把线程数改为10000或者更多就会发现输出 thread,5329,spinnum,1 其中这个1就说明该线程尝试了两个CAS操作，第二次才成功。
  
  因此呢, go中CAS操作可以有效的减少使用锁所带来的开销，但是需要注意在高并发下这是使用cpu资源做交换的。
  
  // 参考 ：http://wearygods.online/articles/2021/04/19/1618823886966.html#toc_h4_28
  
  ```

* 信号量和信号

  ```
  中断信号
  
  信号量 ，一般是计数型信号
  ```


  




* 下面的代码会卡死么，为什么

  ```
  package main
  
  import (
      "fmt"
      "runtime"
  )
  
  func main() {
      var i byte
      go func() {
          for i = 0; i <= 255; i++ {
          }
      }()
      fmt.Println("Dropping mic")
      // Yield execution to force executing other goroutines
      runtime.Gosched()
      runtime.GC()
      fmt.Println("Done")
  }
  
  
  解析：(溢出竟然没什么问题
  )
  
  Golang 中，byte 其实被 alias 到 uint8 上了。所以上面的 for 循环会始终成立，因为 i++ 到 i=255 的时候会溢出，i <= 255 一定成立。
  
  也即是， for 循环永远无法退出，所以上面的代码其实可以等价于这样：
  
  go func() {
      for {}
  }
  正在被执行的 goroutine 发生以下情况时让出当前 goroutine 的执行权，并调度后面的 goroutine 执行：
  
  IO 操作
  Channel 阻塞
  system call
  运行较长时间
  如果一个 goroutine 执行时间太长，scheduler 会在其 G 对象上打上一个标志（ preempt），当这个 goroutine 内部发生函数调用的时候，会先主动检查这个标志，如果为 true 则会让出执行权。
  
  main 函数里启动的 goroutine 其实是一个没有 IO 阻塞、没有 Channel 阻塞、没有 system call、没有函数调用的死循环。
  
  也就是，它无法主动让出自己的执行权，即使已经执行很长时间，scheduler 已经标志了 preempt。
  
  而 golang 的 GC 动作是需要所有正在运行 goroutine 都停止后进行的。因此，程序会卡在 runtime.GC() 等待所有协程退出。
  ```

 

* 实现阻塞度并且安全并发的map

  ```
  // 这题的核心代码，就是那个entry 体，里面有个chan， 通知要获取他的key
  // 这题目的解法和 single flight 特别像
  type Map struct {
  	c   map[string]*entry
  	rmx *sync.RWMutex
  }
  type entry struct {
  	ch      chan struct{}
  	value   interface{}
  	isExist bool
  }
  
  func (m *Map) Out(key string, val interface{}) {
  	m.rmx.Lock()
  	defer m.rmx.Unlock()
  	item, ok := m.c[key]
  	if !ok {
  		m.c[key] = &entry{
  			value: val,
  			isExist: true,
  		}
  		return
  	}
  	item.value = val
  	if !item.isExist {
  		if item.ch != nil {
  			close(item.ch)
  			item.ch = nil
  		}
  	}
  	return
  }
  ```


* go 协程调度

  ```
  1.14 开始引入抢占式协程调度 （时间片用完就会被强制切换）
  信号量抢占
  ```
  
* go 的runtime

  ```
  在把用户写的程序翻译成可执行文件的过程中，把 runtime 代码塞进了
  可执行文件
   1.初始化全局变量，
   2.调用每个模块的init函数
   3.初始化 GC，以及初始化 Go scheduler
   4.启用一个协程，调用用户写的 main 函数。
  ```
  
  
  


  

* go 怎么实现的锁

  ```
  ```



* go 协程池

  ```
  ```



  




* 分布式事务

  ```
  ```

* go 的优势

  ```
  Go 允许跨平台编译，编译出来的是二进制的可执行文件，直接部署在对应系统上即可运行。一次编译到处执行。（但其实也不是所有的操作系统）
  Go 在语言层次上天生支持高并发，通过 goroutine 和 channel 实现。channel 的理论依据是 CSP 并发模型， 即所谓的通过通信来共享内存；Go 在 runtime 运行时里实现了属于自己的调度机制：GMP，降低了内核态和用户态的切换成本。
  Go 的代码风格是强制性的统一，如果没有按照规定来，会编译不通过。
  ```






  









* go 内存模型

  ```
  ```



* go map 缩容

  ```
  ```







 
  // mysql 切主，容灾。
  

  


