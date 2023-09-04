### unsafe

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



### Context

```
平常用到的场景

1.超时控制 （cancel ctx），http 请求， grpc 请求

2.携带值 （value ctx）, 用户uid 透传， trace id 透传

```



```
context.WithTimeout 使用现象和实现？

底层 withDeadLine, 定时器 调用 cancel 方法。


context.WithTimeout 还有 cancel 返回值，如果不去调用是否有问题？

没有问题，定时会调用。 监听 ctx.Done()  这个channel， 如果获取到值，说明到点了。
```





### Sync

```
 关于 sync.WaitGroup


 注意 add 和 done 要一一匹配。
 wait 的核心就是 add 的数据，被done 全都完成了
  
  s := new(sync.WaitGroup) // 注意如果是struct 结构体，就不能在func 中传递了， 因为是直接传值， 需要传引用
  
  //waitgroup 包含一个字段 nocopy ，但只有 go vet 的时候才会提示, 本身代码问题不一定必现
  
  
  s.Add(1)  // 一定要在协程外面， 否则可能 wait 的时候 add 还没加上
  go func() {
     defer s.Done() // 一定要被执行，done 及时就是 add(-1)
     // add 方法会修改 counter（任务的个数）， wait （需要挂起的协程， 咱一般只是wait 一次）
     // 当 c == 0 && wait > 0 的时候，会逐个 release 阻塞的协程
     // 为了不用锁，counter 和 wait 两个int32 用一个int64 替代 （分成高32 和 低 32）
     // 因为 32 位机器 内存对齐是32， 所以 [0, 1, 2], 0位是信号量，1 2 分别是 counter 和 waiter
     // 如果是 64位机器，内存对齐是63， 所以 [0, 1, 2], 0， 1 分别是counter 和waiter，  2是信号量
  }
  
  s.Wait() // 通过信号方法阻塞当前协程
    
```



```
关于 sync.Map

sync.Mutex

read ->amtoic.Value -> {map[key]*entry, amened}

dirty -> map[key]*entry

miss


读取都走 read，如果存在, 就直接返回数据

如果不存在，去读 amended 字段，如果 为 true， 说明 read 和 dirty 不一致，去读dirty 。

如果为false，说明真的不存在。

注意因为读取 read 的时候都没加锁，所以读取dirty 加锁的时候要双重检查。


删除 先走read， 如果read 中存在，直接删除, 因为 map 中存储的 entry 是地址，所以dirty 中也会被删除。
如果read 中不存在，判断ameded ，如果为false， 就真的不存在了，如果为true， 就删除dirty 中对应元素。


存储 先走read， 如果read 中不存在，需要插入到 dirty 中，dirty 先复制一份 read。 （对于nil 修改成删除状态，然后copy 其他元素）
如果 read 中存在，是普通元素状态，直接修改
如果是删除状态，说明 dirty 存在，但可能dirty 和 read 不一致，除了修改 read 还要修改dirty。


元素之所以有 nil， 删除状态，是因为 dirty 中不想copy nil 元素，防止越来越大，这就导致 dirty 和 read 不一致，为了防止read 中元素被修改， dirty 中没有反馈，类似加了个标志，但凡存在这个标志，说明key 可能不存在dirty 中，需要我们也去处理dirty 中元素。
```



```
 关于 sync.Once
 
 一个int变量。通过 atmoic 去比较，判断是否执行过， 如果没执行过，用 锁锁住，然后 执行方法，修改变量。
 
 之所以用锁，是因为 atomic cas 操作不会阻塞。
 
```



```
关于 sync.Pool

变量复用。但因为无状态，然后没有大小限制，也可能随时被回收， 不会用在连接池里面，而只是存储变量
```



```
关于 sync.Mutex
```





### Atomic

```
```



