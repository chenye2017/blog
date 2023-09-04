```
map 是什么。

kv 存储结构

一般实现的用hash 表。

hash 冲突用的 拉链法，

最差时间复杂度 o(n)。 平时 o(1)
```



```
map 需要注意的点

1. map 不能并发读写。(flag 变量校验， recover 不了)
2. map 不能比较大小。
3. map 的value 不能直接寻址。（php 那种 .赋值） map[12].Value = xxx
4. map 使用前需要先初始化，否则可能给一个nil address 赋值报错。(写会有空指针报错，读取是没关系的)
5. map 是没有顺序的
6. map 的缩容其实有点问题，b的数量不会变，所以实际占用空间大小不变
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
4. bmap 中如果 old 桶不为空，说明在迁移。如果在迁移，判断当前桶的状态，也就是对比桶tophash 第一个数据的值，是否 小于 min hash值，如果小于说明此 bucket 中的 key 全部被搬迁到了新 bucket。（因为默认赋值的时候 tophash 的值会加上min hash）(老桶的 B 数量通过 ： 如果是等量扩容，不改变。如果是 增量扩容，B 减少一)
5.先判断老桶中是否有数据，如果没有，就去新桶。
6.遍历tophash， 利用hash 高8位得出一个数值，遍历 索引数组， 判断key 所在 index，通过偏移量得到 value 所在位置
7.判断value 是否是自己需要的元素，如果是返回，如果不是接着在overflow 中寻找。（感觉冲突或者装不下都要overflow）


//
改进的点:
感觉是根绝 oldbucket 是否有值，推断出是否在扩容
通过oldbucket 只到老b， 找到老的bucket， 判断老bucket 是否有数据。
没数据的话去新的bucket 查找
根据hash 高8位，依次匹配key ->value,没有再去overflow 寻找。
```



```
map 的赋值过程 类似上面key 的定位过程。 （会触发搬迁操作）

1.需要检查并发读写。

2.确定bucket 位置，且 该bucket 迁移完成。

3. 修改值正常进行，如果是插入，需要检测是否扩容。
```



```
map 的delete 会触发搬迁操作  （就是写操作都会触发搬迁）

其他的类似 赋值，key 的查找
```



```
map 的扩容。(渐进式hash)

两种场景：
1.2倍扩容。 负载因子 >= 6.5。 （count/ bucket 的数量）
2.等量扩容 （这个不属于扩容，只能算是rehash）。 overflow 过多。（可能之前新增了很多元素，塞满了 bucket中8个空间，但后期删完了）


等量扩容简单，元素bucket 索引不变。
增量扩容，需要迁移元素到新的bucket 中（hash 第b位是否是1， 0的话索引不变）。
每次最多迁移 2 个buckets， 修改或者删除迁移
```



```
map 带 comma 和 不带 comma 是

两种不同的底层函数实现。汇编实现的， go自身没有重载功能
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool)
```



```
map 的两种get

v, ok := m1["name"]
// ok 主要告知到底有没有这这个key
// v 取不到值就是默认值，和ok 本身并没有太大关系. ok只是用来区分 如果 value 是默认零值，到底是不存在，还是本身就赋值成这样
// map 如果是 nil， 其实是可以取值的，但不能给map 赋值，赋值会 panic.
  
  
题外话：
// 断言的时候 ok 可以安全断言，防止 panic 。
// 但是map 取值的时候，如果不想知道到底有没有这个key， 这个ok 可以直接忽略。 因为都是ok,可能弄混淆，以为map 的读取，如果没有ok， 读取不到会panic, 其实不会
```



```
go 对于不同类型key的访问，底层方法不一样。
类似 redis 不同数据类型有不同的实现
```

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/img/20230302231142.png)



```
map 遍历过程。

先随机任意bucket 的任意cell
再顺序遍历，判断是否有迁移。
如果没有迁移，或者已经迁移到新的bucket ，直接遍历。
如果没有迁移到新的bucket， 只遍历后几位满足条件的元素。对于另一个分裂的新的bucket， 等到遍历他的时候再去老 bucket 里面取数据

// 老bucket 不遍历完，而是等到下次遍历到新的时候再遍历，感觉害挺奇怪。
```



```
go map的遍历是无序的

原因 map 会扩容，所以输出元素可能乱序。

为了防止大家误用，因为有时候顺序，扩容后就不顺序

所以go 在每次遍历获取数据的时候都用上随机值。bucket 随机，bucket 下面的cell 随机。
```



![image-20230302232100550](/Users/cy/Library/Application Support/typora-user-images/image-20230302232100550.png)



```
注意 (tophash 需要理解的地方，是顺序遍历)

buckets 中单个元素内，对于 bucket index 一样的元素填充 (8个k-v)

每个元素的地位是一样的，从tophash 第一个开始填充。

所以寻找key 的时候，我们也是遍历 tophash 数组。
```



```
map 线程不安全。

if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	
throw 这种还没法recover。	
```

```
map 的并发panic 是否能被recover ？

不能，直接throw 出来的。

map 的nil 赋值， 是否能recover ？

能，panic 的信息。
```



```
sync.map 底层结构 。

这篇文章大佬讲的很好。 https://mp.weixin.qq.com/s/13usxPGbBxQ9Z5HoEXdgng

sync.Map 零值可用

type Map struct {
 mu Mutex // 保护字段
 read atmoic.Value // 原子性， 主要用来读数据，修改数据 （或者修改数据中的一部分），删除数据， （没有新增 map key 的操作）
 dirty map[any]*entry 
 misses int
}

type readOnly struct {
	m       map[any]*entry  // 本质上和dirty 是同样的数据结构
	amended bool // true if the dirty map contains some key not in m.
}

type entry struct {
    // 这个比较核心，代表一个元素的地址
		p unsafe.Pointer // *interface{}
}


// 代表元素被删除
// 这个状态出现的原因是 dirty 不想copy read 中删除的元素， read 本身删除元素只是把entry 修改成nil（为了减少 read 自身map 的修改），这就导致了 read 和 dirty 数据不一致，我们需要把这些数据标记出来，也就是标记成 expunged， 后面如果真的给这部分数据赋值，同时需要在dirty 中添加这个key
var expunged = unsafe.Pointer(new(any))

// 新增元素
我们通过 read.Load 断言成 map[any]*entry, 
如果元素存在 read 中，且没有被标记为删除状态, 我们直接修改entry 的指向。 因为 dirty 和 read 共享底层数据，所以其中 read entry 修改了指向元素的值，也会反应在 dirty 中.

如果元素存在read 中，且被标记为删除状态，说明是 dirty 存在，dirty 复制了 read 中值，且把 read 中删除元素从nil -》expunged， （这个是后面 生成 新的 dirty 的时候体现）, 此时我们 把这个entry 添加到dirty 中，然后给这个entry 赋值。 （entry 如果是 nil 或者具体的值，在上一步就处理完了。）

如果元素只在dirty 中，我们只要修改entry 的值就好了。

如果元素都不存在，
如果 amended 是 false， 说明 dirty 不存在，需要生成dirty。copy read 中所有元素，且把删除元素从 nil -》expunged。
如果 amended 是 true， 说明dirty 存在，直接生成entry ，赋值就好了。


// 读取元素
先读取 read， read 存在，就直接返回了。

read 不存在，ameded 是 true (说明存在dirty map)，读取dirty map， 同时 miss +1

read 不存在，ameded 是 false， 说明真的不存在了。


// 删除元素
如果不存在，判断 ameded， 如果true， 说明dirty 中可能存在，删除数据。
如果存在，直接删除read 中数据。


```



```
有对比过 sync.Map 和加锁的区别吗？
sync.map 更适用于读场景。
sync.map 更适用不同协程处理不同key 的场景。只要不是频繁的添加 key和 删除key 都不错 （操作dirty map）
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
map 遍历的时候 append， 可以读取到么 ？
 
map 的遍历没有次数限制。 并没有 copy 一个新的 map。(注意 slice的区别， slice 肯定遍历不到。slice 被遍历次数限制)
https://segmentfault.com/a/1190000023477520

map 是无序的，随机从一个bucket（也就是index），故意的， 开始遍历，不会像 slice 那样有固定的次数。

不仅仅bucket 任意，而且bucket 下面的key 也是任意

可能读取到，也可能读取不到，（如果插入的元素在当前遍历到的元素后面，就能输出）
```



```
 map 可以边遍历边删除么 
 
同一个协程中可以的。

不同的协程中会报冲突 （并发不安全，可以用sync map）

被删除的元素可能先被读取到 (原因同上面的 append)
```



```
map 中  value struct 类型value 可以修改吗？


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

```



```
map的比较。


map 本身不能比较， 只能用deep equal   （不能用 == ）

map 只能和 nil 比较是否相等

slice 也不能比较大小， 要用 deep equal （不能用 ==）


只能用 reflect.DeepEqual, 其实很少见 map 和 slice 需要比较大小的。大部分都是和 nil 比较大小。


结构体是否能比较？

其实工作中真的没有碰到过结构体的比较。

结构体是能比较的，前提是包含的所有字段都能比较。

1.map slice func 不能比较，只能深度比较。

2.指针能比较，如果相等要指向同一个地址。
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

slice， map， function  不能比较。直接编译不了






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

不同类型的不能比较。 直接编译不了
```





```
map 是渐近性hash， 如果在扩容，前搬迁老的bucket， 然后写到新的bucket。如果是读取，会根据迁移进度判断是在老的hash 表中还是新的，去对应的hash 表中读取。

删除和赋值会搬迁bucket， 每次最多迁移2个
```



```
go 是值传递

但值传递的话，如果参数是引用类型，还是会修改原先的值的。

map slice channel 

slice 底层包含一个指向数组的指针，自身是一个结构体

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
 
 对 channel ，map， slice 做了特殊处理。 （所以虽然slice 是结构体，但是打印出来的是指针）
 
 打印的是底层数据的地址
```



```
  func main() {
      m := make(map[string]int,2)
      cap(m) 
  }
  
  
  使用 cap() 获取 map 的容量。1.使用 make 创建 map 变量时可以指定第二个参数，不过会被忽略 (这里的忽略说的是len 获取到的值，但对于代码效率还是有帮助的)。2.cap() 函数适用于数组、数组指针、slice 和 channel，不适用于 map，可以使用 len() 返回 map 的元素个数。
  
  cap 代表能存储的元素， slice 和 channel 都可以。感觉map 有点特殊，因为 bucket 下面能存储多个元素
  
  len代表其中的元素，slice， channel， map 都可以。
  
 
```



```
1. invalid memory address or nil pointer dereference

普通 地址类型struct 取值 容易遇到上面问题，空指针。(调用方法不会)

所以 grpc 框架中对于字段的获取，都会 先判断 地址类型struct 是否nil， nil 直接返回默认零值。 否则返回属性


2. map 就算是 nil 读取不会出现上面情况， 但是 赋值会，所以为了减少心智负担，还是提前给初始化吧。
```

