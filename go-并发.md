---
title: go-并发
date: 2020-07-02 21:56:22
tags: [go,并发]
categories: go
---

这些天go的学习，相比较php，给我最大的感觉就是go 更偏向底层，相比较php有更多的接口能让我们和os交互。再者就是go 天生支持高并发，也就是goroutine （coroutine 协程），相比较php 的多进程 （process）和 java的多线程（thread）更轻量。

<!--more-->

我们在我们项目代码中可以随意的通过 go func（）{} （） 生成一个goroutine， 不同的goroutine 可以执行不同的逻辑，我们的主进程也可以看成一个goroutine，他会不等待 其他的goroutine, 如果main 执行完了，其他的goroutine 会自动结束，不管有没有执行完， 所以我们经常在代码的尾部，sleep ，等待其他goroutine执行完。但这只是demo，真正工作中无法知道一个goroutine 要执行的时间，比如 curl的返回，多种方法可以解决上述的问题，等待goroutine 执行完。

* 我们可以通过go 的sync 包，同步阻塞，等待所有的goroutine 执行完。

```
// 为了解决goroutine 一起完成我们引入了
w sync.WaitGroup

// 添加4个goroutine
w.add(4)

go func() {
  defer w.Done()
} ()

// 完成
w.Wait() 
```

sync.waitGroup 只是让goroutine 在没有channel 的执行下也阻塞住了，等待goroutine的执行，但并没有解决线程安全的问题。线程安全解决方式

1.互斥锁。但比如 web， 读多写少，互斥锁并不是很友好。

```
// 为了解决读写不一致问题
// 互斥锁
r sync.Mutex

r.Lock()


r.Unlock()

// 注意这个锁是针对代码的，而不是针对资源的，同一时刻只能有一个goroutine能执行
// 我们不能随便用 defer r.Unlock() , 当锁不存在的时候会报错
// 互斥锁有时候太严格，比如两个读操作，并不需要等待
```

2.读写锁。相比较互斥锁，只是读写互斥，读读不互斥，更友好点。

```
// 为了解决上述问题，出现了读写锁
r Sync.RWMutex

// 读锁
r.Rlock()
r.RUnlock()

// 写锁
r.Lock()
r.Unlock()

// 相比于普通锁 (读写锁性能更高，因为两个读锁之间不会互斥)
r Sync.Mutex 

r.Lock()
r.Unlock()
```

3.并发安全的map。

4.data++ 这种操作原来也会出现线程不安全问题 。除了锁之外，还可以通过原子性操作解决并发问题





* 通过channel ，我们在channel 的使用过程中经常会发生死锁问题，这恰恰是我们可以通过channel 等待goroutine 执行的关键所在。


[https://studygolang.com/articles/18800](https://studygolang.com/articles/18800)

这篇文章很好的解释了channel 造成死锁的几个原因。

[https://juejin.im/post/5d216f07e51d4550bf1ae8e0](https://juejin.im/post/5d216f07e51d4550bf1ae8e0)

掘金的这篇文章比较简单，但也说明了channel的几个使用场景。



关于并发：

首先想说一下关于并发的场景，其实并发在生活中很常见，只要用户量足够多，同一时刻很可能有两个人在做同样的事情，但这不是高并发，高并发常见的场景就是秒杀的时候，很多用户在某一个时刻被召集起来争抢有限的资源。总结：就是某一时刻某个api 被大量请求，这就是我们压测的原因， 在高并发的情况下我们的服务是否稳定。

其实很多时候我们的接口在高并发下其实没啥影响，比如幂等的接口（获取某个后台配置），只要我们的服务器撑得住，多少的并发量不是问题，这不是目前我想表达的并发，想说的是非幂等性接口在高并发下的危险性，比如秒杀情况下的超卖。为什么会出现超卖，就是我们在获取数据的同时并不是立马就能处理逻辑，比如在我们减少库存的途中，另一个请求先完成了，导致库存数已经为0了，这时候虽然我们之前的库存1 ，减少为0 合情合理，但其实我们在减少库存1的时候库存数量已经是0了，我们并不知道，所以实际库存已经是 -1了，这就是超卖。

这个库存可以类比到我们代码中就是公共变量，在lnmp的架构中，因为我们的webserver其实是 fpm提供的，fpm 会有多个进程，每个进程中有php，我们的业务代码存在于每个进程中，每当请求结束，这个进程中的变量生命周期就结束了，这种同步模式不存在公共变量读写不一致问题，唯一可能的就是我们从第三方中间件比如redis 比如 mysql中数据读写不一致问题，一般是redis ，所以就有了 redis + lua 的 原子性操作。

而在go 中，当我们用go 的 net/http 包做webserver 的时候，对于全局变量，不同的goroutine会存在读写不一致问题， 这时候我们就会引入sync 包



模拟并发的出现也是个技术活

```
go func() {
for {
  if n > 0 {
    n := read()  //10
    fmt.Println(n--)
      }
   }
}

这样正常情况下不会发生超卖，因为间隔时间太短

go func() {
for {
  if n > 0 {
    n := read()  //10
    
    //
     time.Sleep(5 * time.Second)  // 睡个5s ，必然出现
     
    fmt.Println(n--)
      }
   }
}

```



go 对于并发问题的解决，经常用到goroutine,我们用goroutine 经常会造成 死锁，我们来分析一下这些原因

```
package main

import "fmt"

func main() {
	result:=make(chan int)

	go func() {
		sum:=0
		for i:=0;i<10;i++{
			sum=sum+i
		}
		result<-sum
	}()
	fmt.Print(<-result)
}

// 不会造成死锁，以上示例使用一个单独的goroutine求和，当得到结果时，存放在result这个chan里，然后供main goroutine读取出来。当result没有被存储值的时候，读取result是阻塞的，所以会等到结果返回，协同工作，通过chan通信。

// 如果这个 <- result 是写在go func 上面，就会造成，因为一直阻塞在读取 chan 处，并不会执行到协程处，就会造成死锁

// 死锁发生的原因就是 主协程被阻塞住，然后也没有办法解决这种状态
// 不管是主协程还是从协程，但凡从channel 读取数据，读取不到都会被阻塞住，之所以 从协程堵塞住不会死锁是因为不会影响主协程的执行。当主协程塞数据进去的时候从协程就能执行了。
// 上面的特性又引入了另一个不通过 sync包而让主协程等待其他协程全都执行完了再往下执行的方法


logicCount := 5
finishChannel = make(chan bool, logicCount)

for i:=0;i<logicCount; i++ {
  go func() {
    finishChannel <- true
}()
}

for i:=0; i< logicCount; i++ {
  <-finishChannel
}
读取logicCount 之后就不堵塞了
```



* 我们平时工作中用到 协程的场景 (这篇文章的核心！！！！)

  1.一个api接口中聚合了不同的逻辑，相互不干扰。比如我们的home/info, 我会启动5个goroutine，去处理不同的业务。我会启动两个channel， 一个代表finishChannel  (chan bool),主要用来阻塞主协程的执行，一个resChannel, 用来读取数据。之所以不用resChannel 来阻塞主协程，是因为我在判断所有业务是否都执行完的时候并不想把这些数据都取出来（现在想想也可）。resChannel 中的消息体一般长这样

  ```
  type Message struct {
    Flag string
    Msg interface{}
    Err err
  }
  ```

  flag 代表不同的逻辑，下面一个switch 接入，进行不同的逻辑处理。msg 因为不同逻辑返回内容不一样，我们取出来之后要断言一次。err 就是错误信息。

  2.同一个接口循环去做通样的事。比如没有群发消息，我们不能bulk，只能单个循环。我们可以利用goroutine。


多进程下面读写安全的map(其实很简单，读取的时候加读锁，写的时候加写锁) (可以读一下cache2go 来了解一下本地缓存， 核心也是一个安全的map， 本地缓存相比较分布式缓存有一个缺点是如果容器重启了，缓存就消失了，如果没有redis 这种落地数据的操作)



上面我的home/Info 还可以有个方式去解决 并发等待问题就是 errGroup， 类似于waitgroup，把常用的几个功能进行了分装 （add  wait done）, 主要作用如下

- errgroup 可以捕获和记录子协程的错误(只能记录最先出错的协程的错误)
- errgroup 可以控制协程并发顺序。确保子协程执行完成后再执行主协程 （waitgroup 的wait）
- errgroup 可以使用 context 实现协程撤销。或者超时撤销。子协程中使用 ctx.Done()来获取撤销信号

```
// demo

package main

import (
    "context"
    "fmt"
    "time"

    "golang.org/x/sync/errgroup"
)

func main() {
    group, ctx := errgroup.WithContext(context.Background())
    for i := 0; i < 5; i++ {
        index := i
        group.Go(func() error {
            fmt.Printf("start to execute the %d gorouting\n", index)
            time.Sleep(time.Duration(index) * time.Second)
            if index%2 == 0 {
                return fmt.Errorf("something has failed on grouting:%d", index)
            }
            fmt.Printf("gorouting:%d end\n", index)
            
            select {
            case: err <- ctx.Done():
   				// 获取其他协程或者主协程的终止信号，来进行相对应的处理
            }
            
            return nil
        })
    }
    if err := group.Wait(); err != nil {
        fmt.Println(err)
    }
}

```

分析一下errGroup 的源码

```
type Group struct {
  cancel  func()             //context cancel()
    wg      sync.WaitGroup         
    errOnce sync.Once          //只会传递第一个出现错的协程的 error
    err     error              //传递子协程错误
}

// 虽然我们没法主动调用cancel， 但是我们可以通过返回的ctx 接收到 cancel 信号
func WithContext(ctx context.Context) (*Group, context.Context) {
    ctx, cancel := context.WithCancel(ctx)
    return &Group{cancel: cancel}, ctx
}

// 就是go 方法挺蛋疼，不能传参数进去
func (g *Group) Go(f func() error) {
    g.wg.Add(1)

    go func() {
        defer g.wg.Done()
        if err := f(); err != nil {
            g.errOnce.Do(func() {       
                g.err = err             //记录子协程中的错误
                if g.cancel != nil {
                    g.cancel()
                }
            })
        }
    }()
}

```

errGroup 虽然好，但是仅仅解决了我们 waitgroup 的工作 和 goroutine 中 err传递的工作，对于结果集的传递，感觉还是得造一个channel 用来传递

```
package common

import (
	"sync"
)
//安全的Map
type SynchronizedMap struct {
	rw *sync.RWMutex
	data map[interface{}]interface{}
}
//存储操作
func (sm *SynchronizedMap) Put(k,v interface{}){
	sm.rw.Lock()
	defer sm.rw.Unlock()

	sm.data[k]=v
}
//获取操作
func (sm *SynchronizedMap) Get(k interface{}) interface{}{
	sm.rw.RLock()
	defer sm.rw.RUnlock()

	return sm.data[k]
}

//删除操作
func (sm *SynchronizedMap) Delete(k interface{}) {
	sm.rw.Lock()
	defer sm.rw.Unlock()

	delete(sm.data,k)
}

//遍历Map，并且把遍历的值给回调函数，可以让调用者控制做任何事情
func (sm *SynchronizedMap) Each(cb func (interface{},interface{})){
	sm.rw.RLock()
	defer sm.rw.RUnlock()

	for k, v := range sm.data {
		cb(k,v)
	}
}

//生成初始化一个SynchronizedMap
func NewSynchronizedMap() *SynchronizedMap{
	return &SynchronizedMap{
		rw:new(sync.RWMutex),
		data:make(map[interface{}]interface{}),
	}
}
```



```
// 之前模拟过一个例子
// 利用500 个goroutine 对 普通map 进行增加，会发现运行结束，count 的值不是500
// 对上面的 安全map 修改，count 的值也不正确
// 因为上面的值只有get ， get完 +1 put， 这不是原子性，所以只有 incr 方法才能让最终的值是500

```



关于 context ,很重要的一个知识点，对于并发，我们之前一直通过 waitgroup . add 去添加信号量，但是对于 树状的goroutine， 这样会越来越复杂，完美的方式还是通过 context 上下文的传递。 我们设置一个可以随时取消的上下文，当上下文被cancel 的时候，这个请求自然就结束了。

```
// 本质上这些 控制并发用的都是 channel
// channel 也经常用于 代码的阻塞
// 下面的代码中如果 一直select 不到数据，会一直default， 但我不知道select 这个频率是多少
// select 只会被调用一次，这是我们需要知道的
// select 下面的case 都满足，会随机公平的选择一个
// 当default 和 case 都满足的时候 ，会优先case

func main() {
	stop := make(chan bool)

	go func() {
		for {
			select {
			case <-stop:
				fmt.Println("监控退出，停止了...")
				return
			default:
				fmt.Println("goroutine监控中...")
				time.Sleep(2 * time.Second)
			}
		}
	}()

	time.Sleep(10 * time.Second)
	fmt.Println("可以了，通知监控停止")
	stop<- true
	//为了检测监控过是否停止，如果没有监控输出，就表示停止了
	time.Sleep(5 * time.Second)
}
```



```
// 关于 context 需要认知的几个东西

// 首先是context 的 struct
type Context interface {
	Deadline() (deadline time.Time, ok bool) // 方法是获取设置的截止时间的意思，第一个返回式是截止时间，到了这个时间点，Context会自动发起取消请求；第二个返回值ok==false时表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消。

	Done() <-chan struct{} // 这个done 方法我们经常用，就是select 获取到值就说明超时了

	Err() error  //方法返回取消的错误原因，因为什么Context被取消。

	Value(key interface{}) interface{} // 这个value 方法在我们项目中用到的就是中间件解析内容，让后放到context里面
}

// 经典用法
func Stream(ctx context.Context, out chan<- Value) error {
  	for {
  		v, err := DoSomething(ctx)
  		if err != nil {
  			return err
  		}
  		select {
  		case <-ctx.Done():
  			return ctx.Err()
  		case out <- v:
  		}
  	}
  }  

// 平时经常用的两个context 
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```



context 的衍生

```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key, val interface{}) Context

这四个With函数，接收的都有一个partent参数，就是父Context，我们要基于这个父Context创建出子Context的意思，这种方式可以理解为子Context对父Context的继承，也可以理解为基于父Context的衍生。

通过这些函数，就创建了一颗Context树，树的每个节点都可以有任意多个子节点，节点层级可以有任意多个。

WithCancel函数，传递一个父Context作为参数，返回子Context，以及一个取消函数用来取消Context。
// 通过主动调用这个cancel 方法，可以结束当前函数的执行


WithDeadline函数，和WithCancel差不多，它会多传递一个截止时间参数，意味着到了这个时间点，会自动取消Context，当然我们也可以不等到这个时候，可以提前通过取消函数进行取消。
// 理解 成截止日期。当然我们也可以手动调用，结束函数的执行


WithTimeout和WithDeadline基本上一样，这个表示是超时自动取消，是多少时间后自动取消Context的意思。
// 这个是duration，上面的是time


WithValue函数和取消Context无关，它是为了生成一个绑定了一个键值对数据的Context，这个绑定的数据可以通过


withxxx  返回的 cancel 我们可以主动调用， 完了就可以取消这个context ， 这时候 done() 方法就能获取到内容了


关于 withValue 赋值
//
我们可以使用context.WithValue方法附加一对K-V的键值对，这里Key必须是等价性的，也就是具有可比性；Value值要是线程安全的。

这样我们就生成了一个新的Context，这个新的Context带有这个键值对，在使用的时候，可以通过Value方法读取ctx.Value(key)。

记住，使用WithValue传值，一般是必须的值，不要什么值都传递。
// 就是 k v 形式
```



上下文真是个好东西, 方便我们 协程隔离

```
之前写代码的过程中，发现大佬们在中间件解析用户uid 的时候都通过 constant.withValue 存储到上下文中，当时很不理解，为啥不直接 全局变量存储，后来想通了，go 中 包内变量多goroutine共享，很容易相互污染。
```







