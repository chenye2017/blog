源码和面试中经常用到.

首先是

* unsafe.Pointer 。 可以代表任意类型的指针，但想使用的话，必须得转换成具体的类型，比如

```
s := "cc"

c := (*string)(unsafe.Pointer(&s))

fmt.Println(*c)

// 一般打印出来是16进制

// 这个很强大，可以转换任意内存布局结构相同的struct， 比如后面说的 string 和  []byte 的转换，
// 一些私有struct 我们想转换
// 比如 interface 的iface
iface
{
tab  *itab
data unsafe.Pointer
}

简化一下
iface
{
tab uintptr  // 8字节
data uinptr  // 8字节
}
```



* uintptr,  这个也代表指针， 本质上是一个无符号的整形，长度和平台相关。 和上面相比， 这个主要用来进行内存地址的计算，比如结构体中一个field 占用 4 字节，想获取第二个字节的内容 （通过加法运算），可以如下

```
type Name struct {
  n int32
  n2 int32
}

n := Name{11,12}

c := (*int32)(unsafe.Pointer(uintptr(unsafe.Pointer(&n)) + 4))

fmt.Println(*c)

// 上面 + 4 的主要原因是 n 是int32， 占用4字节，&n 代表的结构体第一个属性开始的位置。

// 还有上面没有内存对齐，因为都是 int32, 如果第一个是 int8, 那需要填充3 字节， 仍就是4

// 关于 uintptr 除了运算不建议使用赋值，有下面2个限制，目前认知不够明朗

1.如果uintptr地址相关联对象移动，则其值也不会更新。例如goroutine的堆栈信息发生变化

2.uintptr地址关联的对象可以被垃圾回收。GC不认为uintptr是活引用，因此unitptr地址指向的对象可以被垃圾收集。

// 参考： https://studygolang.com/articles/33151

// 一般打印出来是10进制
```



* 题外话: 出了上面的两种指针，还有种最常见的，如下

  ```
  c := 10
  
  fmt.Println(&c)
  // 同 unsafe.Pointer 16进制
  ```

  

上面说的 + 4 就是偏移量，可以通过

* unsafe.Offset(n.n2) 来 获取， 需要哪个字段的偏移就传入哪个字段，是累加的返回，如

```
type Name struct {
  n int32
  n1 int32
  n2 int32
}
 n := Name{}
//  unsafe.Offset(n.n)  0
// unsafe.Offset(n.n1) 4
// unsafe.offset(n.n2) 8
```



想获取一个字段占用多少大小，通过该

* unsafe.SizeOf(), 常用的 int64 这些我们知道 64/8 = 8 ，不确定的可以通过方法来获取

  ```
  任意的指针，大小都是 8
  
  type itab struct {
  	inter  *interfacetype // 8字节
  	_type  *_type // 8字节
  	link   *itab // 8字节
  	hash   uint32 // 4字节
  	bad    bool   // 1字节
  	inhash bool   // 1字节
  	unused [2]byte // 2字节
  	fun    [1]uintptr // variable sized // 8字节
  }
  
  以上为例，每个字段只在乎前面的内存布局能否满足自己的步长，整读。
  
  struct 的字段并发赋值安全，迁移是每个字段单独赋值
  
  ```

  



上面说道的另一个知识点 内存对齐，对齐基础通过方法

* unsafe.Alignof() 来获取， 我们的机器是 64位，所以最大的就是8 位（机器步长）



面试中问的常见问题

1. 利用unsafe包修改私有成员

   ```
   type s1 struct {
   	id int8  // 8
   	name string // 16 8
   }
   
   s1 := new(s1)
   idAddr := (*int8)(unsafe.Pointer(s1))
   *idAddr = 12
   ```

   

 2.获取unsafe 包获取 slice的len 和 map 的count

```
slice

{
 data unsafe.Pointer
 len int
 cap int
}

// 获取slice 的len
arr := []int{12}
lenAddr := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&arr)) + 8))
fmt.Println(*lenAddr)

map

本身就是hmap
{
count int
...
}
m := map[int]int{11:10, 12:13}
lenAddr := (**int)(unsafe.Pointer(&m))
fmt.Println(**lenAddr)

// 好家伙， * * 
```



3.字符串和byte 切片零复制转换

```
// 底层string 的实现 https://juejin.cn/post/7111953294469267493 
// 和 byte 的转换都要进行一次内存copy


string2slice
s := "test" 
*(*[]byte)(unsafe.Pointer(&s))

slice2String
arr := []byte("s")
*(*string)(unsafe.Pointer(&arr))

// 这种pointer能转换的本质是内存结构一致

string
{
Data unsafe.Pointer
Len int
}

slice
{
Data unsafe.Pointer
Len int
Cap int
}

string 转换成的slice cap值会比较奇怪。
```







内存对齐的好处参考：

https://eddycjy.gitbook.io/golang/di-1-ke-za-tan/go-memory-align

https://geektutu.com/post/hpg-struct-alignment.html



如果我们不知道内存布局，就无法通过 uintptr + unsafe.SizeOf 获取指针的位置，但也无妨，可以通过 uintptr + unsafe.Offset().













