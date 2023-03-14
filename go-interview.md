

```
1.slice  相关问题
make([]int, 5), len是5， 被0元素填充。 ----> [0,0,0,0,0]
只想防止扩容， 应该改成 make([]int, 0, 5) ---> []. len 0, cap 5

slice 三个属性组成，

数组指针.
len 实际占用大小，
cap 容量。



slice 的底层是数组，数组不能扩容，slice 会扩容，扩容后数组地址会变！！！ notice。
```





```

slice append， 如果cap 容量满足，就在原先数组len 后面填充，无论是否已经有元素占领。
容易在用切片的时候掉进坑里。(都是因为切片是 浅 copy， 一般 = 都是公用底层相同的数组)
当前切片的len 会指示当前切片能看到的元素。

slice 切片， []int{}[0:4], 影响自身， 此时并没有生成新的数组，指针只是移动了位置，之前指向数组第一个元素，现在指向的是index开始的元素。修改了 len的值， cap 的值随着 start index 的变化而变化。末尾到原切片末尾

start:end 这种切片append 很容易出现意想不到的效果，给原始数组造成影响。
因为append 填充的元素是针对原始数组的，是否扩容是原始数组的cap 是否满足当前需求, 如果原始数组cap 能容纳，不用扩容，这个append的操作会影响多个切片。
append 元素的位置又是当前切片的 len 的下一个位置。（这个地方是个坑点，append 是len 的下个位置，而不一定是元素为0的位置）

所以append的 slice，最好用copy 函数深度copy下。如下：


  s123 := []int{5}
	s123 = append(s123, 7)
	s123 = append(s123, 9)
	x := append(s123, 11)
	y := append(s123, 12)
	
	fmt.Println(s123, x, y)
	
	// s :[5, 7, 9]. len：3， cap: 4
	// x ： [5, 7, 9, 12]
	// y : [5, 7, 9, 12]
  
  
```

 ```
 切片的扩容,也是经常问的。
 
 
 分版本。
 
 1.18版本之前：小于等于1024 ，都是double， 之后是0.25倍增长
 
 1.18之后，包含。 小于256 都是double， 之后是  (n + 256 * 3)/ 4 的增长
 
 还要进行内存对齐，内存对齐后一般都会多余之前的长度。
 
 特别要注意如果一次性插入多个元素。
 
 一次扩容2倍小于要出入的元素，那就以新的大小为准。
 ```



```
切片作为函数参数。

切片底层是 struct， 和 map ，channel 还不太一样，后面2个都是 struct 的地址。

所以在go 的值传递中，切片自身len ，cap 不会被修改，但因为包含了指向数组的指针， 所以如果底层数组的中的值被修改，作为函数参数的切片也会被修改。

但如果因为扩容，底层数组的地址发生变化，则对底层数组的修改不会反应到函数参数切片上。
```



```
go  for range slice 循环的时候执行 append, 会死循环么

结论：不会
  https://blog.csdn.net/wohu1104/article/details/115496718

  这个for range 其实就是 

  arrCopy = arr
  for i:=0； i< len(arrCopy); i++ 的封装
  循环的次数在循环开始的时候就确定了


  for_temp := range
  len_temp := len(for_temp)
  for index_temp = 0; index_temp < len_temp; index_temp++ {
       value_temp = for_temp[index_temp]
       index = index_temp
       value = value_temp
       original body
  }


  func main() {
      var a = [5]int{1, 2, 3, 4, 5}  // 这个是数组
      var r [5]int

      for i, v := range a {
          if i == 0 {
              a[1] = 12
              a[2] = 13
          }
          r[i] = v
      }
      fmt.Println("r = ", r)
      fmt.Println("a = ", a)

  }

  // 输出。说明真的 a 是copy
  r =  [1 2 3 4 5]
  a =  [1 12 13 4 5]
  
  

  // 一种修改方式 (注意是arr)
  func main() {
      var a = [5]int{1, 2, 3, 4, 5}
      var r [5]int
      for i, v := range &a {  // 这个是数组的指针
      if i == 0 {
          a[1] = 12
          a[2] = 13
      }
      r[i] = v
  }
  fmt.Println("r = ", r)
  fmt.Println("a = ", a)
 }
   
 // r =  [1 12 13 4 5]
 // a =  [1 12 13 4 5]


  // 一种修改方式  (注意是slice)
  func main() {
      var a = []int{1, 2, 3, 4, 5}
      var r [5]int
      // 这个是切片， for range slice 属于浅copy, 注意 for range 里面用的 arr 一定是真实的变量而不是copy的变量
  for i, v := range a {
      if i == 0 {
          a[1] = 12
          a[2] = 13
      }
      r[i] = v
  }
  fmt.Println("r = ", r)
  fmt.Println("a = ", a)
  
  
  //  r =  [1 12 13 4 5]
  //  a =  [1 12 13 4 5]
```







核心函数

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/img/20230302093813.png)



![image-20230302093834449](/Users/cy/Library/Application Support/typora-user-images/image-20230302093834449.png)



```
 new 和 make 的区别

  https://sanyuesha.com/2017/07/26/go-make-and-new/
  
  都是内存分配。
  1.make 用在 slice， map， channel， 初始化非零值（nil），返回的就是他们本身 (slice 是本身，map 和 channel 是自身的引用类型)。也只能用在上面三个类型
  
  （
  slice 结构体， 包含两个属性： 1.len 2.平时我们用的时候是指向底层数组第一个元素的指针，但本质上还是结构体，一旦扩容，指向底层数组指针改变，值就会改变，go 只有值拷贝 
  map 是 *hmap， 
  channel 是 *hchanel
  上面两个都是指针类型，所以当做参数传入函数内部的时候，可以被改变
  ）
  （slice 零值的时候不能 [0] 这样赋值， 但是可以append。 
    map 给 nil map赋值，会导致panic， 空指针
    channel 给 nil channel 写数据，会阻塞
  ）
  
  2.new 是对应类型零值元素的地址。new([]int) , 通过 reflect 可以看到的是 *[]int 类型。 new 用在上面三个类型也不方便，类似地址的地址。 
  可被替代，能够通过字面值快速初始化。 &test{}
```







```
  2. for  range 执行 goroutine, 变量问题 （这个应该各个语言都有，闭包变量作用域问题）（黑金宝盒问题类似，闭包中使用的变量是外部改变的变量，而没有用传入的方法，类似defer func() {} 的执行，

  之前有个同事 defer log.Print("%+v", res). 本想着打印最后的结果集，结果因为参数开始执行的时候已经确定了。并不会真的在打印出最终的res
  
  defer f(). 这个 f 是初始传进去的样子，后期再修改这个 f 并不会改变 defer 的执行


  3.interface 比较问题。

  iface {
   _itab 类型
   data 数据指针
  }

  eface {
   type 类型
   data 数据指针
  } (empty的意思)

  必须type data 都是nil, 才能代表是 nil。
  
 
 3.并发问题 （race）-race 参数


 4.cannot use names (type []string) as type []interface {} in argument to printAll） 
  es 的 terms 查询条件。
  interface 参数可以被 string 等很多类型传入，但是 []interface 不能直接用 []string, 需要主动构造[]interface。 时间复杂度是 o(n)， 得一个一个转换
  https://www.jianshu.com/p/03c81f4e3956
```



















```
map 是什么。

kv 存储结构

一般实现的用hash 表。

hash 冲突用的 拉链法，

最差时间复杂度 o(n)。 平时 o(1)

```





```
map 结构体。 map 底层。


// A header for a Go map.
type hmap struct {
    // 元素个数，调用 len(map) 时，直接返回此值
	count     int
	flags     uint8  // 读写竞态校验
	// buckets 的对数 log_2
	B         uint8  // 比如 B = 3, 代表 有 []buckets 有8个元素
	// overflow 的 bucket 近似数
	noverflow uint16
	// 计算 key 的哈希的时候会传入哈希函数
	hash0     uint32
    // 指向 buckets 数组，大小为 2^B
    // 如果元素个数为0，就为 nil
	buckets    unsafe.Pointer  , hash 表
	// 等量扩容的时候，buckets 长度和 oldbuckets 相等
	// 双倍扩容的时候，buckets 长度会是 oldbuckets 的两倍
	oldbuckets unsafe.Pointer
	// 指示扩容进度，小于此地址的 buckets 迁移完成, 感觉可以通过top hash 来判断
	nevacuate  uintptr 
	extra *mapextra // optional fields, 
}


读取顺序
1.判断map 是否为空，为空返回默认零值。
2.计算hash 值。
3. 后b位 判断落在拿个桶中， 定位具体的 buckt （struct 名称 bmap）
4. bmap 中如果 old 桶不为空，说明在迁移。如果在迁移，判断当前桶的状态，也就是对比桶tophash 第一个数据的值，是否 小于 min hash值，如果小于说明此 bucket 中的 key 全部被搬迁到了新 bucket。（因为默认赋值的时候 tophash 的值会加上min hash）
5.得到在新桶还是老桶中那数据。
6.遍历tophash， 利用hash 高8位，判断是在key 所在 index，通过偏移量得到 value 所在位置
7.判断value 是否是自己需要的元素，如果是返回，如果不是接着在overflow 中寻找。


```

```
map key 的定位过程。 （不会触发搬迁操作）

1.key 经过hash， 获取后b 位。 判断落在哪个bucket 中。

2.（定位 bucket）获取 map 的 oldbucket 是否为nil， 如果是nil， 说明不在rehash 过程中，接着往下走。 如果 oldbucket 不是 nil， 说明在rehash， 此时需要获取 key 在老 map 中 bucket 所以位置。 判断是否是等量扩容，还是 倍数扩容，确定b 的大小。 找到 key 所在老的 bucket 索引元素， 读取该元素 tophash 的第0个元素，判断该bucket 是否迁移完。如果迁移完了，就回到新的 buckets 中，没有迁移就去老的buckets 中。

3. （在bucket 中定位元素）遍历tophash 数组，判断 key 是否相等，如果相等，拿到index， 通过 offset 偏移 拿到 value 的值。 如果没有相等的，通过 overflow 寻找。

```



```
map 的赋值过程 类似上面key 的定位过程。 （会触发搬迁操作）

1.需要检查并发读写。

2.确定bucket 位置，且 该bucket 迁移完成。

3. 修改值正常进行，如果是插入，需要检测是否扩容。
```



```
map 的delete 会触发搬迁操作

其他的类似 赋值，key 的查找
```



```
map 的扩容。

两种场景：
1.2倍扩容。 负载因子 > 6.5。 （count/ bucket 的数量）
2.等量扩容。 overflow 过多。（可能之前新增了很多元素，塞满了 bucket中8个空间，但后期删完了）


等量扩容简单，元素bucket 索引不变。

增量扩容，需要迁移元素到新的bucket 中（hash 第b位是否是1， 0的话索引不变）。

对于新老bucket ，有 x， y 标志区分。 在tophash 中有体现。
```



```
注意

buckets 中单个元素内，对于 bucket index 一样的元素填充 (8个k-v)

每个元素的地位是一样的，从tophash 第一个开始填充。

所以寻找key 的时候，我们也是遍历 tophash 数组。
```



```
map 带 comma 和 不带 comma 是

两种不同的底层函数实现。汇编实现的， go自身没有重载功能

func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool)


go 对于不同类型key的访问，底层方法不一样。
类似 redis 不同数据类型有不同的实现
```



![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/img/20230302231142.png)



```
map 遍历过程。

先随机任意bucket 的任意cell

再顺序遍历，判断是否有迁移。

如果没有迁移，或者已经迁移到新的bucket ，直接遍历。

如果没有迁移到新的bucket， 只遍历后几位满足条件的元素。对于另一个新的bucket， 等到遍历他的时候再取数据
```



![image-20230302232100550](/Users/cy/Library/Application Support/typora-user-images/image-20230302232100550.png)

 





```
map 线程不安全。

if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	
throw 这种还没法recover。	
```



```
map 不能取地址。

/main.go:8:14: cannot take the address of m["qcrao"]

func main() {
	m := make(map[string]int)

	fmt.Println(&m["qcrao"])
}


如果通过其他 hack 的方式，例如 unsafe.Pointer 等获取到了 key 或 value 的地址，也不能长期持有，因为一旦发生扩容，key 和 value 的位置就会改变，之前保存的地址也就失效了。

```



```
go map的遍历是无序的

原因 map 会扩容，所以输出元素可能乱序。

为了防止大家误用，因为有时候顺序，扩容后就不顺序

所以go 在每次遍历获取数据的时候都用上随机值。bucket 随机，bucket 下面的cell 随机。
```

 



```
 map 遍历的时候append， 可以读取到么
 
 map 的遍历没有次数限制。 并没有 copy 一个新的 map。
https://segmentfault.com/a/1190000023477520

 map 是无序的，随机从一个bucket（也就是index），故意的， 开始遍历，不会像 slice 那样有固定的次数。

不仅仅bucket 任意，而且bucket 下面的key 也是任意

 可能读取到，也可能读取不到，（如果插入的元素在当前遍历到的元素后面，就能输出）

```





```
 map 可以边遍历边删除么 
 
 同一个协程中可以的。

不同的协程中会报冲突 （并发不安全，可以用sync map）

被删除的元素可能先被读取到

```



```
map  struct 类型value 修改  


 不可以， 会rehash， 

key => value 地址 ，可能会变动

m1 := make(map[string]S1)

m1["test"] = S1{Name: "test"}
m1["test1"] = S1{Name: "test1"}
m1["test1"].Name = "TEST" // php 中经常这么编写

// go 中这样是不允许的， 
//slice之所以可以这样取值感觉是因为slice每次取值都是针对 初始指针做的偏移计算


 // 下面这种方法可以解决map 的取地址问题。直接用地址类型
  m1 := make(map[string]*S1)
  m1["test"] = &S1{Name: "test"}
  m1["test1"] = &S1{Name: "test1"}
  m1["test1"].Name = "TEST" 


  // string 也不能寻址，所以
  a := "tt"
  a[0] = "c" // 这样是不能修改的
  a = "ttt" // a的地址其实是改变的

  // 常量，零值也不能寻址，意思就是不能修改。

  issure: https://github.com/golang/go/issues/3117
  
  最常见的说法是 map 的value 不可以寻址。

  m1 := map[int]int{1:2}
  fmt.Println(&m1[1]) // 报错，map 的value 不能寻址

  T{} 也不能直接寻址, 不能直接修改他的值，但是能读取。
```

 

```
map的比较。


map 本身不能比较， 只能用deep equal   （不能用 == ）

map 只能和 nil 比较是否相等

slice 也不能比较大小， 要用 deep equal （不能用 ==）

只能用 reflect.DeepEqual, 其实很少见 map 和 slice 需要比较大小的。


```

  

```
工作中印象比较深的事

背景 : go routine panic 没有捕获到 ，导致 pod 重启，  而且还没有打印 panic 堆栈位置，很难排查问题。 (一般容易发生在 job 任务中 ， web server 框架是 gin 会打印出堆栈)
解决 :  
1.消息走先提交，后执行的模式，有问题消息不会被重试， 不会一直导致服务重启 。 
2，利用 addr2line, 查询是哪一行代码出了问题。


 背景： 佩洛西访台， 单redis 流出过大。（100m）
  排查：
  1.观察流出陡增，和链接数增长趋势一致，因为是池化，所以判断 client 增长，此时有扩容，判断是扩容机器上的定时任务。
  2.运维找出大key


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
interface 可以比较大小, 倒不是比较大小，是比较是否相等。

常见error 比较大小, 其实是比较是否相等


slice， map 和 func 痘印不能比较是否相等，

需要通过 deep equal.

(数组能比较是否相等)

  注意这些比较大小的时候，都和元素顺序有关，（slice 有关，map 本身就是无序的，应该没啥关系，interface 这种涉及到 type 元素 和 元素自身的比较大小）


  https://juejin.cn/post/6881912621616857102 
  关于是否可以比较


  struct 可以比较的前提是必须是同一种数据类型，或者是强制转换成统一种类型。还有如果 == ，必须所有字段都是可比较类型 


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
  fmt.Println(ss3==ss1)   // 含有不可比较成员变量， 还有这似乎得在同一个包中
  }
```





  ```

 go 中关于interface 解释   
 
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
  
    
 // 包含的第一个属性是 引用类型，类似 slice 。

 // 本质上和 map -》 hmap 不一样， map 取得是 hmap 的地址类型，上面的验证可以通过 unsafe 去验证
 
 
 
 
 
 interface 和 nil 比较，需要 动态类型和动态值 都是 nil， 才能判断  == nil
 
  例子：

  func main() {
      var v interface{}
      v = (*int)(nil)
      fmt.Println(v == nil)
  }

  // 输出 false， 因为 type 是 *int

  func main() {
      var data *byte
      var in interface{}
      fmt.Println(data, data == nil)
      fmt.Println(in, in == nil)
       in = data
     fmt.Println(in, in == nil)
   }

  // 输出
  true
  true
  false
  
  
  
  
  
  
  
   https://mp.weixin.qq.com/s/F9II4xc4yimOCSTeKBDWqw

  煎鱼的这篇文章很好的诠释了工作中可能不小心遇到的问题。

  对于结构体指针类型，我们可以直接用nil 比较。

  对于 interface （error） 类型，我们要结合 type 和 value 一起比较


  以上interface 的比较，就遇到之前在做 redis ErrNil 错误的对比，注意ErrNil 是我们申明的一个变量，方法里面我们使用这个变量的时候，直接return 这个包级别的变量，不能根据 message 再重新生成，否则不等于 （临时生成的变量地址，还外抛， 野指针）
  ```



![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/img/20211012182635.png)

 

```
  下面本质上还是interface 比较的问题
  
  我们在返回类型的时候，如果返回类型是interface 类型，比如 error

 notice:
  1.我们直接返回nil （不给具体的动态type类型）
  2.如果我们返回 具体的类型，那不管这个类型表示的意义如何， 他都不等于 nil, 因为interface 的动态类型有了具体的值。

  


 因为返回interface 的话，我们很难判断是否是nil， （因为 interface 的 type 肯定不是nil 了）
 我们需要把他的值拉出来判断是否是 nil。可以通过反射。

 b站的 error ，相判断二者是否相等， 先断言成标准error  struct, 再利用该 error 的cause 方法解析。
 
 
 
 
 判断interface 是否value 是nil

  func main() {
      var data *byte
      var in interface{}
       in = data
  fmt.Println(IsNil(in))
  }


   // go的 反射包封装了 isNil 方法。可以了解下 reflect的 key ， value 之间的转换。
  func IsNil(i interface{}) bool {
      vi := reflect.ValueOf(i)
      if vi.Kind() == reflect.Ptr {  // 这块其实有缺陷，除了指针，还有 map slice func 的特殊处理
          return vi.IsNil()
      }
      return false
  }
```







```
  // unsafe 指针的使用
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
  len := (*int)(unsafe.Pointer)(uintptr(unsafe.Pointer(&arr) + 8)
```



```
 // 这个 error 错误比较很重要，因为类似redis 查询的时候都会用到， redis.ErrNil, 所以比较的时候都是对比 package 里面定义的变量
  https://www.veaxen.com/golang%e6%8e%a5%e5%8f%a3%e5%80%bc%ef%bc%88interface%ef%bc%89%e7%9a%84%e6%af%94%e8%be%83%e6%93%8d%e4%bd%9c%e5%88%86%e6%9e%90.html
  
   看了上面的文章，发现要真的是 errors.New("11") == errors.New("11") 如果为true 的话，那就完了。因为error 生成的时候又不会全局检查，万一一样了。
   
   
  没有方法的interface , 包含元素类型 和 元素的实际存储位置，只有二者都相等，我们才能认为2个interface 相等。

  同理 interface 和 nil 比较大小，也就是只有 type 和 value 都等于 nil 的时候我们才能认为他是 nil
  
```



![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/img/20211003170536.png)

 






 




```
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
 }
 
  // ----------------------
  
 // p2 是指针类型
 p2 := &Person{age: 100}
  
 // 指针类型 调用接收者是值类型的方法

 // 指针类型 调用接收者也是指针类型的方法
 p2.GetAge()
 
 
 
  // 上面那块代码虽然是能直接执行的  !!!! （因为语法糖， 而不是 生成了对应的发放）

  // 但是如果我们定义了 interface 类型， 我们的 p 变量是没有拥有 *p 的方法的。 虽然  *p 包含了 所有 p 的方法

  // p 能自动生成*p 的方法， 但是 *p 的方法不能自动生成 p。

  // interface 之所以对应的 struct 不能执行 *struct 方法，有种解释是， interface 转换后的变量不能寻址，所以 struct{} 不能像语法糖那样 直接调用 &struct{} 的方法
  
 
```

   

 

  ```
  
  map 的两种get

v, ok := m1["name"]
  // ok 主要告知到底有没有这这个key
  // v 取不到值就是默认值，和ok 本身并没有太大关系. map 取值唯一要考虑的就是 map 自身是 nil 的情况
  // map 如果是 nil， 其实是可以取值的，但不能给map 赋值，赋值会 panic.
  // 断言的时候 ok 可以安全断言，防止 panic 。 但是map 取值的时候，如果不想知道到底有没有这个key， 这个ok 可以直接忽略、

  ```



```
发现引用类型 ,如果是 nil 也能调用对应 结构的方法，只是不能调用对应结构 中的字段

type Name struct {
    N1 string
  }

// grpc 中经常封装这种方法
  func (n  *Name) Get() string{
     if n == nil {
      return ""
    }
     return n.N1

}

   var t *Name 
t.Get() 是正常执行的，我们再调用字段的时候，用 t.Get() 可以省去判断 t 是否是nil
```





  ```
   

* channel 的读取

  	// 两种单方向的 chan
  	var  c  <- chan int
  	  
  	var  c1  chan <- int
  	
  	  c, bool := <-chan
  	
  	  注意，channel中有元素的时候，不管 channel 是否关闭，都是true， 其实就是标志这个数据是不是init data（这块可以结合源码看一下）
  	
  	  就是这个标志位不会实时根据channel 是否关闭，而是在读完所有数据之后，如果关闭了，channel 不会阻塞，可以返回值，否则是阻塞。
  	
  	
  	  // 给 nil chan 发送数据和接受数据，都会永远堵塞
  	
  	  
  	
  	  // 代码中我们经常要做到通知 关闭的功能
  	  // 之前 我们是向一个 作用关闭的channel 中塞入一个信号，当我们从中获取到 信号的时候，代表功能关闭
  	  // 其实我们大可不必， 可以直接把这个channel 设置成  <-chan struct{}, 当这个chan 不堵塞的时候，代表这个chan 关闭了

  

* 断言

  ```
  t, _ := m1.(int) // 安全断言，通过 comma 判断断言是否正确，如果断言错了， t 是该类型零值
  // 如果不用ok， 断言错了，会panic
   // 这个断言可以是 具体类型，也可以是 interface
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
  

* Atmoic

  ```

  atomic 包含的有 int  ,bool , interface

  add  增加

  load 加载 

  store 存储

  compare and swap  参数 addr， old， new  返回是否swap 成功。

  之前看到一篇文章关于 cas 和 sync.Once 的区别。 
  sync.Once 会阻塞 通过 sync.Mutx, 
  cas 操作并不会。 

  感觉如果不阻塞 住， 后面代码 的变更如果 依赖 那个只用执行一次的func， 但那个func 又比较耗时，结果会不符合预期。

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

  可以，但是float 存在精度问题。

  浮点数比较大小一定要注意。不能用 ==

  ```
 

* map 的赋值过程是怎么样的 

  ```

  --》并发检测 （flag）
  --》定位key （hash 后b位，2^b 为桶的个数）所在的bucket，是否正在扩容（ tophash 和 minTophash 比较），如果是必须得先完成迁移， 再进行增删改查
  --》是否存在，存在即 更新， 不存在查找 tophash 空闲的地方
  -》 判断插入后是否会扩容，如果扩容需要扩容完再走一边流程。
  —》 修改 count 值，修改 flag 值



  ```


![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/img/20230109012832.png) 



* Map 怎么实现顺序读取

  ```
   把 key 排序，用slice 存储
  ```



* go 引用类型

  ```

   nil 是值，代表初始值。这个变量如果是 nil 还是能取到地址的

  map   （可以直接在函数内部修改，但是感觉这样写容易出bug， 我们的func 作用是修改map， 感觉容易被忽略，感觉不如把值返回出来）
  slice （原则上讲还不是，只是结构体中包含一个指向数组的指针）
  channel  （所以channel 可以在各个func 中传递， 让并发编程更加方便）

  

  这三个我们一定要 make 初始化之后使用，否则会 panic （slice 的 append 倒是不会）

  interface // 自身值类型 struct，只是struct 里面指向了地址类型 data

  https://zhuanlan.zhihu.com/p/105554073

  上面这篇文章写得还是不错的，虽然和 当前题目感觉没有多大关系

  ```
 

  

*  go interface

  ```

  没有方法的 eface
  type + data (unsafe pointer)



  有方法的 iface
  (itab (inter + _type  + 方法集合的第一个元素地址，便于寻找 (
  methods := (*[1 << 16]unsafe.Pointer)(unsafe.Pointer(&m.fun[0]))[:ni:ni]
  所以可以用 fun [1]uintptr 去存储所有方法，其实有一大片连续的地址空间
  ))  + data (unsafe pointer  ))

  ```


* Go 的 sync 包都用过哪些

  ```
  sync.Mutex

  sync.Map

  sync.WaitGroup

  sync.Once

  sync.Pool


  sync.Cond 不好用，随机唤醒协程，容易饥饿，一般通过 channel 去解决特定唤醒的问题。
  ```

  



* go 除了 mutex 锁意外还有那些方式安全读写共享变量

  ```
  channel (sync.mutex -> atomic)


  atomic 包 （sync.Mutex 底层调用了atomic）

  感觉本质上都是调用atomic 去控制并发



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


​    
    例2：
    // 如果是defer body 依赖外部的参数，那defer 最后真正执行的时候才能确定参数值。
    
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


​    
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


​    
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
        f1()
    }


​    
    例5：
    
    func f1() (r int) {
        defer func() {
            r++  // go 和 php 不一样，这样不能直接使用，和上面返回的r 是同一个变量
        }()
        return 0
    }
    func main() {
        fmt.Println(f1()) // 1
    }


​    
    例6：
    
    func f() (r int) {
         t := 5
         defer func() {
           t = t + 5 
         }()
         return t
    }
    
    输出：
    5


​    
​    
    例7：
    func f() (r int) {
        defer func(r int) {
              r = r + 5  // 局部变量
        }(r)
        return 1
    }  // 1


​    
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


​    
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
    
    

​    

```
例10：

很难的一道题
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
```





```
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

2.defer 嵌套不能生效。

3.recover 只能捕获最近的一个panic， panic 会被覆盖。

```



```
1. invalid memory address or nil pointer dereference

所以 grpc 框架中对于字段的获取，都会 先判断 地址类型struct 是否nil， nil 直接返回默认零值。 否则返回属性


2. map 就算是 nil 读取不会出现上面情况， 但是赋值会，所以为了减少心智负担，还是提前给初始化吧。

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
go什么时候 内存逃逸从栈上逃逸到堆上 
  1.地址类型的返回，（野指针, 闭包函数， map的返回）
  2.大的内存占用 （slice len很大，大于栈的内存空间）
  3.interface类型 （fmt， 编译期间不知道具体的数据类型）

  gc 回收的是堆上的对象。
```





 





```

GMP 数量的限制
G 每个对应内幕才能 2 ~ 4k， 比如 100 0000 ，需要内存 4000 000k = 4G

M 工作线程， M 的默认数量限制是 10000，如果超出则会报错

P process， 可以设置，一般和核数大小一致

```

```
其实goroutine id是存在的

只是大家不建议使用
```

```
GMP 模型，为什要有p

1.之前没有p 的时候，都是全局队列上取 goroutine， 锁竞争激烈

2.如果把 g直接挂在m 上，因为m 会阻塞，这时候我们不希望 g 的处理会被阻塞，所以需要把老的 g队列转移到新的 m的g队列上，麻烦

3.g 队列的数量不会随着 m的增加而增加。
```



```
结构体是否能比较？

其实工作中真的没有碰到过结构体的比较。

倒是遇到 interface 和 nil 的比较。

结构体是能比较的，如果所有字段都能比较。

1.map slice func 不能比较，只能深度比较。

2.指针能比较，如果相等要指向同一个地址。

```



```
go time after 协程泄露

res := make(chan int, 1)
res <- 10
select {
 case time.After(3 * time.Second):
    // timer 不能及时被回收
 case <-res:
    fmt.Println("--test--")
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
var nums1 []interface{}
 nums2 := []int{1, 3, 4}
 num3 := append(nums1, nums2)
 fmt.Println(num3)
 
 
 输出：
 1
 
 这题我感觉并不是很难，主要是 append 最后的元素并没有拆解。
 
 
```



```
go 是值传递

但值传递的话，如果参数是引用类型，还是会修改原先的值的。

map slice channel 

slice 底层包含一个指向数组的指针

map channel 也包含了指向数组的指针

区别是 slice 自身是个结构体，如果存在扩容情况， 指向底层数组的指针被改变，因为不是引用类型，所以字段值被修改后，无法体现在传入变量上。

map 和 channel 因为底层就是值类型，所以不存在上述情况， 不用担心扩容的情况。




还有个要注意的点是打印变量

func (p *pp) fmtPointer(value reflect.Value, verb rune) {
 var u uintptr
 switch value.Kind() {
 case reflect.Chan, reflect.Func, reflect.Map, reflect.Ptr, reflect.Slice, reflect.UnsafePointer:
  u = value.Pointer()
 default:
  p.badVerb(verb)
  return
 }
 
 对 channel ，map， slice 做了特殊处理。
 
 打印的是底层数据的地址
```



```
面向对象


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
GMP 模型下为什么要有p

1.没有p的时候，全局唯一一个队列，锁竞争严重，goroutine 窃取也尽可能减少了 m空转，减少了 cpu 的无效消耗。

2.为什么不挂在m 上，而是直接抽象一个新的组件p。 因为m 可能阻塞，阻塞后并不会销毁，而是用新的 m替代，如果每次产生新的m 就多一条 goroutine 队列，显然不合理，队列的管理也更加复杂。
```



```
结构体是否能够比较

能比较，只要所有字段都能比较。

func main() {
    v1 := Value{Name: "煎鱼", Gender: "男"}
    v2 := Value{Name: "煎鱼", Gender: "男"}
    if v1 == v2 {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}

相等。




type Value struct {
    Name   string
    Gender *string
}

func main() {
    v1 := Value{Name: "煎鱼", Gender: new(string)}
    v2 := Value{Name: "煎鱼", Gender: new(string)}
    if v1 == v2 {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}

不相等，gender 指向的变量不一致。






type Value struct {
    Name   string
    GoodAt []string
}

func main() {
    v1 := Value{Name: "煎鱼", GoodAt: []string{"炸", "煎", "蒸"}}
    v2 := Value{Name: "煎鱼", GoodAt: []string{"炸", "煎", "蒸"}}
    if v1 == v2 {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}

slice， map， function  不能比较。






type Value1 struct {
    Name string
}

type Value2 struct {
    Name string
}

func main() {
    v1 := Value1{Name: "煎鱼"}
    v2 := Value2{Name: "煎鱼"}
    if v1 == v2 {
        fmt.Println("脑子进煎鱼了")
        return
    }

    fmt.Println("脑子没进煎鱼")
}

不同类型的不能比较。
```



```
单核cpu， 开两个goroutine， 其中一个死循环会怎么样 

1.14版本以下会hang住

包含以及之后的版本会正常，出现了基于信号的抢占式调度。


runtime.sysmon 做的检测的事
1. sysmon线程是无限循环执行。
2. sysmon线程在每个循环中，会进行netpool（获取fd事件）、retake（抢占）、force gc（按时间强制执行gc），scavenge heap（释放自由列表中多余的项减少内存占用）等处理。
3. sysmon线程一开始每次循环后，休眠 20us，50次之后（即1ms后）每次休眠时间倍增，最终每一轮都会休眠 10ms。

sysmon 发信号给 m， m收到信号好休眠正在阻塞的goroutine， 调用绑定的信号方法，并进行重新调度。
```



```
进程，线程都有id， 为什么goroutine 没有 id

是有goroutine id 的，但是不建议使用。
```



```
Goroutine 数量控制在多少合适，会影响 GC 和调度

gmp 

p  processor 可以设置，一般和系统核数一样。 gomaxprocs 

m machine, 默认设置 10000， 超过会报错。 GO: runtime: program exceeds 10000-thread limit

g  初始 2k ~ 4k 大小， 所以手内存大小限制。


go working steal 先从 global 队列取，再从其他 本地队列取。

```





```
slice 和 map 是否 线性安全

slice 不。 len， cap ，data组成，没有锁。

容易造成 索引覆盖。


map 更不能，会throw error 

用sync.map 替代。


```



```
 sync.map 底层结构 ？
```



```
map 的并发 是否能被recover ？

不能，直接throw 出来的。

map 的nil， 是否能recover ？

能，panic 的信息。
```











```
米哈游面试题 （https://learnku.com/go/t/60933）

...
m := make(map[int]int, 10)
for i := 1; i<= 10; i++ {
    m[i] = i
}

for k, v := range(m) {
    go func() {
        fmt.Println("k ->", k, "v ->", v)
    }()
}
...

核心问题就是 k v 引用了外部变量，需要用  k := k, v := v 替换。


内存泄露， 什么情况下回内存泄露 ？

很出名的一个例子，time.After 时间间隔比较久。

之前同事写了一个例子 


tim := time.NewTimer(3 * time.Second)
res := make(chan interface{})
select {
case <-time: 
    do()
case tmp := <-res:
    do()
}
结果没了 case2， res 一直阻塞状态， go 协程没办法塞入数据，导致go 协程一直存在，内存泄露。

自己也写过一个例子

goArr := make(chan struct{}, 5)

for _, v := range funcArr {
   goArr <- v
}

for _, v := range goArr {
   go func() {
      v()
   }
}

没有close goArr, 导致 goArr 所以协程一直没有释放。



channel 的底层实现原理

1.环形数组。
2.goroutine 组成的双向链表。
3. sync.Mutex  保证线性安全。


make 和 new 的区别。
make 主要用在 channel， slice， map ，数据类型初始化。 new 主要是用在结构体初始化，返回地址。


channel 关闭了接着 send 数据会发生什么，关闭一个已经关闭的 channel 会发生什么。
send close 会panic 。（想象一下，send 并没有返回值，不能出现别的情况）
close close 的 也会panic。 （没有返回值）
获取 close 的channel， 先把数据读取完，再获取到默认零值，通过 comma 去判断。

send 或者 获取 没有初始化的 channel， 会阻塞。


map 是线程安全的吗，map 的扩容规则。
map 非线性安全，每次读取写入都通过flag 去判断。
map 是渐近性hash， 如果在扩容，会只写到新的 hash 表中。如果是读取，会根据迁移进度判断是在老的hash 表中还是新的，去对应的hash 表中读取，同时迁移未完成的数据。


数组和切片的区别。
切片的底层有个field 是指向数组的指针。
切片可以扩容，大小不一定。
切片不能比较大小， 数组可以。

GC


GMP 模型。

g goroutine， 数量受内存限制。
p process 和核数保持一致
m 工作线程，默认 10000， 超过报错，可以修改。

g 在队列中，通过p 和 m 进行绑定。 如果本地队列为空，会先到global 队列中拿数据，再没有会窃取其他的本地队列。
如果遇到系统调用，例如网络请求， g会和 m 一起脱离 p， 等到完成， g 回到可调用队列中，m 也回到自己的队列中。



进程、线程、协程。

进程，资源分配的最小单位。
线程， cpu 执行的最小单位。内存占用大。 内核态。
协程，用户态线程。内存占用小



微服务相关，包括微服务的注册与发现，微服务的监控，微服务的限流相关等等，还有微服务怎么守护进程，我回答的是 supervisor，也不知道对不对。
具体业务的实现，兑换码的实现，如何批量生成几十万或者上百万的兑换码，（这个我回答的是用雪花算法实现），高并发下，一个兑换码如何保证只能被一个人兑换一次，以及一个兑换码如何可以被多个人兑换的实现。（这道题前前后后回答了有半个小时吧，因为之前做过相关的业务，所以心里有点底）
三个算法问题。
写一个方法，解决：输入 4 个数字，通过加减乘除，输出一个期望值。
广度优先算法：怎么输出各层的值。
台阶问题，假如对于上台阶，可以一次上一阶，也可以一次上两阶，写一个方法，实现输入台阶数，输出可以有多少种上法。


Goroutine 阻塞的话，是不是对应的M也会阻塞?


如何并发100个任务，但是同一时间最多运行的10个任务（waitgroup + channel）
errgroup ，设置最大协程数，for range  通过channel 拿任务。

```



```
米哈游面试二

go 里面使用map 应该注意的问题 和 数据结构。

1. map 不能并发读写。
2. map 不能比较大小。
3. map 的value 不能直接寻址。
4. map 使用前需要先初始化，否则可能给一个nil map赋值报错。(读取是没关系的)
5. map 的缩容其实有点问题， 如果是value 是指针类型还好。

数据结构
{
count 字段
hash 算法
[]bmap
}

bmap 对应一个 

hash 索引
key 数组
value 数组，通过偏移量读取数据
overflow  hash冲突的时候溢出




Map 扩容是怎么做的？
扩容成之前2倍大小，渐进式hash， 有字段代表是否在rehash 中，如果在rehash 中插入操作只往新的 hash表中插入，读取的时候会先判断数据在新的还是老的hash 表中，读取对应hash 值， 也会迁移对应数据。


Map 的 panic 能被 recover 掉吗？了解 panic 和 recover 的机制吗？
map 的panic 有的能被，比如给nil map 赋值。 有的不能，比如并发读写， throw.
panic机制 ？ 只能捕获当前协程的panic， recover 只能在func 中执行，靠栈存储，先进后执行， 可以参考小米的那篇文章原理。


Map 怎么知道自己处于竞争状态？是 Go 编码实现的还是底层硬件实现的？
flag 变量，go 编码实现。


CAS 具体是怎么实现的呢？



并发使用 Map 除了加锁还有什么其他方案吗？
sync.map.


有对比过 sync.Map 和加锁的区别吗？
sync.map 更适用于读场景。



sync.Mutex 的数据结构可以说一下吗？


Context 平时什么场景使用到？
基本上每个函数都用到。比如 http接口中，例如 gin ，把 req 封装在了 context 中，还有 context 中通过中间件注册了用户uid 这些，从里面拿。


context.WithTimeout 使用现象和实现？
如果想缩短超时时间。比如总的服务超时250ms， 依赖的其中某个服务只能给50ms。
里面会判断传入的超时是否超过当前已存在的超时，如果是延长，直接不处理返回。

实现：应该是一个定时器，到点cancel 。


context.WithTimeout 还有 cancel 返回值，如果不去调用是否有问题？
不会有问题，如果想提前cancel，可以调用，如果不调用，到点了函数内部会自己调用。


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



30 个并发打印 0-99

ch := make(chan int)
	go func() {
		for i := 0; i < 30; i++ {
			ch <- i
		}
	}()

	wg := sync.WaitGroup{}
	for i := 0; i < 30; i++ {
		go func() {
			defer wg.Done()
			wg.Add(1)
			fmt.Println(<-ch)
		}()
	}

	wg.Wait()
	
	
	

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
   （3）go 中 switch  type 可以用于获取字段类型 (只能是interface 类型)
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
  
  CSP (描述并发系统中交互模式的形式化语言，通过共享内存去通信，而不是通信去共享内存 。go 语言参考了三种， c的语法， csp 的并发 channel 通信， Pascal 的 import package)
  
  CAS （原子性操作， compare and swap）
  
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
  （因为没有加recover， 所以panic 还是终止了程序继续往下执行
  ）
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
  // 注意 struct 的类型一定要一致呀，但是匿名结构体的比较，只考虑字段
  // 结构体比较所有field 都得能比较，如果包含map 这种不能比较的会报错。
  
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

* go 的优势

  ```
  Go 允许跨平台编译，编译出来的是二进制的可执行文件，直接部署在对应系统上即可运行。一次编译到处执行。（但其实也不是所有的操作系统）
  Go 在语言层次上天生支持高并发，通过 goroutine 和 channel 实现。channel 的理论依据是 CSP 并发模型， 即所谓的通过通信来共享内存；Go 在 runtime 运行时里实现了属于自己的调度机制：GMP，降低了内核态和用户态的切换成本。
  Go 的代码风格是强制性的统一，如果没有按照规定来，会编译不通过。
  ```

* For 循环

  ```
  案例一：
  
  var all []*Item
  for _, item := range items {
   all = append(all, &item)
  }
  
  // 存储的都是item 这个变量的地址，也就是 items 最后一个变量
  
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

* append

  ```
  func main() {
   var nums1 []interface{}
   nums2 := []int{1, 3, 4}
   num3 := append(nums1, nums2)
   fmt.Println(len(num3))
  }
  
  // 1, 虽然不知道想考证的是啥
  // 但是  []string 不能直接转换成 []interface{} 
  ```

* 返回值命名的问题

  ```
  
  func aaa() (done func(), err error) {
   return func() { 
     print("aaa: done") 
   }, nil
  }
  
  func bbb() (done func(), _ error) {
   done, err := aaa()
   
   // 这块, 递归了
   // done = func {
   //   print("bbb: surprise!"); 
   //   done() 
   //}
   
   return func() { 
     print("bbb: surprise!"); 
     done() 
   }, err
  }
  
  func main() {
   done, _ := bbb()
   done()
  }
  
  
  // 上面执行会栈溢出
  ```

  

* go 值传递

  ```
  go 语言都是值传递，之所以 slice, map, channel 传递的时候能修改原始的值是因为 引用类型的原因
  
  1.slice  
   {
    data uintptr
    len int
    cap int
   }
   
  slice 传递的是结构体，但结构体的第一个属性是指向底层数据的指针，所以在不扩容的情况下底层数据会被修改
  
  2.map channel 都是 指针类型，所以能被修改
  
  
  // 下面的打印 地址是一样的，是因为 go 的fmt 做了特殊处理
  
  func main() {
   s := []string{"烤鱼", "咸鱼", "摸鱼"}
   fmt.Printf("main 内存地址：%p\n", s)
   hello(s)
   fmt.Println(s)
  }
  
  func hello(s []string) {
   fmt.Printf("hello 内存地址：%p\n", s)
   s[0] = "煎鱼"
  }
  
  
  
  // 
  func (p *pp) fmtPointer(value reflect.Value, verb rune) {
   var u uintptr
   switch value.Kind() {
   case reflect.Chan, reflect.Func, reflect.Map, reflect.Ptr, reflect.Slice, reflect.UnsafePointer:
    u = value.Pointer()
   default:
    p.badVerb(verb)
    return
   }
  
  // v.ptr 是 unsafe.Pointer 
  func (v Value) Pointer() uintptr {
   ...
   case Slice:
    return (*SliceHeader)(v.ptr).Data
   }
  }
  
  type SliceHeader struct {
   Data uintptr
   Len  int
   Cap  int
  }
  
  ```

* go 怎么实现 协程一个出错，其他任务都被取消

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
```

* go  中如何实现面向对象

  ```
  1.封装 。（首字母大写 公共方法， 首字母小写，私有方法）
  2.继承。 struct 嵌套， 组合
  3.多态。 interface
  
  官方说既可以说算面向对象，又可以不算。go 缺乏层次感，没有extends的概念
  ```

* 怎么 借用外力 主动停止  一直执行的 goroutine 

  ```
  https://mp.weixin.qq.com/s/tN8Q1GRmphZyAuaHrkYFEg
  
  goroutine如果本身在主动执行某个任务是不能主动停止的，煎鱼上面文章的介绍主要是场景：轮训的定时任务，怎么通知这个协程去主动结束
  
  1. select 多通道，有个关闭通道， 关闭之后能读取到内容。 （
     1. ctx.done() 是一种类型，只读channel， close 之后不在阻塞。 
     2. 普通有数据channel， data, ok := <- c, ok = false 关闭
     
  2. for range 通道，关闭通道之后不再阻塞。
  ```

* go 内存模型

  ```
  ```

* Go happen before

  ```
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

* go map 缩容

  ```
  ```

* sync.Pool 的使用场景

  ```
  sync.Pool 本身是并发安全的.但是New 方法的并发安全性需要自己保证。
  
  
  bufferPool := &sync.Pool{
          New: createBuffer,
      }
  
  func createBuffer() interface{} {
      // 这里要注意下，非常重要的一点。这里必须使用原子加，不然有并发问题；
      atomic.AddInt32(&numCalcsCreated, 1)
      buffer := make([]byte, 1024)
      return &buffer
  }
  
  // 记得熟悉一下 sync.Pool 的源码
  ```

* 为什么 map 和 slice 是非线程安全的

  ```
  slice 和 map 底层都是 struct
  
  slice {
  data  uintptr
  len  int
  cap  int
  }
  
  对struct 修改协程不安全，不能保证一致性
  
  
  map 的底层结构更复杂，map 底层还会检测 flag 状态，如果多写，会throw 报错。
  
  
  map 并发读写
  
  1.map + sync.Rw 锁
  2.sync.Map  
  
  ```

* gmp 模型数量限制

  ```
  g goroutine， 数量没限制，但创建需要 2k~ 4k 内存大小，受内存限制
  m machine , 默认数量是 10000， 超出会报错，通常只有在 Goroutine 出现阻塞操作的情况下，才会遇到这种情况。这可能也预示着你的程序有问题。
  若确切是需要那么多，还可以通过 debug.SetMaxThreads 方法进行设置。
  
  p processor, 在 Go 语言中，通过设置 GOMAXPROCS，用户可以调整调度中 P（Processor）的数量。
  另一个重点在于，与 P 相关联的的 M（系统线程），是需要绑定 P 才能进行具体的任务执行的，因此 P 的多少会影响到 Go 程序的运行表现。
  
  所以三者数量相互独立。但实际执行的时候
  
  每个p 上挂着 一个本地队列 ，本地队列里面装着 g, m会轮训p队列上的runable  g。
  但如果出现系统调用这样的阻塞， m 和 g 会一起离开p ，感觉是这个原因导致 m 的数量比p 多。
  ```

* gmp 为什么要有 p

  ```
  1.刚开是的模型是没有p 的， 所有的m 从全局队列上拿 g， 会有严重的锁竞争。
  2.出现了本地队列。 那为什么 本地队列不挂在 m 上呢 ？m 会因为 系统调用暂时陷入睡眠状态，这时候会生成新的 m 去执行g 。 如果本地队列 挂在 m 上，会导致本地队列增加， 本地队列理论上 和 cpu 的核数保持一致就可以了。
  ```

* go time after 的坑

  ```
  经常 
  
  select {
  case <-res.Chan
  
  case <- time.After(time.Second)
     // 超时
  }
  
  
  超时这个没有被执行到，但是变量不会释放，duration 后会被释放，导致内存泄露， 更可怕的是 for 循环这种
  
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
    case <- time.After(time.Second)
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
  
  
  2. 上面的方法类似 time.NewTicker(), 这个有定时器的作用，不用重置，而且也可以主动 stop。
  ```

* 单核cpu. 一个协程 死循环，另一个会怎么样

  ```
  核心就是 1.14 之后，go 的调度策略有改变，从以前协作式，增加了抢占式调度。
  
  runtime.sysmon会检测：
  
  1.抢占阻塞在系统调用上的 P。
  2.抢占运行时间过长的 G。
  
  发送信号给 M。M 收到信号后将会休眠正在阻塞的 Goroutine，调用绑定的信号方法，并进行重新调度
  ```

* 那些年遇到的goroutine 泄露的事

  ```
  1. goroutine 给channel 发送值被阻塞
  
  go func() {
    // 泄露
    time.Sleep(2 * time.Second)
    res.Chan <- res
  }
  
  c := time.Ticker(time.Second)
  select {
  case <-res.Chan:
  
  case <-c.C
  	
  }
  
  超时之后 res.Chan 没法塞值进去，堵塞。
  
  常见的还有 var c chan int
  var 声明的chan 没有初始化，发送数据会被堵塞。
  
  
  2. errGroup ， 调用了 maxProcess。
  for _, v := range chanFunc {
     v()
  }
  
  wait 里面有 close(chanFunc)
  结果没有调用wait， 协程一直没有被释放。
  ```

* 关于 sync.WaitGroup

  ```
  注意 add 和 done 要一一匹配。
  
  wait 的核心就是 add 的数据，被done 全都完成了
  
  s := new(sync.WaitGroup) // 注意如果是struct 结构体，就不能在func 中传递了， 因为是直接传值， 需要传引用
  
  s.Add(1)  // 一定要在协程外面
  go func() {
     defer s.Done() // 一定要被执行
     
  }
  
  s.Wait()
  
  ```

* 关于 zero-base

  ```
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

* func 的定义

  ```
  func Test() {}
  
  func main() {
    c := Test  // func 类型的赋值。 c 引用类型，不为 nil
  }
  
  ```

* 在使用切片的时候，面试题经常会有第三个变量，主要用来修改切片的容量

  ```
  arr := []int{0, 1, 2, 3, 4}
  c := arr[0:2:1]  // 编译不通过
  c := arr[0:2:2]  // cap(c), 2
  
  ```

* Int 竟然能被string 强转, 因为 UTF-8 编码的存在。

  ```
  c := 12
  d := string(c)
  fmt.Println(d)
  ```

* interface 的转换问题

  ```
  type S struct {
  }
  
  func f(x interface{}) {
  }
  
  func g(x *interface{}) {
  }
  
  func main() {
      s := S{}
      p := &s
      f(s) //A
      g(s) //B
      f(p) //C
      g(p) //D
  }
  
  g() 函数之所有有问题，是因为 interface ， []interface{}, 都需要单独转换，不能一次性转换
  
  如下：
  var ss interface{}
  ss = s
  g1(&ss)
  
  ```

* ```
  func main() {
      if a := 1; false {
      } else if b := 2; false {
      } else {
          println(a, b)
      }
  }
  
  曾经觉得上面的代码有问题，其实
  
  if err := do(); err != nil {
  
  }
  
  所以上面的写法是没问题的，
  
  print(a, b) 也是在作用域范围内
  
  ```

* ```
  func (i int) PrintInt ()  {
      fmt.Println(i)
  }
  
  func main() {
      var i int = 1
      i.PrintInt()
  }
  
  // 编译错误
  
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

* 感觉很沙雕的题目

  ```
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

* 断言和 switch type 只能用在 interface身上

  ```
  A.
  type MyInt int
  var i int = 1
  var j MyInt = i
  
  B.
  type MyInt int
  var i int = 1
  var j MyInt = (MyInt)i  //  (MyInt)(i). 经常这样修改
  
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

* i++的细节

  ```
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

* 那些年不能寻的址

  ```
  const i = 100
  var j = 123
  
  func main() {
      fmt.Println(&j, j)
      fmt.Println(&i, i)
  }
  
  // 常量不能寻址 （因为不能改）
  // 所以上面不能编译通过
  ```

* Channel 经常问的

  ```
  一个 nil channel 发送数据，造成永远阻塞
  从一个 nil channel 接收数据，造成永远阻塞
  ```

* for range 中容易遇到的

  ```
  func main() {
      x := []string{"a", "b", "c"}
      for v := range x {
          fmt.Print(v)
      }
  }
  
  // 注意for range 第一个元素是key， 第二个元素是value
  ```

* 别名

  ```
  type User struct{}
  type User1 User
  type User2 = User
  
  func (i User1) m1() {
      fmt.Println("m1")
  }
  func (i User) m2() {
      fmt.Println("m2")
  }
  
  func main() {
      var i1 User1
      var i2 User2
      i1.m1()
      i2.m2()
  }
  
  能，输出m1 m2，第 2 行代码基于类型 User 创建了新类型 User1，第 3 行代码是创建了 User 的类型别名 User2，注意使用 = 定义类型别名。因为 User2 是别名，完全等价于 User，所以 User2 具有 User 所有的方法。但是 i1.m2() 是不能执行的，因为 User1 没有定义该方法。
  ```

* 关闭 channel

  ```
  func Stop(stop <-chan bool) {
      close(stop)
  }
  不能关闭 读channel
  
  var cc  chan <- int
  close(cc)  // 这是可以的
  ```

  

* ```
  go 中 i++ 是语句，不能直接用来进行赋值
  
  c[i++]
  //  不能这样使用，得分成 2步，如下
  
  i++
  c[i]
  ```

* ```
  常量和 函数参数未使用，在 go 中是可以被允许的
  ```

* ```
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

* ```
  func main() {
      m := make(map[string]int,2)
      cap(m) 
  }
  
  
  使用 cap() 获取 map 的容量。1.使用 make 创建 map 变量时可以指定第二个参数，不过会被忽略。2.cap() 函数适用于数组、数组指针、slice 和 channel，不适用于 map，可以使用 len() 返回 map 的元素个数。
  
  cap 代表能存储的元素， slice 和 channel 都可以。感觉map 有点特殊，因为 bucket 下面能存储多个元素
  
  len代表其中的元素，slice， channel， map 都可以。
  ```

* ```
  var c error
   c = errors.New("11")
  
   c1, cok := c.(interface{})
  
   fmt.Println(cok)
   fmt.Println(reflect.TypeOf(c1).Name())
   
   今天遇到断言的一个小问题。 interface 再断言成 interface 。其实是可以的，不会报错， ok 那边代表的是但凡有实体type 类型，返回的就是true， 否则就是 false。
  ```

* ```
  import (  
     _ "fmt"
      "log"
      "time"
  )
  var _ = log.Println
  func main() {  
      _ = time.Now
  }
  
  go中对于未使用，但需要引入场景的修改。
  
  _ 占位
  ```

* ```
  var x = []int{2: 2, 3, 0: 1}
  
  func main() {
      fmt.Println(x)
  }
  
  [1 0 2 3]，字面量初始化切片时候，可以指定索引，没有指定索引的元素会在前一个索引基础之上加一，所以输出[1 0 2 3]
  
  // []int{2: 2, 3, 3: 1} 这样会报错，输出 Duplicate index
  ```

  
