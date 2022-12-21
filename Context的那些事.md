之前面试的时候会被问到context 的几个问题。

```
1. context 是什么类型 ?  interface, 实现的有 valueCtx, cancelCtx, timerCtx
2. context 的取消是怎么实现的？父context 取消能取消子context吗？子context取消能取消父context吗？父context 取消子context 需要做什么才能实现自己取消操作吗？
遍历 child map， 挨个调用 cancel。
父context 能取消子context，子context 不能取消父context， 只能把自己从父 context 的 child map 中删除。
所有 context 的取消都要监听 信号channel， c.Done(), 主动触发
```



so， 我们来看下context 包的内部实现。



### Context

Context 是一个interface, 但凡实现了下面的方法都属于Context （第一个问题）

Done() ， 返回一个channel， 一般 context 被取消的时候，我们能从这个channel 中读取到内容，不需要纠结读取到的内容，我们可以概率化的理解是一个信号📶。

Err()  , Context 被取消的原因

Value(key interface{}) interface{} ,  获取到 k-v, 对应的v

Deadline() (deadline time.Time, ok bool), 获取Context 如果能被取消最后的时间点



### Canceled

主要是WithCancel(ctx context.Context) 包装返回的子context 和 cancel()， cancel() 方法执行的时候，会给Context 实现的struct  error 存储这个Canceled , 下次 这个struct 的 Error() 方法执行的时候可以直接读取到这个主动cancel error



### DeadlineExceeded

本质是是一个deadlineExceededError struct， 因为实现了Error() 方法， 所以也是 error 类型interface。 实现了如下方法:

Error() { return "context deadline exceeded" }

Timeout() { return true }

Temporary() { return true }

很好理解，定义好的变量，当做一个常量错误使用



### emptyCtx

是 int 的类型别名 https://learnku.com/articles/31280。他是一个实现了 Context 接口的 struct， 因为所有实现的方法都是类似初始化，所以一般把他当做root ctx。

Deadline()  返回初始化time.Time和 false

Done() 返回nil， 永远不会被关闭

Err() 返回 nil

Value()  返回nil

String(), 主要是为定义的两个变量 TODO， background 的打印（反射）。

```
background = new(emptyCtx)
todo       = new(emptyCtx)
```



### Background()

返回上面的变量



### Todo()

返回上面的变量



### Canceler()

实现cancel() 和 done()



### contextName()

Context 的输出



### propagateCancel

总结就是如果一切正常的话，如果父是 cancelCtx， 把自己塞入到父的child map中



### WithCancel(parent Context) (son Context, cancel func)

包装一个Context， 返回一个 cancelCtx 和 可以主动取消的函数。这个函数触发的时候，会给 生成的cancelCtx 的 channel 中塞入结束信号，用来取消自己和 孩子 context。

调用propagateCancel 构建父子context 关系



### cancelCtx

Struct,  上面方法的返回

```
type cancelCtx struct {
	Context  // 父context， 所以实现了 Context 接口
	mu       sync.Mutex            // protects following fields， 主要防止修改 field的并发
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call，当再次调用withcancel 的 时候会给 子 context 和 父 context 关联上
	err      error                 // set to non-nil by the first cancel call // 如果取消了，这个err 就有值
}

```

Value()  方法，除了 &cancelCtxKey 这个key 特殊处理外，剩下的都是调用父Context 的Value() 方法。

Done() 方法， lazily 创建，就是用来装载取消信号的 channel

Err() , 返回 err 属性

String().   调用上面的 contextname()  +  .WithCancel 这个字符串，通过这个感觉可以看出当前ctx 的属于第几代。

cancel()   

1.关闭自己的channel， 

2.For 循环 child map， 都执行cancel ，

3.如果父类也是 cancelCtx, 可以把自己从父类的childrenmap 中删除。



### timerCtx

```
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.
	deadline time.Time
}
```



### WithDeadline(parent Context, d time,Time)

1.如果当前时间已经结束，直接调用前面的withCancel 方法返回，也不用起定时器了,返回的是cancelCtx

2.如果没结束，返回 timerCtx， 相当于两层， parentCtx ----》 cancelCtx ----》timerCtx. 如果已经过期了，直接调用 timerCtx 的取消。相比较 cancelCtx 的取消，不同的两件事1.停止定时器。2.删除父节点的childmap 中内容。因为构建的时候我们用的是 parentCtx 和 timerCtx 做的，所以不能直接用 cancelCtx 的cancel 方法，这块细评一下

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20211231193004.png)



### WithTimeout

转换成调用deadline 方法



### valueCtx

struct

Value() 读取就是递归的往父级读取。

### WithValue

对于每个k-v 都包装一下，返回valueCtx, 





### 易错点

```
对于异步任务，公司有个组件叫 fanout， 相比较传统的 go func ，他可以限制协程的数量 。
其实本质上就是一个 errgroup， 利用 errgroup  自身限制协程数量 ，还有阻塞功能，还有平滑关闭的功能 （题外话，还是返回到文章重点 ctx 上吧）

因为是异步任务，所以 context 并不需要 timeout，看下面这个方法

fanout.add(ctx, func(ctx1 context.Context){
	 // 自己的方法
	 do(ctx1) // 注意一定要用ctx1
})

这个add 方法就是对我们传入的 ctx 进行重写，如下：
Done() {
 retuen nil
}
Err() {
 retrun nil
}

使其成为一个没有结束时间的ctxNew,

然后把我们的 ctx 和 func 组成一个结构体，丢入任务channel 中。

另一方面，限制了最大并发数的 works 从任务channel 中取出消息结构体，执行我们的 func()，参数是结构体的另一个 field ctx, 所以我们在 编写我们具体的方法的时候，一定要用参数ctx1, 而不是 add 的ctx 参数，add 的ctx 参数是外面传进来没有经过处理的， 可能有超时时间的， 而消息结构体的 ctx1 是经过重写的ctx ，是没有超时时间的
```







