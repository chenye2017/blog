```
slice 三个属性组成，

数组指针.
len 实际占用大小，
cap 容量。

slice 的底层是数组，数组不能扩容，slice 会扩容，扩容后数组地址会变！！！ notice。
```



```
 go 有且仅有值传递
 
 go 语言都是值传递，之所以 slice, map, channel 传递的时候能修改原始的值是因为 引用类型的原因
  
  1.slice  
   {
    data uintptr
    len int
    cap int
   }
   
  slice 传递的是结构体，但结构体的第一个属性是指向底层数据的指针，所以在不扩容的情况下底层数据会被修改
  
```





```
make([]int, 5), len是5， 被0元素填充。 ----> [0,0,0,0,0]
只想防止扩容， 应该改成 make([]int, 0, 5) ---> []. len 0, cap 5
```



```
slice append， 如果cap 容量满足，就在原先数组len 后面填充，无论是否已经有元素占领。

容易在用切片的时候掉进坑里。(都是因为切片是 浅 copy， 一般 = 都是公用底层相同的数组)

当前切片的len 会指示当前切片能看到的元素。



上面核心是为了防止这种问题：

arr := make([]int, 0, 10)

arr1 := arr[0:1] （len 1, cap 10）

arr1 = append(arr1, 10)

arr2 := arr[0:1] （len 1, cap 10）

arr2 = append(arr2, 100)


fmt.Println(arr1, arr2)
[0 100] [0 100] 

  
  
原理解析：

slice 切片， []int{}[0:4], 影响自身， 此时并没有生成新的数组，指针只是移动了位置，之前start 指向数组第一个元素，现在start指向的是index开始的元素。
len 的 end 指向切片的 end
cap 的 end 依旧指向原数组的结尾

```



```

start:end 这种切片append 很容易出现意想不到的效果，给原始数组造成影响。
因为append 填充的元素是针对原始数组的，是否扩容是当前数组的 cap 是否满足当前需求, 如果当前数组的 cap 能容纳，不用扩容，这个append的操作会影响多个切片，（在原始数组上修改）
append 元素的位置又是当前切片的 len 的下一个位置。（这个地方是个坑点，append 是len 的下个位置，而不一定是元素为0的位置）

所以append的 slice，最好用copy 函数深度copy下。如下：


  s123 := []int{5}
	s123 = append(s123, 7)
	s123 = append(s123, 9)
	x := append(s123, 11)
	y := append(s123, 12)
	
	fmt.Println(s123, x, y)
	
	// s :[5, 7, 9]. len：3， cap: 4
	// x ：[5, 7, 9, 12]
	// y : [5, 7, 9, 12]
	
我们对 slice的操作最好都copy 一下，防止对原始数组影响 。（如果你真的知道自己在做什么，也可以不copy）	
```



```
切片的扩容,也是经常问的。


分版本。

1.18版本之前：小于等于1024 ，都是double， 之后是0.25倍增长

1.18之后，包含。 小于256 都是double， 之后是  (n + 256 * 3)/ 4 的增长

还要进行内存对齐，内存对齐后一般都会多于之前的长度。

特别要注意如果一次性插入多个元素， 如果一次扩容2倍 不能 满足要同时插入多个元素的容量， 那就以同时插入多个元素的容量为标准， 而不是上面的算法为标准。
```

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/img/20230302093813.png)



![image-20230302093834449](/Users/cy/Library/Application Support/typora-user-images/image-20230302093834449.png)



```
切片作为函数参数。

切片底层是 struct， 和 map ，channel 还不太一样，后面2个都是 struct 的地址。

所以在go 的值传递中，切片自身len ，cap 不会被修改，但因为包含了指向数组的指针， 所以如果底层数组的中的值被修改，作为函数参数的切片也会被修改。

但如果因为扩容，底层数组的地址发生变化，则对底层数组的修改不会反应到函数参数切片上。


func Change1(arr []int) {
  // arr 还没扩容，修改会反应在数组上
	arr[0] = 10
	// arr 扩容了，修改不会反应在原始数组上
	arr = append(arr, 2)
}

func main() {
	test := make([]int, 1)
	Change1(test)
	fmt.Println(test)
	return
}	
```



```
  map channel 都是 指针类型，所以能被修改
  
  下面这一段都是为了证明 slice 是值传递
  
  
  // 为啥值传递， 下面的打印 变量 地址是一样的 ？？？ 是因为 go 的fmt 做了特殊处理
  
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
       // 这里面用的是 for_temp
       // for range 数组和 切片的核心区别所在
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



```
new 和 make 的区别

https://sanyuesha.com/2017/07/26/go-make-and-new/
  
  都是内存分配。
  1.make 用在 slice， map， channel， 初始化非零值（nil），返回的就是他们本身 (slice 是本身(底层)，map 和 channel 是自身的引用类型)。也只能用在上面三个类型
  
  （
  slice 结构体， 包含两个属性： 1.len 2.平时我们用的时候是指向底层数组第一个元素的指针，但本质上还是结构体，一旦扩容，指向底层数组指针改变，值就会改变，go 只有值拷贝 
  map 是 *hmap， 
  channel 是 *hchanel
  上面两个都是指针类型，所以当做参数传入函数内部的时候，可以被改变
  ）
  
  （slice 零值的时候不能 [0] 这样赋值， 但是可以append。 
    map 给 nil map赋值，会导致panic， 空指针， nil address 。 map nil 读取不会有问题。
    channel 给 nil channel 写数据，会阻塞
  ）
  
  2.new 是对应类型零值元素的地址。new([]int) , 通过 reflect 可以看到的是 *[]int 类型。 new 用在上面三个类型也不方便，类似地址的地址。 
  可被替代，能够通过字面值快速初始化。 &test{}
```



```
 在使用切片的时候，面试题经常会有第三个变量，主要用来修改切片的容量

 
 arr := []int{0, 1, 2, 3, 4}
 c := arr[0:2:1]  // 编译不通过
 c := arr[0:2:2]  // len 2， cap(c), 2 （都是结束索引减去 开始索引）
```

