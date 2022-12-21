## slice (非并发安全，协程不安全)



slice 底层结构是 结构体， 由len, cap 和指向底层数组的指针组成 unsafe.pointer （底层数组的第一个元素）。

我们说的 go 函数传递 slice 是指针指的就是这个这个



### 场景一

```
arr := []int{1, 2,3}

func change(arr []int) {
   for k, v := range arr {
      arr[1] = 12
   }
}

抽象的场景，写算法题的时候，在做递归的时候，如果对一个arr 传递修改处理 （只修改其中值，不扩容）， 会直接在原数组上修改。

原因： 上面说了 slice 是指向底层数组的指针，go 是值传递，所以在不扩容的情况下， 底层数组指针没有变化，函数引用中修改元素值，也会修改到外部传入 slice 对应元素的值。

```



### 场景二

```
arr := []int{1,2,3}

arr1 := arr

for k, v := range arr {
      arr[1] = 12
}

赋值后，对新变量的修改会影响到老的变量

原因：slice 赋值，直接修改，也会相互影响，其实等同于场景一，go 的函数值传递和 赋值 一样。

```



### 场景三

```
arr := []int{1,2,3}

arr1 := arr[1:]

for k, v := range arr {
      arr[1] = 12
}

修改arr 后，相互影响。

原因： 
这种方式：底层也是指向同一个数组，只是第一个参数unsafe.pointer 不再指向第一个元素，而是对应的index 值， 同时修改 len 和 cap。
所以在不扩容的情况下，修改会相互影响，因为本质上底层还是指向同一个数组。

```



### 场景四

```
arr := []int{1,2,3}


func A([]int) []int{
  arr = append(arr, 1)
  return arr
}

append 追加后，不影响

原因:
我们的slice 是个结构体， 扩容之后unsafe.pointer 虽然改变，但本质上只是结构体中的一个字段，已知值传递的话是不影响原变量，所以append 不影响 原slice， 需要我们赋值后才能生效.
不扩容的时候也不影响，因为决定slice 能够被看到范围还有 len 字段，值传递的len 不会被修改，所以原变量也不会被修改。

```



### 场景五

```
arr := []int{1,2,3}


func change(arr []int) {
   arr = append(arr, 3)
   for k, v := range arr {
      arr[1] = 12
   }
}


外层arr 包含元素不受影响。

原因： arr = append(arr, 3) 修改了slice 的地址，导致 for 循环中修改的 slice 不是外面的slice
```



## Map(非并发安全，协程不安全)

### 场景一

```
m := make(map[int]int)


func change(m map[int]int){
   m[1] = 1
}

m 被修改
原因: map 底层是mhash 的指针, 引用类型变量，虽然go是值传递，但在方法内被修改，原变量也会被修改
```





## Channel(并发安全，协程安全)

channel 底层是 hchannel 的指针，所以可以通过方法传入变量的形式往内部塞值，正是这个特性 + 并发安全，才会成为协程间通信的标准。