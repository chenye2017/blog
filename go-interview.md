* go 语言遇到的坑

  ```
  1.slice  相关问题
  make([]int, 5), len是5， 被0元素填充
  只想防止扩容， 应该改成 make([]int, 0, 5)
  
  slice 三个属性组成，
  数组指针，
  len 实际占用大小， 
  cap 容量。
  
  slice append， 如果cap 容量满足，就在原先数组len 后面填充，无论是否已经有元素占领。
  容易在用切片的时候掉进坑里。
  
  slice 切片， []int{}[0:4], 影响自身， 此时并没有生成新的数组，指针只是移动了位置，之前指向数组第一个元素，现在指向的是index开始的元素。
  
  start:end 这种切片append 很容易出现意想不到的效果，给原始数组造成影响。因为append 填充的元素是针对原始数组的，是否扩容是原始数组的cap 是否满足当前需求, 如果原始数组cap 能容纳，不用扩容，这个append的操作会影响多个切片
  
  
  2. for  range 执行 goroutine, 变量问题 （这个应该各个语言都有，闭包变量作用域问题）
  （黑金宝盒问题类似，闭包中使用的变量是外部改变的变量，而没有用传入的方法，类似defer 执行）
  
  之前有个同事 defer log.Print("%+v", res). 本想着打印最后的结果集，结果因为参数开始执行的时候已经确定了。并不会真的在打印出最终的res
  
  
  3.interface 比较问题。
  
  iface {
   _itab 类型
   data 数据指针
  }
  
  eface {
   type 类型
   data 数据指针
  }
  
  interface 和 nil 比较，必须是 data 和 type 都是 nil，才能判定。
  
  
  4. 并发问题 （race）
  -race 参数
  
  
  5.cannot use names (type []string) as type []interface {} in argument to printAll） 
  es 的 terms 查询条件。（不明朗）
  interface 参数可以被 string 等很多类型传入，但是 []interface 不能直接用 []string, 需要主动构造[]interface。 确实时间复杂度是 o(n)， 得一个一个转换
  https://www.jianshu.com/p/03c81f4e3956
  
  ```


* 工作中印象比较深刻的事

  ```
  背景 : go routine panic 没有捕获到 ，导致 pod 重启，  而且还没有打印 panic 堆栈位置，很难排查问题。 (一般容易发生在 job 任务中 ， web server 框架是 gin 会打印出堆栈)
  解决 :  
  1.消息走先提交，后执行的模式，有问题消息不会被重试， 不会一直导致服务重启 。 
  2，利用 address2line, 查询是哪一行代码出了问题。
  
  
  背景： 佩洛西访台， 单redis 流出过大。（100m）
  排查：
  1.观察流出陡增，此刻监控上链接数增长，因为是池化，所以client 增长，此时有扩容，判断是机器上的定时任务。
  2.运维找出大key
  
  
  背景： 压测，mysql 链接数告警。
  解决：
  先重启
  排查：
  1.机器没有扩容。
  2.链接数之前也有增长比较快，只是没到阈值，没有告警，一段时间之后回归正常， 怀疑自然释放，查询数据库配置 idle， 2h ，和监控上链接回归整时间一致，确定是业务增长导致连接池链接数增长。
  
  
  背景：error != nil, 但是打印出来的 %+v error 是nil。 还有panic
  
  排查：
  1.error 是interface， 可能 type 不等于 nil, data 等于 nil， 但这%+v 也不应该打印出nil， 本地测试会执行Error() 相关
  2.发现 errgroup 中并发修改 err， 所以导致了一些列 后面打印 以及 panic 这种异常情况
  
  ```

  

* go  for range slice 循环的时候执行 append, 会死循环么

  ```
  不会，
  https://blog.csdn.net/wohu1104/article/details/115496718
  
  这个for range 其实就是 
  
  arrCopy = arr
  for i:=0； i< len(arrCopy); i++ 的封装
  循环的次数在循环开始的时候就确定了
  ```

  

* map 遍历的时候append， 可以读取到么

  ```
  map 的遍历没有次数限制
  https://segmentfault.com/a/1190000023477520
  
  map 是无序的，随机从一个bucket（也就是index），故意的， 开始遍历，不会像 slice 那样有固定的次数。
  
  不仅仅bucket 任意，而且bucket 下面的key 也是任意
  
  可能读取到，也可能读取不到，随机（如果插入的元素在当前遍历到的元素后面，就能输出）
  
  ```

  

* map 可以边遍历边删除么 

  ```
  同一个协程中可以的。
  
  不同的协程中会报冲突 （并发不安全，可以用sync map）
  ```

  

* map  struct 类型value 修改  （不是很明确，后面还是看看rehash 的源码理解下）

  ```
  不可以， 会rehash， 
  
  key => value 地址 ，可能会变动
  
  m1 := make(map[string]S1)
  
  m1["test"] = S1{Name: "test"}
  m1["test1"] = S1{Name: "test1"}
  m1["test1"].Name = "TEST" // php 中经常这么编写
  
  // go 中这样是不允许的， 我觉得和rehash 有关，go的迁移过程是动态的，
  //slice之所以可以这样取值感觉是因为slice 的迁移一次性完成
  
  
  // 下面这种方法可以解决map 的取地址问题。
  m1 := make(map[string]*S1)
  m1["test"] = &S1{Name: "test"}
  m1["test1"] = &S1{Name: "test1"}
  m1["test1"].Name = "TEST" //
  
  
  // string 也不能寻址，所以
  a := "tt"
  a[0] = "c" // 这样是不能修改的
  a = "ttt" // a的地址其实是改变的
  
  // 常量，零值也不能寻址，意思就是不能修改。
  
  issure: https://github.com/golang/go/issues/3117
  问题原因不明，主要是
  m1[int]int
  m1[int] = 1 这种就可以直接赋值，为啥struct 不可以。
  目前只知道的是解决办法
  
  
  官方最简单的说法是 m[1] 可能不存在。这时候不知道怎么处理，到底应该是error 么，slice 是直接panic
  
  最常见的说法是 map 的value 不可以寻址。
  ```

  

* map 的比较  

  ```
  map 本身不能比较，deep equal   （不能用 == ）
  slice 也不能比较大小 （不能用 ==）
  只能用 reflect.DeepEqual, 其实很少见 map 和 slice 需要比较大小的
  interface 可以比较大小, 常见error 比较大小。
  
  注意这些比较大小的时候，都和元素顺序有关，（slice 有关，map 本身就是无序的，应该没啥关系，interface 这种涉及到 type 元素 和 元素自身的比较大小）
  
  
  https://juejin.cn/post/6881912621616857102 
  关于是否可以比较
  
  struct 可以比较的前提是必须是同一种数据类型，或者是强制转换成统一种类型。 
  
  
  type T2 struct {
      Name  string
      Age   int
      Arr   [2]bool
      ptr   *int
      map1  map[string]string
  }
  
  type T3 struct {
      Name  string
      Age   int
      Arr   [2]bool
      ptr   *int
      map1  map[string]string
  }
  
  func main() {
      var ss1 T2
      var ss2 T3
      
      ss3 := T2(ss2)
      fmt.Println(ss3==ss1)   // 含有不可比较成员变量
  }
  
  go 中强制转换还挺容易
   
  ```

  

* go 中关于interface 解释

  ```
  // interface 是值类型， 只是两个字段 _type 和 data 都是 unsafe.pointer, 都是引用
  
  // interface ===> iface (包含方法), eface (不包含方法)
  
  // 并不是说 iface 或者 eface 是引用类型，而是因为
  iface {
   itab uintptr
   data uintptr
  }
  
  eface {
   _type uintptr
   data uintptr
  }
  
  包含的第一个属性是 引用类型，类似 slice 的说法。
  
  // 本质上和 map -》 hmap 不一样， map 取得是 hmap 的地址类型，上面的验证发放可以通过 unsafe 去验证
  
  // 获取 map 的 len
  hmap 
  {
   count int
  }
  
  m := map[int]int{11:10}
  len := **(**int)(unsafe.Pointer(&m))
  
  
  // 获取 slice 的len
  slice 
  {
   data uintptr
   len int
   cap int
  }
  
  arr := []int{1,2}
  len := (*int)(unsafe.Pointer)(uintptr((unsafe.Pointer(&arr) + 8))
  
  
  // 这个 error 错误比较很重要，因为类似redis 查询的时候都会用到， redis.ErrNil, 所以比较的时候都是对比 package 里面定义的变量
  
  
  https://www.veaxen.com/golang%e6%8e%a5%e5%8f%a3%e5%80%bc%ef%bc%88interface%ef%bc%89%e7%9a%84%e6%af%94%e8%be%83%e6%93%8d%e4%bd%9c%e5%88%86%e6%9e%90.html
  
  看了上面的文章，发现要真的是 errors.New("11") == errors.New("11") 如果为true 的话，那就完了。因为error 生成的时候又不会全局检查，万一一样了。
  
  没有方法的interface , 包含元素类型 和 元素的实际存储位置，只有二者都相等，我们才能认为2个interface 相等。
  
  同理 interface 和 nil 比较大小，也就是只有 type 和 value 都等于 nil 的时候我们才能认为他是 nil
  
  
  https://mp.weixin.qq.com/s/F9II4xc4yimOCSTeKBDWqw
  
  煎鱼的这篇文章很好的诠释了工作中可能不小心遇到的问题。
  
  对于结构体指针类型，我们可以直接用nil 比较。
  
  对于 interface （error） 类型，我们要结合 type 和 value 一起比较
  
  
  以上interface 的比较，就遇到之前在做 redis ErrNil 错误的对比，注意ErrNil 是我们申明的一个变量，方法里面我们使用这个变量的时候，直接return 这个包级别的变量，不能根据 message 再重新生成，否则不等于 （临时生成的变量地址，还外抛， 野指针）
  
   
  
  go 中interface 容易遇到的问题：
  https://sanyuesha.com/2017/07/22/how-to-understand-go-interface/
  
  package main
  
  import "fmt"
  
  type Person struct {
      age int
  }
  
  func (p Person) Elegance() int {
      return p.age
  }
  
  func (p *Person) GetAge() {
      p.age += 1
  }
  
  func main() {
      // p1 是值类型
      p := Person{age: 18}
  
      // 值类型 调用接收者也是值类型的方法
      fmt.Println(p.Elegance())
  
      // 值类型 调用接收者是指针类型的方法
      p.GetAge()
     
  
      // ----------------------
  
      // p2 是指针类型
      p2 := &Person{age: 100}
  
      // 指针类型 调用接收者是值类型的方法
    
      // 指针类型 调用接收者也是指针类型的方法
      p2.GetAge()
      
  }
  
  // 上面那块代码虽然是能直接执行的  !!!! （因为语法糖， 而不是 生成了对应的发放）
  //但是如果我们定义了 interface 类型， 我们的 p 变量是没有拥有 *p 的方法的。 虽然  *p 包含了 所有 p 的方法
  // p 能自动生成*p 的方法， 但是 *p 的方法不能自动生成 p。
  ```

​     ![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/img/20211003170536.png)

*  不要把 interface 和nil 对比弄混了！！！interface 的对比除了值以外，还要对比type

​     ![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/img/20211012182635.png)



* map 的两种get

  ```
  v, ok := m1["name"]
  // ok 主要告知到底有没有这这个key
  // v 取不到值就是默认值，和ok 本身并没有太大关系 （断言的时候 ok 可以安全断言，但是map 取值的时候，如果不想知道到底有没有这个key， 这个ok 可以直接忽略）
  
  ```




* 发现引用类型 ,如果是 nil 也能调用对应 结构的方法，只是不能调用对应结构 中的字段

  ```
  type Name struct {
    N1 string
  }
  
  func (n  *Name) Get() string{
     if n == nil {
      return ""
     }
     
    return n.N1
  }
  
  var t *Name
  
  t.Get() 是正常执行的，我们再调用字段的时候，用 t.Get() 可以省去判断 t 是否是nil
  
  ```

  



* channel 的读取

  ```
  c, bool := <-chan
  
  注意，channel中有元素的时候，不管 channel 是否关闭，都是true， 其实就是标志这个数据是不是init data（这块可以结合源码看一下）
  
  就是这个标志位不会实时根据channel 是否关闭，而是在读完所有数据之后，如果关闭了，channel 不会阻塞，可以返回值，否则是阻塞。
  
  
  // 代码中我们经常要做到通知 关闭的功能
  // 之前 我们是向一个 作用关闭的channel 中塞入一个信号，当我们从中获取到 信号的时候，代表功能关闭
  // 其实我们大可不必， 可以直接把这个channel 设置成  <-chan struct{}, 当这个chan 不堵塞的时候，代表这个chan 关闭了
  ```



* 断言

  ```
  t, _ := m1.(int) // 安全断言，通过 comma 判断断言是否正确，如果断言错了， t 是该类型零值
  // 如果不用ok， 断言错了，会panic
  ```

  

* map 是线程安全的吗

  ```
  不是，属于临界资源，没有加锁。
  只有一个标志位， 并发读写的时候会panic （throw 错误）。
  
  channel 是线程安全的，他的struct 结构体中包含一个sync.Mutex 锁
  
  slice 也非线程安全
  string 也非线程安全。
  int 这些但凡没有 锁的都是线程不安全，需要用 atomic包。（复习一下 atmoic）
  ```

  

* map 的遍历过程  (不明朗，看下map 的底层数据结构，看下map 的for range 代码)

  ```
  随机数--》任意bucket (index) 下面 任意 元素 开始遍历-- 》key(8个) -》overflow -》查询value
  ```

  

* map 中key 无序

  ```
  开始遍历的 bucket 不一定从哪个开始
  ```

  

* float 类型可以作为map key吗 

  ```
  go 中float 好像一直有精度问题，所以我们在比较的时候一定要注意
  
  可以，但是float 存在精度问题
  ```

  

* map 的赋值过程是怎么样的 （不明朗）

  ```
  并发检测 --》是否正在扩容（rehash过程中）--》查询key -》是否hash 冲突--》overflow查询--》count（包含 key 的总数）修改--》是否要overflow
  ```

  

* Map 怎么实现顺序读取

  ```
  第一想法就是用slice 存储 key.
  
  但其实还要加一个sort 排序， 让 key 的顺序固定住
  
  for range  key ===> 从map 中取数
  ```

  

* go 引用类型

  ```
  nil 是值，代表初始值。这个变量如果是 nil 还是能取到地址的
  
  map   （可以直接在函数内部修改，但是感觉这样写容易出bug， 我们的func 作用是修改map， 感觉容易被忽略，感觉不如把值返回出来）
  slice （原则上讲还不是，只是结构体中包含一个指向数组的指针）
  channel  （所以channel 可以在各个func 中传递， 让并发编程更加方便）
  
  这三个我们一定要 make 初始化之后使用，否则会 panic （slice 的 append 倒是不会）
  
  
  
  
  func 
  
  interface // 确实是引用类型，但是我们想做到引用传递，必须得是 * 这种类型type
  
  https://zhuanlan.zhihu.com/p/105554073
  
  上面这篇文章写得还是不错的，虽然和 当前题目感觉没有多大关系
  ```

  

* go interface 需要解决的坑

  ```
  我们在返回类型的时候，返回 具体的类型，而不是返回interface。 
  
  上面写的有问题， error 就是interface。
  
  就是想表述的是 某些实现了interface 的字段，即使自身是 nil，转换成 interface 后，和 nil 相比都是不相等的。
  
  
  
  因为返回interface 的话，我们很难判断是否是nil， （因为 interface 的 type 肯定不是nil 了）
  我们需要把他的值拉出来判断是否是 nil。（通过反射拿出来不太现实）
  
  b站的 error ，相判断二者是否相等，就得使用自身实现的 cause 方法
  
  ```

  

* go interface

  ```
  没有方法的 eface
  
  type + data (unsafe pointer)
  
  有方法的 iface
  
  (itab (inter + _type  + 方法集合的第一个元素地址，便于寻找 (
  methods := (*[1 << 16]unsafe.Pointer)(unsafe.Pointer(&m.fun[0]))[:ni:ni]
  所以可以用 fun [1]uintptr 去存储所有方法，其实有一大片连续的地址空间
  ))  + data (unsafe pointer  ))
  
  ```

* go 除了 mutex 锁意外还有那些方式安全读写共享变量

  ```
  channel (sync.mutex -> atomic)
  
  atomic 包 （sync.Mutex 底层调用了atomic）
  
  感觉本质上都是调用atomic 去控制并发
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
  
  内存逃逸从栈上逃逸到堆上 
  1.地址类型的返回，
  2.或者大的数据类型
  3.interface类型
  
  gc 回收的是堆上的对象。
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
  
  解决办法，其实就是上面说的共享变量的访问
  
  1.sync.Mutex
  2.atomic (cas compare and swap)
  3.channel 去解决 （底层是加锁了）
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

* 并发和并行的区别

  ```
  并行是真正意义上的同时执行
  
  并发 本质上是轮流执行，速度太快，外界看上去是同时执行 (时间片的切换。轮流执行)
  ```

* 怎么查看goroutine 的数量

  ```
  runtime 里面有个变量
  
  goroutine id ， 黑科技，go 官方不希望给出来。（不希望 goroutine 级别的信息存储，不容易回收。goroutine 数据不能主动回收 和 结束， 是通过 gc 来进行回收）
  ```

* 无意间看到 string 并发的坑 (string 底层也是一个结构体， len 和 data 不一致)

  ```
  https://segmentfault.com/a/1190000023283854
  
  万物都是线程不安全的。
  
  线程安全的 channel， sync.Map (mutex 实现)
  atomic.Value （atomic 实现）
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

* 对 三种类型的channel 操作

  ```
  1. 未初始化  （send, nil channel）
  2. close channel  ( send, panic  close  close channel， panic) 是否close ，有个标志位 closed
  3. active channel (正常)
  
  
  channel 的 读取
  
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
  
  如果 close了，不阻塞， 读取不到值。
  
  
  3. for {
     select
  }
  
  同第一种方案。
  
  
  这种正常情况下没有问题， channel 读取不出来数据会被阻塞，但是当channel close 之后，会一直能不间断的读取到数据，需要注意，一定要判断 value, comma := <- chan  ,判断这个数据是否是默认值。
  
  ```

* Var 定义的变量 （make , new ,  var , :=  对比）

  ```
  都是未初始化的
  
  chan  slice channel （指针类型）（零值初始化） 都是 nil ，不能用。 必须得 make
  
  但是对于 struct int，的初始换 （准确说是零值初始化），可用。 
  
  感觉类似于 new (var 的对比， 或者自身 的 := 赋值)， 只不过返回了 地址类型。
  
  var :=   ==========> new =  &(var, :=), make  对指向的底层变量初始化。
  
  new 和 make 都会生成内存空间
  ```

  

* go 的gc机制

  ```
  计数法和标记染色法。
  
  从root节点到达的第一批节点标记为灰色，灰色上升黑色，黑色节点到达的节点标记为灰色。重复上述步骤。
  
  因为存在并发，所以引入了stw。
  
  为了减少stw，在生成新的节点和 节点修改引用的时候都会标记成灰色。
  ```

  

* 链表是否有回环

  ```
  快慢指针，最后是否相遇
  
  ```

* go 的引用类型有哪些


```
https://blog.csdn.net/love666666shen/article/details/99882528

这篇文章对值类型和引用类型有很好的解释

值类型： int float， bool， string (不可变类型)， 数组， struct

引用类型 ：slice, map, interface （感觉就是能否用nil 表示）, chan （多个函数内部传递, 虽然是值传递，但架不住引用类型）, func

值类型的特点是：变量直接存储值
引用类型的特点是：变量存储的是一个地址
```


* go 的select 的default 的作用

  ```
  当 select 中的其他条件分支都没有准备好的时候，`default` 分支会被执行。
  为了非阻塞的发送或者接收，可使用 default 分支：
  ```

  

* Go 变量逃逸

  ```
  变量原先分配在栈上。逃逸到堆上。
  
  比如函数内的局部变量，返回了该局部变量的地址，变成野指针。
  
  interface 类型的函数，编译的时候没办法确定具体类型，只有执行的时候才知道 （鸭子类型，像动态语言）
  
  chan *int, 这种元素没办法确定以后用在哪，会造成变量逃逸
  
  ```

* go 的 defer， （结合面试题看）

  ```
  https://segmentfault.com/a/1190000006823652 （核心就是defer 的传入参数是 定义函数的时候决定，defer 内部的时候参数 是执行决定）
  
  https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.4.html (这个有个需要注意的是变量的作用域，最近的作用域 ， 比如 for range 中 给 goroutine 添加参数， v:=v 这样)
  
  https://learnku.com/articles/42255
  
  defer 的执行要学会转换
  
  核心
  
  1.return 定义的内容 和  函数内部内容是同一个函数空间 （如果定义了返回参数的名称，则这个变量应用空间是整个函数）。 defer 的参数如果是 外部传入的，变量值维持定义时候的状态，但是他也不会影响外部变量的状态。
  2.return 的赋值先执行，再执行defer ， 再执行 函数结束指令 (return 的执行不是原子性)
  3. defer 如果是一个方法，那就是匿名方法，匿名方法的变量如果传递进去，就走传递进去的值。 如果是引用外面的，就走实际执行到那会的值。
  4. fmt.println() 看做一个func 的执行，参数是传进去的
  ```

  ```
  example 
  
  i := 1
  defer fmt.Println("Deferred print:", i)
  i++
  fmt.Println("Normal print:", i)
  
  Normal print: 2
  Deferred print: 1  // println 传入了函数
  
  
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
      f1()
  }
  
  
  func f1() (r int) {
      defer func() {
          r++  // go 和 php 不一样，这样不能直接使用，和上面返回的r 是同一个变量
      }()
      return 0
  }
  func main() {
      fmt.Println(f1()) // 1
  }
  
  
  func f() (r int) {
       t := 5
       defer func() {
         t = t + 5 // 5
       }()
       return t
  }
  
  
  func f() (r int) {
      defer func(r int) {
            r = r + 5  // 局部变量
      }(r)
      return 1
  }  // 1
  ```

  

* 两个goroutine 怎么控制先后顺序

  ```
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

* channel 被关闭还能读出值么？ 多次读的时候会返回什么

  ```
  channel被关闭后是可读的，直到数据读完为止。
  被关闭后，读取的如果是已经存在的值，注意标志位是true(想象一下，如果关闭之后直接就读取到false，那就没法判断读到的0 值到底是 真实数据， 还是 close 后给的默认数据)
  如果继续读数据，得到的是零值(对于int，就是0)， 两个返回值，另一个是 false
  
  
  channel 被关闭 可以一直读取。 比如利用 for 循环，关闭后，for 读取不会被阻塞。
  
  for range 读取不到值，且不会被阻塞。
  
  channel 没有被关闭，读取会被阻塞。 for  n 循环 和 for range 都会被阻塞。
  ```

* 对一个关闭的channel 写入呢，返回值是什么类型. 关闭一个已经关闭的channel

  ```
  channel 会 pannic
  
  panic: close of closed channel
  ```

* go 语言并发模型 (CSPCommunicating Sequential Process )

  ```
  以通信的方式来共享内存。
  用于描述两个独立的并发实体通过共享的通讯 channel(管道)进行通信的并发模型
  
  Golang中channel 是被单独创建并且可以在进程之间传递，它的通信模式类似于 boss-worker 模式的，一个实体通过将消息发送到channel 中，然后又监听这个 channel 的实体处理，两个实体之间是匿名的，这个就实现实体中间的解耦，其中 channel 是同步的一个消息被发送到 channel 中，最终是一定要被另外的实体消费掉的，在实现原理上其实类似一个阻塞的消息队列。
  
  Goroutine 是Golang实际并发执行的实体，它底层是使用协程(coroutine)实现并发，coroutine是一种运行在用户态的用户线程，类似于greenthread，go底层选择使用coroutine的出发点是因为，
  
  它具有以下特点：
  
  用户空间 避免了内核态和用户态的切换导致的成本.
  可以由语言和框架层进行调度.
  更小的栈空间允许创建大量的实例.
  
  
  Golang中常用的并发模型有三种:
  
  1.通过channel通知实现并发控制 (channel 读取阻塞，防止程序没有执行完)
  2.通过sync包中的WaitGroup实现并发控 （waitgroup 的坑，之前遇到过，他本身是一个struct, 所以如果在另一个方法中 done， 一定要传入地址类型，否则不是修改自身）（waitgroup 底层也是通过 atmoic 包修改 计数信号量）
  3.在Go 1.7 以后引进的强大的Context上下文，实现并发控制.（errgroup 的接受到一个 错误就返回，用到的就是遇到一个error  sync.Once执行一次 ctx 的 cancel() 方法，我们可以通过各个 goroutine 中 select ctx.Done() 获取取消的channel 信号 ）
  ```

* GMP 应该是属于调度模型

  ```
  G goroutine
  
  M machine 实际执行的线程
  
  p process 逻辑
  
  gobal 队列中取g， 会造成严重的锁竞争，也会失去 cpu亲和性。
  
  生成多个本地队列 runtime设置大小 ，每个本地队列上绑定一个p， m 和 g 直接通过p 关联， m通过p 取本地队列的g （本队队列最大256，如果取完了从global 队列取，如果也没有，去本地队列中偷取一半）。
  
  g的堵塞 （io 堵塞，channel 等待）。都不会切换m， 比如 io 堵塞会把 g 放到netpoll 中，类似多路复用，如果数据准备好，又会回到队列尾部等待执行。
  
  syscall 导致的堵塞， g 和 m 脱离 p， 执行完，g 回归队列，m 回到空闲 m 队列。
  ```

* 空slice 和 nil slice

  ```
  make([]int, 0) // empty slice
  var []int // nill slice
  ```

  

* go 的缓冲channel 和 单个channel 有什么区别

  ```
   无缓冲： 不管是存值还是取值，都需要对应的消耗者或者提供者，否则容易造成dead lock
   有缓冲： 不会阻塞，因为有缓冲池，只有当缓冲池满了，还往里面塞，才会发送阻塞。
           或者空了，读取阻塞
   
   无缓冲的chan 发送和接受同步
  ```

* go 中两个nil 可能不相等吗

  ```
  想到interface 
  
  interface  == nil，  不止要 value  == nil  还要  type == nil
  
  但是 struct 的地址类型 等一些引用类型，  这种 == nil， 只需要 value  == nill。
  
  struct 属于零值可用，不会等于 nil
  
  //
  type A interface {} // iface
  
  var z A  
  z == nil (true) // _type  没有绑定实际的变量
  
  var z1 interface{} // eface
  
  z == z1 (true)  // runtime 的 efaceeq  和 ifaceeq 做了特殊处理，详见 (_type 为nil ,就返回true)
  
  go tool compile -S xx.go 
  
  
  
  
  // 上面的举例不正确
  var a interface{}
  var c map[int]int
  a = c
  
  fmt.Println(a == nil) (false)
  ```

* go 函数返回局部变量的指针是否安全

  ```
  这个其实是相对于c++ 来说的，c++ 中说这种属于野指针，go 中属于内存逃逸
  
  变量分配从栈上逃逸到堆上
  ```

* Goroutine 发生泄漏如果检测

  ```
  表象： pod 不断扩容
  
  granfa 上pod 内存一直不断上涨, 导致自动扩容。
  
  检测方法 pprof。 启动一个协程，对外抛出一个web 接口，通过这个接口，可以通过curl 拉取文件，再通过ui库展示这个文件，包含 内存，cpu， 协程，堆栈。
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

  

* golang 平滑重启原理

  ```
  https://www.tizi365.com/archives/83.html
  
  
  // 当前的goroutine等待信号量
  quit := make(chan os.Signal)
  // 监控信号：SIGINT, SIGTERM, SIGQUIT
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

* channel 底层数据结构

  ```
  https://segmentfault.com/a/1190000040263156
  
  （buf 只有非阻塞 channel 才会有）循环数组，
  
  sendx
  
  recqx (发送 和 接受指针)
  
  两个双向链表 （一个send 生产者grooutine list， 一个 consumer 消费者 goroutine list），保证了相进相出。
  
  close 状态
  
  sync.Mutex 互斥锁，防止并发。
  
  ```

  

* golang 的 context 的 withcancel 用过么，什么场景

  ```
  withcancel ，
  withtimeout 
  withdeadline 这些都会返回一个新的context 和一个主动触发的cancel ，我们通过 监听 ctx.Done() 这个chan， 在需要的时候，比如超时设置，获取到 chan 中msg， 主动释放资源 （goroutine 不能通过外力去结束，只能在自己的代码中设置对应的条件去结束）
  ```

* Goroutine 变量作用域的问题, 在循环中调用goroutinue修改变量，传递的变量会改变吗？如何优化

  ```
  循环中调用goroutinue，并在协程中打印value:value 值不确定。
   解决方案：（1）value作为参数传递给goroutinue func(a int, b int)， 传进去
            （2） 在循环中新创建变量， 比如 v:= v
  
  ```

* Golang swith 和 php switch 区别

  ```
  （1）go中加上了默认break，匹配到对应case，在执行完相应代码后就会退出整个
      switch 代码块
  （2）go中用fallthrough关键字继续执行后续分支的代码
   （3）go 中 switch  type 可以用于获取字段类型
  ```

* go 怎么调度 goroutine

  ```
  gmp 调度模型。
  ```

  

* go waitgroup (看一下源码)

  ```
  https://juejin.cn/post/6844904078653292551
  
  Add()：每次激活想要被等待完成的goroutine之前，先调用Add()，用来设置或添加 count
  例如Add(2)或者两次调用Add(1)都会设置等待计数器的值为2 
  Done()：每次需要等待的goroutine在真正完成之前，应该调用该方法来人为
          表示goroutine完成了，该方法会对等待计数器减1 (add -1)
          我们也可以主动写 sync.WaitGroup.Add(-1) // Add(-count), 直接唤醒goroutine, 但 会导致panic, 别的程序执行 done() 的时候。
          
  Wait()：在等待计数器减为0之前，Wait()会一直挂起 （阻塞） 当前的goroutine （当请求计数器为0时，会唤起所有等待的goroutine）
      
  // done 和 add 不匹配的时候 会panic， 底层是 通过计数的方式
  // atomic ， 计数方式
  
  ```

* new 和 make 的区别

  ```
  https://sanyuesha.com/2017/07/26/go-make-and-new/
  
  都是内存分配。
  1.make 用在 slice， map， channel， 初始化非零值（nil），返回的就是他们本身 （slice 结构体，平时我们用的时候是指向底层数组第一个元素的指针， map 是 *hmap， channel 是 *hchanel），但会对他们的内部元素进行初始化（上面三个地址类型 零值都是nil）
  （slice 零值的时候不能 [0] 这样赋值， 但是可以append。 map 因为没有append 的操作，所以啥也不能做， 但是 map 和 slice 都是可以 []int{}, map[string]{} 这样直接给值的。）
  2.new 是对应类型零值元素的地址。new([]int) , 通过 reflect 可以看到的是 *[]int 类型
  ```

* go 的init 用过么

  ```
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

* go 的 map怎么删除元素

  ```
  delete 
  ```

* map 底层原理

  ```
  hashtable
  
  hash 表 + 链表 （hash 碰撞，溢出超过8个 kv 对的时候，bmap 组成链表，平时是数组）
  
  tips: hash函数,有加密型和非加密型。加密型的一般用于加密数据、数字摘要等，典型代表就是md5、sha1、sha256、aes256 这种,非加密型的一般就是查找。(所以查找这种我们应该用的比较少，他一般生成的就是 index)
  
  在map的应用场景中，用的是查找。
  
  选择hash函数主要考察的是两点：性能、碰撞概率。
  
  扩容机制， hash 因子 > 6.5 (hash 因子 = 键值对 / bucket 的个数， 渐进式hash)
  
  核心数据结构 hmap 和 bmap
  
  hmap 就是 map 的struct 结构
  
  bmap 就是我们说的bucket
  
  默认情况下用不到链表，是通过连续数组的偏移去寻找 key value 的 （超过8个就得走链表结构了， bmap 里面overflow 里面查找）
  
  https://segmentfault.com/a/1190000039101378
  https://juejin.cn/post/6844903940866179079
  ```

  ![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/img/20211001161838.png)

* go 的 sync.map (sync. Map 原理， 读写分离)

  ```
   1.map在并发编程中，读是线程安全的，写不是
   2.sync.map是线程安全的，不需要初始化，声明即可
   3.Store 表示存储，Load 表示获取，Delete 表示删除
   
   核心 read map（read only）  dirty map， 标志位
   
  https://juejin.cn/post/6844903895227957262#heading-5
   
  
   和 sync.Mutex 区别 。(sync.Map 是 互斥锁)
   相同处都是 表级锁，
   区别。sync.map 并发读， 能读取到，而且 dirty 和 read 一致的时候不用加锁。通过miss 计数，把dirty 同步到read 中，避免加锁。
  ```

* go 中线程安全和线程不安全的元素

  ```
  线程安全 ： channel， 就是用来协程间同步数据的。
  
  线程不安全：
  string ， 不可修改， 防止 copy on write （上面有个例子，讲的是string 又读又写出现的问题）
  
  int， float， 都不行 所以出了 atomic 包， 需要我们去自行处理。
  
  map, slice, 这些 在读写并发时候会有问题， 
  
  struct 对于相同field 并发读写会有问题，不同field 并发读写没问题.(因为不同的field 存在于不同的内存地址)
  
  map 有个好处是并发会panic.
  
  ```

* go的并发模型 （golang 如何调度goroutine）

  ```
  GMP （runtime 怎么调度 goroutine 的执行的）
  
  CSP (并发模型三种)
  
  CAS （原子性操作）
  
  ```
  



* 开发的流程规范是什么

  ```
  1.产品提需求
  2.需求移交
  3.技术评审
  4.排期
  5.开发
  ```

  

* 半连接是什么

  ```
  全双工链接没有建立成功。三次握手，只是握了2次
  ```
  
* 粘包是什么，怎么发送的

  ```
  1.tcp 本质上是流的存在，不存在粘包，粘包，是上一层协议的解析导致
  2.由于存在缓冲区（发送缓冲区， 凑足整个buf）一次性发送，可以通过关闭这个设定。但由于接受缓冲区的存在，还是有粘包问题，我们可以通过类似content-length, 获取到整个包的长度。或者通过某个特殊字符进行区分 （但会存在二进制不安全问题）。
  ```

  

* 怎么创建索引

  ```
  alter table add index
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

* 简要描述下变量逃逸

  ```
  https://www.cnblogs.com/itbsl/p/10476674.html
  ```

* 简要描述下slice 在 apend时发生了什么

  ```
  是否有 空间存储， 是否要扩容。
  修改len， 修改 cap
  ```

* 给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那两个整数，并返回他们的数组下标

  ```
  遍历，存储到map 中
  ```

* 给定两个数组，编写一个函数来计算它们的交集、以及差集（交集简单，重点在差集上）。leetcode 上只讲了交集没讲差集
  如输入 nums1 = [4,9,5], nums2 = [7,5,10,8,4]
  交集返回 [4,5]
  差集返回 [9],[7,10,8]

  ```
  用map 。交集 时间复杂度 o(n)， 空间复杂度o(n)
  
  差集 。也可用map。 o(n)
  ```
  
* 简单谈下 chan 的应用场景及注意事项

  ```
  控制goroutine 的执行顺序，goroutine 通信。
  ```

* 往一个对象里写入 10w 条数据，怎样保证数据的准确性（chan、mutex 之类的胡扯就对了）

  ```
  atomic 自增
  sync.Mutex 互斥锁
  ```

* 闭包的注意事项

  ```
  （这里变量的引用是个坑）
  ```

* 简单介绍下 interface 的应用场景（我给扯出什么 2.0go 打算加入泛型什么的，实际上面试官想问的是接口不是 interface {} 这个数据类型…, 等面试官给我纠正了下思路然后，我又继续讲接口相关的东西…）

  ```
  爬虫。interface定义规范。
  ```

* For  range 容易引起的问题

  ```
  这个最初遇到导致的问题就是在一次 循环增加 goroutine 的时候，碰到了意外的情况。
  
  go 官方也是 go func 引用的变量不被外来影响，两种方式
  
  1.函数传递进去， func(var)
  2.利用函数包装，临时保存 ， func () {
     v := v
  	go func() {
  	  fmt.println(v)
  	}
  }
  
  我觉得本质上是2 个问题导致
  1.go func 不能马上执行，和 闭包是同样的道理，一个间隔的时间长，一个间隔时间段
  2.for range 声明的变量v 只声明了一次， 循环赋值，所以这个变量实际存储的值一直在改变
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

  

* 需要注意对象new 只是初始化，会把一切类型都赋值成 ’零‘, 但 map  channel 都是 零值不可用的， slice 在append 的时候是零值可用的， 在 直接给值的时候，比如 a[0], 非零值可用

  ```
  map 往空里面写是 nil
  
  channel 往空里面写 也是报 nil 错误
  
  slice ，append 不出错很神奇。 但直接取值就会报错了。
  
  
  new 用在 slice， map 和 channel 都是返回对应的地址，都没法用了。
  
  new(map[int]int) // 无法直接赋值
  new(chan int) // 无法往里面塞入数据
  new([]int) // 无法append
  
  
  一般只能用在 struct 上
  
  ```
  
  


* 感觉用 new 和 直接 struct 生成没啥区别 （有区别，一个是地址类型，一个是原始类型）

  ```
  new 和 make 都开辟了内存空间，哪怕 pointer 类型变量零值初始化之后是 nil
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

* recover 不能跨协程，只能在当前协程中捕捉，但是把这个捕捉函数独立出去是没问题的, 如下，就是不能在协程之外捕获

  ```
  func T1() {
  	if err := recover(); err != nil {
  		fmt.Println("recover: ", err)
  	}
  }
  
  func main() {
   defer T1()
  
  }
  
  // 主要是defer 函数不能在panic 的函数外面.
  
  // 下面这种没办法捕捉到panic
  func T2() {
   defer func() {
  	if err := recover(); err != nil {
  		fmt.Println("recover: ", err)
  	}
   }
  }
  
  func main() {
    T2()
  }
  ```

* 原来map 还可以这么初始化，目前看到只有chan 离开 make 不能活了

  ```
  type Student struct {
  	name string
  }
  
  func main() {
  	m := map[string]Student{"people": {"zhoujielun"}}
  	m["people"].name = "wuyanzu"
  }
  
  
  map的value本身是不可寻址的，因为map中的值会在内存中移动，并且旧的指针地址在map改变时会变得无效。故如果需要修改map值，可以将map中的非指针类型value，修改为指针类型，比如使用map[string]*Student.
  
  负载因子过大，会有hash 变动的过程，导致元素改变
  ```

* 下面这代码是真的恶心，就是为了考验 for range 的时候 v := v, 作用域

  ```
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

  

* 写出下面代码的输出内容

  ```
  package main
  
  import (
  	"fmt"
  )
  
  func main() {
  	defer_call()
  }
  
  func defer_call() {
  	defer func() { fmt.Println("打印前") }()
  	defer func() { fmt.Println("打印中") }()
  	defer func() { fmt.Println("打印后") }()
  
  	panic("触发异常")
  }
  
  defer 关键字的实现跟go关键字很类似，不同的是它调用的是runtime.deferproc而不是runtime.newproc。
  
  在defer出现的地方，插入了指令call runtime.deferproc，然后在函数返回之前的地方，插入指令call runtime.deferreturn。
  
  goroutine的控制结构中，有一张表记录defer，调用runtime.deferproc时会将需要defer的表达式记录在表中，而在调用runtime.deferreturn的时候，则会依次从defer表中出栈并执行。
  
  因此，题目最后输出顺序应该是defer 定义顺序的倒序。panic 错误并不能终止 defer 的执行。
  
  (这题的关键不是输出内容，而是说出了 return 在 defer 之前触发)
  （因为没有加recover， 所以后面panic 还是会报出来）
  ```

* for range 问题

  ```
  type student struct {
  	Name string
  	Age  int
  }
  
  func pase_student() {
  	m := make(map[string]*student)
  	stus := []student{
  		{Name: "zhou", Age: 24},
  		{Name: "li", Age: 23},
  		{Name: "wang", Age: 22},
  	}
  	for _, stu := range stus {
  		m[stu.Name] = &stu
  	}
  }
  
  golang 的 for ... range 语法中，stu 变量会被复用，每次循环会将集合中的值复制给这个变量，因此，会导致最后m中的map中储存的都是stus最后一个student的值。
  ```

  

* 这个调度任务看不懂

  ```
  func main() {
  	runtime.GOMAXPROCS(1)
  	wg := sync.WaitGroup{}
  	wg.Add(20)
  	for i := 0; i < 10; i++ {
  		go func() {
  			fmt.Println("i: ", i)
  			wg.Done()
  		}()
  	}
  	for i := 0; i < 10; i++ {
  		go func(i int) {
  			fmt.Println("i: ", i)
  			wg.Done()
  		}(i)
  	}
  	wg.Wait()
  }
  ```

* go 中类似 php的语法

  ```
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

  

* 这个defer 的顺序真的厉害 (命中了defer 的一个点就是defer 参数定义的时候执行)

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
  
  ```

* 需要总结一下什么时候结构体和指针能互相转换，什么时候不可以

  ```
  编译不通过
  package main
  
  import (
  	"fmt"
  )
  
  type People interface {
  	Speak(string) string
  }
  
  type Student struct{}
  
  func (stu *Student) Speak(think string) (talk string) {
  	if think == "bitch" {
  		talk = "You are a good boy"
  	} else {
  		talk = "hi"
  	}
  	return
  }
  
  func main() {
  	var peo People = Student{}
  	think := "bitch"
  	fmt.Println(peo.Speak(think))
  }
  
  一个T类型的值可以调用为*T类型声明的方法，但是仅当此T的值是可寻址(addressable) 的情况下。编译器在调用指针属主方法前，会自动取此T值的地址。因为不是任何T值都是可寻址的，所以并非任何T值都能够调用为类型*T声明的方法。
  反过来，一个*T类型的值可以调用为类型T声明的方法，这是因为解引用指针总是合法的。事实上，你可以认为对于每一个为类型 T 声明的方法，编译器都会为类型*T自动隐式声明一个同名和同签名的方法。
  哪些值是不可寻址的呢？
  
  字符串中的字节；
  map 对象中的元素（slice 对象中的元素是可寻址的，slice的底层是数组）；
  常量；
  包级别的函数等。
  
  所以上面那个 var s = Student{} 可以执行 speak 方法，但是 不能 声明成 people， 是因为 t 结构方法 会默认生成 *t， 但反过来不会
  
  之前有种说法是。 *T 的方法，可以直接修改比如这个结构体的field， 验证了下是错误的概念。
  ```

* interface 和 nil 的区别

  ```
  ```

* 交替打印数字和字母

  ```
  使用两个 goroutine 交替打印序列，一个 goroutine 打印数字， 另外一个 goroutine 打印字母， 最终效果如下：
  
  package main
  
  import (
  	"fmt"
  	"sync"
  )
  
  func main()  {
  	var g sync.WaitGroup
  
  	g.Add(2)
  
  	c1 := make(chan struct{})
  	c2 := make(chan struct{})
  
  	go func() {
  		start := 'A'
  
  		for {
  			select {
  			case _, ok := <-c2:
  				if start > 'Z' {
  					close(c1)
  					g.Done()
  					break
  				}
  
  				if !ok {
  					break
  				} else {
  					fmt.Println(string(start))
  					start++
  					fmt.Println(string(start))
  					start++
  					c1<-struct{}{}
  				}
  			}
  		}
  	}()
  
  
  
  	go func() {
  		start := 1
  			for {
  				select {
  				case _, ok := <-c1:
  					fmt.Println(ok, "pppp")
  					if !ok {
  						g.Done()
  						goto Here
  					} else {
  						fmt.Println(start)
  						start++
  						fmt.Println(start)
  						start++
  					}
  					c2<- struct{}{}
  				}
  			}
  		Here:
  			fmt.Println("done")
  	}()
  
  	c1 <-struct{}{}
  
  
  	g.Wait()
  
  	fmt.Println("end")
  }
  ```

  

* 判断字符串中字符是否全都不同

  ```
  请实现一个算法，确定一个字符串的所有字符【是否全都不同】。这里我们要求【不允许使用额外的存储结构】。 给定一个string，请返回一个bool值,true代表所有字符全都不同，false代表存在相同的字符。 保证字符串中的字符为【ASCII字符】。字符串的长度小于等于【3000】。
  (这题目主要恶性的点在于不能额外存储)
  // 核心就是 strings.count
  // strings.index, strings.lastindex
  
  go 获取 ascii 特别容易，不管是 rune （int64） 还是 byte (int8) 直接赋值，都是 int 类型，想转换成 string， 直接强制转换 string()
  
  // 
  第一个方法使用的是golang内置方法strings.Count,可以用来判断在一个字符串中包含的另外一个字符串的数量。
  挨个检查
  
  第二个方法使用的是golang内置方法strings.Index和strings.LastIndex，用来判断指定字符串在另外一个字符串的索引未知，分别是第一次发现位置和最后发现位置。
  // for range 循环遍历
  
  ```

* 字符串翻转

  ```
  package main
  
  import (
  	"fmt"
  	"strings"
  )
  
  func main()  {
  	astr := "ss345"
  
  	//astr = "ssss2222"
  
  	a := strings.Split(astr, "")
  
  	c := len(a)
  
  	tmp := c/2
  	for i :=0 ; i<=tmp-1; i++ {
  		a[i], a[c-1-i] = a[c-1-i],a[i]
  	}
  
  	end := strings.Join(a, "")
  	fmt.Println(end)
  }
  ```

* 判断两个给定的字符串排序后是否一致

  ```
  // 直接判断的两个字符串中所有字符是否一致
  // 核心就是 strings.count
  func isRegroup(s1,s2 string) bool {
  	sl1 := len([]rune(s1))
  	sl2 := len([]rune(s2))
  
  	if sl1 > 5000 || sl2 > 5000 || sl1 != sl2{
  		return false
  	}
  
  	for _,v := range s1 {
  		if strings.Count(s1,string(v)) != strings.Count(s2,string(v)) {
  			return false
  		}
  	}
  	return true
  }
  ```

* 字符串替换问题

  ```
  请编写一个方法，将字符串中的空格全部替换为“%20”。 假定该字符串有足够的空间存放新增的字符，并且知道字符串的真实长度(小于等于1000)，同时保证字符串由【大小写的英文字母组成】。 给定一个string为原始的串，返回替换后的string。
  
  // 关于字母，可以转换字符串成为  []rune, 
  // for range 字符串的时候，即使我们没有手动转换类型成 []rune, for range 也会帮助我们转换。  判断每个字符的 int 大小，在不在 大小写字母范围内
  // replace strings.replace
  ```

* Chan 读取的方式有两种， 一种 select 形式，需要手动监听 _, oK := <-tmp, ok ,退出for select 循环 （注意break 退出不了，要么 return， 要么 goto）

  一种 for range 手动关闭 chan



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

* Time ticker 和 timer 的区别

  ```
  ticker  类似 循环执行
  timer 需要手动重置间隔时间，否则会中断，只执行一次
  ```

* 为 sync.waitgroup 中wait 函数 支持 wait timeout 功能

  ```
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
  ```

* 写出下面打印的内容

  ```
  package main
     import "fmt"
     const (
         a = iota
         b = iota
     )
     const (
         name = "menglu"
         c    = iota
         d    = iota
     )
     func main() {
         fmt.Println(a) // 0
         fmt.Println(b) // 1
         fmt.Println(c)  // 1
         fmt.Println(d)  // 2
     }
     
     iota 在一个const 片段中会重置
  ```

  

* go 中 slice， map ，function 不可以比较 （也就是地址类型不能比较呗, 也不对 chan interface 就可以比较）

  ```
  http://bazingafeng.com/2017/06/04/slice-how-to-work-in-go/
  
  这篇文章写得很好，核心2个
  //  [:9],这种操作，使用的是切片的cap， 只有超过 cap 才会panic， 否则可以使用到 slice 看不到的元素
  // slice 不能比较的原因 。slice之间之所以不能进行比较，是因为slice的元素是间接引用的。slice引用的底层数组的元素随时可能会被修改，即slice在不同的时间可能包含不同的值，所以无法进行比较。
  // 如果真要比较用 reflect.Equeal
  
  
  go 语言中可以使用反射 reflect.DeepEqual(a, b) 判断 a、b 两个切片是否相等，但是通常不推荐这么做，使用反射非常影响性能。
  
  通常采用的方式如下，遍历比较切片中的每一个元素（注意处理越界的情况）。
  
  func StringSliceEqualBCE(a, b []string) bool {
      if len(a) != len(b) {
          return false
      }
  
      if (a == nil) != (b == nil) {
          return false
      }
  
      b = b[:len(a)]
      for i, v := range a {
          if v != b[i] {
              return false
          }
      }
  
      return true
  }
  ```

* struct 是能比较的，但如果包含不能比较的元素，就不能比较了

  ```
  https://juejin.cn/post/6881912621616857102
  // 这篇文章讲的很好。
  // 注意结构体的field 顺序得一致，要不然也不一样，感觉像是内存对齐的问题
  // 注意 struct 的类型一定要一致呀
  
  // 下面这种就没法比较大小
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

* 常量不可寻址

* go 的 map key ---》 value 是不可寻址的

  ```
  https://darkreunion.tech/2020/06/the-value-type-struct-of-map-cannot-assign/
  所以记住不能随便赋值哦
  =  这个能执行的关键是左边能寻址
  ```

* interface 值接收者方法和指针接收者方法

  ```
  https://qcrao91.gitbook.io/go/interface/zhi-jie-shou-zhe-he-zhi-zhen-jie-shou-zhe-de-qu-bie
  
  核心就是
  现了一个接收者是值类型的方法，就可以自动生成一个接收者是对应指针类型的方法，因为两者都不会影响接收者。但是，当实现了一个接收者是指针类型的方法，如果此时自动生成一个接收者是值类型的方法，原本期望对接收者的改变（通过指针实现），现在无法实现，因为值类型会产生一个拷贝，不会真正影响调用者。
  ```

* eface, iface

  ```
  https://www.kancloud.cn/aceld/golang/1958316#1_interface_4
  
  eface 没有 func 的 interface
  
  iface 包含 func 的 interface
  
  就算是 nil 给了 interface{}, 但这个nil 给了 type ，比如 （*struct），这个interface != nil, 因为只有 interface 中 data point 是 nil， 但type 属性已经不是nil 了
  
  ```

* waitgroup的核心就是 add() 和 done() 的匹配

* 下面代码的解释

  ```
  package main
  import (
  	"fmt"
  )
  type Student struct {
  	Age int
  }
  func main() {
  	kv := map[string]*Student{"menglu": {Age: 21}}
  	kv["menglu"].Age = 22
  	s := []Student{{Age: 21}}
  	s[0].Age = 22
  	fmt.Println(kv, s)
  }
  
  https://blog.csdn.net/qq_36431213/article/details/82805043
  // 数组不会rehash ，所以没有什么问题
  ```

* 进程，线程 和 协程的区别

  ```
  https://zhuanlan.zhihu.com/p/70256971
  
  1.进程是程序运行的实体，资源分配的最小单位 （qq 和 网易云同时 运行）
  2.线程类似于程序中的多个任务，cpu 独立运行和独立调度的最小单位， 内核态线程 （网易云 播放歌曲 和 歌词滚动同时运行）
  3.进程拥有自己的资源空间，一个进程包含多个线程，多个线程共享同一个进程的资源
  4.cpu 上跑的任务是线程
  5.协程 ，微线程，占用内存小，2kb（go 协程），可以自动扩容， 用户态线程。 线程栈2m
  对于cpu 来说，他是不知道协程的存在的。
  
  为什么协程切换代价比线程切换代价小？
  1.协程间任务的切换发生在用户态，程序控制，资源消耗小。线程的切换发生在内核态，所以需要用户态到内核态的切换。
  2、而且对于多核情况下，如果协程的切换都发生在一个cpu 上执行，线程的切换在多个cpu上执行， 消耗大。（协程间不需要加锁，因为本质上协程是在一个线程中切换的）https://www.v2ex.com/t/387596​， 协程不用加锁的解释，少场景
  3.协程的切换只需要把 cpu 的寄存器上内容切换到上次执行的地方，线程包含的资源更多，比如协程切换不用管线程共享的栈内存，但线程间切换就得管。
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
  
  
  
* GMP 模型

  ```
  G groutine
  m machine
  p process
  
  g 组成的队列有本地队列（256）和global 队列，通过p 和 m 执行。 
  g 的限制相当于内存/ g的内核消耗，按照2kb 来算。
  p 我们可以看做cpu 的核数，同时并行的任务。
  m 是线程，类似跑在p上面（cpu）， 可以进行线程切换，他的数量一般内核能支撑到1000.
  
  g的队列多个主要是为了防止重复加锁。p 从本地队列里面取g， 如果没有，会去别的队列窃取一半或者取全局队列获取。m阻塞，p会切换到比的m 上执行，防止导致某个g 不被执行。
  ```

  

* go 怎么实现的锁

  ```
  ```

* go 变量逃逸 （栈上是局部变量，何时会逃逸到堆上）

  ```
  https://segmentfault.com/a/1190000039843497
  
  栈内存：函数级别，用来存储自己的局部变量，返回地址，返回值之类的数据。这一块内存区域有特定的结构和寻址方式，寻址迅速，开销很少。
  堆: 线程级别，大小在创建的时候确定，当变量太大的时候，会逃逸到堆上。
  
  go tool compile  -m -l, 是在编译阶段确定。 所以感觉
  
  go run --gcflags="-m -m " xxx.go
  
  arr := make([]int, 0)
  for i:=0; i< 10000; i++ {
    arr = append(arr, i)
  }
  这样不会内存逃逸
  
  但是 
  arr := make([]int, 0, 10000) 会内存逃逸
  
  //
  内存逃逸的危害，堆上元素的回收靠的是gc。gc 压力过大
  
  内存逃逸时机
  
  1。向 channel 发送指针数据。因为在编译时，不知道channel中的数据会被哪个 goroutine 接收，因此编译器没法知道变量什么时候才会被释放，因此只能放入堆中。(这个我试过，不管是阻塞还是飞阻塞都没有逃逸)
  
  2.局部变量在函数调用结束后还被其他地方使用，比如函数返回局部变量指针。因为变量的生命周期可能会超过函数周期，因此只能放入堆中。
  
  3.在 slice 或 map 中存储指针。比如 []*string，其后面的数组可能是在栈上分配的，但其引用的值还是在堆上。
  （ channel， slice 中包含了地址类型，都会出现内存逃逸）
  
  4.切片扩容后长度太大，导致栈空间不足，逃逸到堆上。
  
  5.在 interface 类型上调用方法。 在 interface 类型上调用方法时会把interface变量使用堆分配，编译期间不能确定类型 （fmt 调用的是 interface， 所以 fmt 会导致 内存逃逸）
  
  // 如何避免
  1.对于小型的数据，使用传值而不是传指针，避免内存逃逸。
  2. interface调用方法会发生内存逃逸，在热点代码片段，谨慎使用。
  
  
  //  
  https://mp.weixin.qq.com/s/mFfza7DayFqsiS93Ep15BA
  
  煎鱼大佬从另一个角度的分析
  
  1. 指针 （并不是指针都会逃逸，而是指针才当前作用域内使用，就不会逃逸。所以值类型不会逃逸，因为是copy， 而且会copy on write）
  2. 未确定类型（所以interface 会存在内存逃逸，所以fmt 这种存在内存逃逸）
  3. 泄露参数 （如果指针是从外部传进来，再传出去，就不会存在内存逃逸）
  ```

* go 协程池

  ```
  ```

* 下面会出现什么问题

  ```
  // 两种方式解决
  // k := k  v:=v ,或者通过参数传进去
  m := make(map[int]int, 10)
  for i := 1; i<= 10; i++ {
      m[i] = i
  }
  
  for k, v := range(m) {
      go func() {
          fmt.Println("k ->", k, "v ->", v)
      }()
  }
  ```

* 内存泄露，什么情况下内存会泄露。

  ```
  goroutine 没有得到很好的释放
  
  1.同事封装的超时函
  
  resChannel := make(chan int)
  go func() {
  	resChannel <- 1
  }
  
  deadline := time.After(time.Second)
  
  select {
   case : <-resChannel
  
   case : <-deadline
     return responeseDead
  }
  
  
  通过 select 接受 两个 channel， 当超时后 自动返回超时错误 。
  
  可这样一来没有消费者了，导致channel 生产堵塞，所在goroutine 无法释放，导致协程泄露。
  
  
  2.用的 b站封装的 errgroup 函数，有个设置groutine 数量的，GOMAXPROCES。用了这个方法之后，当我们添加 func 的时候，他是塞入 channel 中， 先通过 for 启动固定数量的协程，然后在协程里面通过 for range 去读取对应的方法
  
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

  

* 外部接口请求很慢，怎么排查

  ```
  通过trace 查看每一部分执行花费的时间。
  
  其实还是比较好奇trace 的执行原理。
  
  1.慢sql ，sql 执行是否过多。sql 执行一般也会监听 ctx.done
  
  2.连接数是否够用，等待连接池分配连接的时间 （之前粉丝勋章 发现在从 redis 或者mysql 中获取数据都要等待好长一段时间）
  
  3.缓存获取。 一般也会监听 ctx.done, 缓存获取的是否是个大内容
  
  4.cpu pod负载是否过高
  ```

* 分布式事务

  ```
  ```

  
