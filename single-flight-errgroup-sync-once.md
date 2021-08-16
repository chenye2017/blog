---
title: 'single flight, errgroup, sync once'
date: 2021-05-25 15:35:10
tags: [golang, package]
categories: golang
---

一些源码包的解读，帮助更好的写代码

<!--more-->

## sync.Once##

关于sync once, https://geektutu.com/post/hpg-sync-once.html 这篇文章讲的很详细，代码很简洁，对于我们平时用的需要关注的就是我们每一种类型的动作，如果都想使用sync.Once, 一定要分开单独定义 once 对象

```
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

package sync

import (
	"sync/atomic"
)

// Once is an object that will perform exactly one action.
type Once struct {
	// done indicates whether the action has been performed.
	// It is first in the struct because it is used in the hot path.
	// The hot path is inlined at every call site.
	// Placing done first allows more compact instructions on some architectures (amd64/x86),
	// and fewer instructions (to calculate offset) on other architectures.
	done uint32
	m    Mutex
}

// Do calls the function f if and only if Do is being called for the
// first time for this instance of Once. In other words, given
// 	var once Once
// if once.Do(f) is called multiple times, only the first call will invoke f,
// even if f has a different value in each invocation. A new instance of
// Once is required for each function to execute.
//
// Do is intended for initialization that must be run exactly once. Since f
// is niladic, it may be necessary to use a function literal to capture the
// arguments to a function to be invoked by Do:
// 	config.once.Do(func() { config.init(filename) })
//
// Because no call to Do returns until the one call to f returns, if f causes
// Do to be called, it will deadlock.
//
// If f panics, Do considers it to have returned; future calls of Do return
// without calling f.
//
func (o *Once) Do(f func()) {
	// Note: Here is an incorrect implementation of Do:
	//
	//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	//		f()
	//	}
	//
	// Do guarantees that when it returns, f has finished.
	// This implementation would not implement that guarantee:
	// given two simultaneous calls, the winner of the cas would
	// call f, and the second would return immediately, without
	// waiting for the first's call to f to complete.
	// This is why the slow path falls back to a mutex, and why
	// the atomic.StoreUint32 must be delayed until after f returns.

 // 如果不分开定义，这里可能被公用
	if atomic.LoadUint32(&o.done) == 0 {
		// Outlined slow-path to allow inlining of the fast-path.
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

注意golang 中很多这种包都是内存型，一旦重启，变量就会被重置。对于一些只能执行一次的方法，我们还得用redis 的分布式锁。

我们用 sync.once 只是为了过滤掉大部分的重复动作的。





## single flight##

 single flight 变量group 的定义，就需要我们全局定义了，这点一定要注意，我们不要在controller 接口里面随意定义一个这样的变量。 查看源码，核心struct 是个map， 所以我们定义的key 一定不能重复，否则可能会出现别人的结果返回。single flight 主要用来就是 防止比如单个内存过期，并发太高，相同结果的请求都打到db 上。（如果活动针对单人，一种因为设置了相同的过期时间，导致缓存同时过期（雪崩），这时候 single flight 不能解决，可以给每个缓存设置一个base 时间 + rand 时间，防止同时过期。对于运营活动，上述雪崩情况可能真的会发生！！！！



single flight 的异常处理很复杂，暂且没有仔细研读，说一下其中几个核心的知识点。



```
// Group represents a class of work and forms a namespace in
// which units of work can be executed with duplicate suppression.
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // lazily initialized
}

我们平时 var g  singleflight.Group 就是申明了一个这个变量 。为了控制某个时间段（后端处理逻辑花费的时间）我们http server 接受到的请求不被同时执行，相同的key 只被执行一次，我们把接受到的请求都存入这个group 的 m 属性 map 中。

// call is an in-flight or completed singleflight.Do call
type call struct {
	wg sync.WaitGroup

	// These fields are written once before the WaitGroup is done
	// and are only read after the WaitGroup is done.
	val interface{}
	err error

	// forgotten indicates whether Forget was called with this call's key
	// while the call was still in flight.
	forgotten bool

	// These fields are read and written with the singleflight
	// mutex held before the WaitGroup is done, and are read but
	// not written after the WaitGroup is done.
	dups  int
	chans []chan<- Result
}

在看一下 map 中 每个key 对应的结构体，wg 主要是为了控制 同一个key 对应的一批goroutine 的阻塞，

val, err 是这批groutine 执行后共享的结果， 当我们调用 Do 方法获取结果的时候用的就是这个值。

forgotten 是主动删除掉这个key ，比如阻塞的时间过长，我们不希望这么久时间内的请求共享返回结果，我们希望2s 后的请求用新的执行结果，可以启动个 time.after 2 second, 利用select 接受值，然后forgotten 这个 key 。 （这种工作模式经常使用，比如下面的errgroup 的cancel， 即使我们返回err 触发了 g.cancel ，但是我们的 errgroup 组中的goroutine 也没有被终止，可以这样写）


package main

import (
	"context"
	"errors"
	"fmt"
	"golang.org/x/sync/errgroup"
	"time"
)

func T21() chan int {
	time.Sleep(2 * time.Second)
	fmt.Println("success")
	c1 := make(chan int, 1)
	c1 <- 1
	return c1
}

func main() {
	var g *errgroup.Group
	var ctx context.Context
	g, ctx = errgroup.WithContext(context.Background())

	g.Go(func() error {
		select {
		case <-ctx.Done():
			fmt.Println("errr")
		case <-T21(): // 这个可以利用single flight 中控制超时的方式改写， t21()协程启动，然后从外部传入一个 chan 变量
			fmt.Println("success")
		}
		return nil
	})

	g.Go(func() error {
		return nil
		return errors.New("test")
	})

	_ = g.Wait()

	fmt.Println("end")
}

跑题了 ┭┮﹏┭┮

// Result holds the results of Do, so they can be passed
// on a channel.
type Result struct {
	Val    interface{}
	Err    error
	Shared bool
}

这个很好理解，需要注意的就是shared， 但凡阻塞，所有元素的shared 都是true， 不存在第一个是 false， 后面是true


// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
// The return value shared indicates whether v was given to multiple callers.
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
	g.mu.Lock() // 写锁
	if g.m == nil {
		g.m = make(map[string]*call) // 方便我们不用声明
	}
	if c, ok := g.m[key]; ok {
		c.dups++  // 这个就是用来控制是否是共享的
		g.mu.Unlock()
		c.wg.Wait() // wait 可以很多次

		if e, ok := c.err.(*panicError); ok {
			panic(e)
		} else if c.err == errGoexit {
			runtime.Goexit()
		}
		return c.val, c.err, true
	}
	c := new(call)
	c.wg.Add(1) // 只执行一次，所以add 一次
	g.m[key] = c
	g.mu.Unlock()

	g.doCall(c, key, fn)
	return c.val, c.err, c.dups > 0
}

上面的do 方法是阻塞的，直接返回值，想不阻塞只能用chan ，就是下面要介绍的doChan 的方式。其实转过头来和我们errgroup 控制超时是一样的机制。我们用select 控制超时，我们只是提前返回了，但是没有中断程序的执行，这是我们要避免的一个误区，之前遇到一个协程泄漏的问题就是正常情况下这个协程不阻塞，非正常情况下这个协程中chan 数据没人消费了。


// doCall handles the single call for a key.
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
	normalReturn := false
	recovered := false

	// use double-defer to distinguish panic from runtime.Goexit,
	// more details see https://golang.org/cl/134395
	defer func() {
		// the given function invoked runtime.Goexit
		if !normalReturn && !recovered {
			c.err = errGoexit
		}

		c.wg.Done()
		g.mu.Lock()
		defer g.mu.Unlock()
		if !c.forgotten {
			delete(g.m, key)
		}

		if e, ok := c.err.(*panicError); ok {
			// In order to prevent the waiting channels from being blocked forever,
			// needs to ensure that this panic cannot be recovered.
			if len(c.chans) > 0 {
				go panic(e)
				select {} // Keep this goroutine around so that it will appear in the crash dump.
			} else {
				panic(e)
			}
		} else if c.err == errGoexit {
			// Already in the process of goexit, no need to call again
		} else {
			// Normal return
			// 这个 chans 只有在 dochans 的时候才会有切片数组 > 0 ,才会往里面塞入值
			for _, ch := range c.chans {
				ch <- Result{c.val, c.err, c.dups > 0}
			}
		}
	}()

	func() {
		defer func() {
			if !normalReturn {
				// Ideally, we would wait to take a stack trace until we've determined
				// whether this is a panic or a runtime.Goexit.
				//
				// Unfortunately, the only way we can distinguish the two is to see
				// whether the recover stopped the goroutine from terminating, and by
				// the time we know that, the part of the stack trace relevant to the
				// panic has been discarded.
				if r := recover(); r != nil {
					c.err = newPanicError(r)
				}
			}
		}()

		c.val, c.err = fn()
		normalReturn = true
	}()

	if !normalReturn {
		recovered = true
	}
}

我们看看异步的do chan

func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result {
	ch := make(chan Result, 1) // 为了防止协程泄漏
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		c.dups++
		c.chans = append(c.chans, ch) // 这个地方是 chan 独有的
		g.mu.Unlock()
		return ch
	}
	c := &call{chans: []chan<- Result{ch}} // 此时slice 的数量已经是1 了 
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	go g.doCall(c, key, fn) // do 方法中没有启动协程，相比较我的写法更加优雅，为了非阻塞，只能这样协程的写

	return ch
}


// Forget tells the singleflight to forget about a key.  Future calls
// to Do for this key will call the function rather than waiting for
// an earlier call to complete.
func (g *Group) Forget(key string) {
	g.mu.Lock()
	if c, ok := g.m[key]; ok {
		c.forgotten = true
	}
	delete(g.m, key)
	g.mu.Unlock()
}

现在想了下， forget 这的需要，比如堵塞1个请求堵塞了1min， 这1min 的请求都失败了，运维那边估计炸了，还是用 do chan 的方式吧，用来代替 forget 的作用。
```



## errgroup##

这里介绍两个errgroup ，一个是原生的，另一个是 b站开源kratos 工具包中 errgroup

```
// Copyright 2016 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Package errgroup provides synchronization, error propagation, and Context
// cancelation for groups of goroutines working on subtasks of a common task.
package errgroup

import (
	"context"
	"sync"
)

// A Group is a collection of goroutines working on subtasks that are part of
// the same overall task.
//
// A zero Group is valid and does not cancel on error.
type Group struct {
	cancel func()

	wg sync.WaitGroup

	errOnce sync.Once
	err     error
}

// WithContext returns a new Group and an associated Context derived from ctx.
//
// The derived Context is canceled the first time a function passed to Go
// returns a non-nil error or the first time Wait returns, whichever occurs
// first.
func WithContext(ctx context.Context) (*Group, context.Context) {
	ctx, cancel := context.WithCancel(ctx)
	return &Group{cancel: cancel}, ctx
}

注意下这个cancel 被调用， 只是 ctx.done 中会有数据，协程自身并不受影响。所以当这个协程中的ctx 好无用处的时候，这个ctx 会继续执行，并不能达到errgroup的初衷

1. 协程中返回 error(这个肯定能达到)
2. 但凡一个协程用问题，结果直接返回 （注意之前的work协程停止是绝对不可能的，没有任何人有这个权利，我们能做到的就是对其中用到ctx 的方法，比如http 请求，直接 超时的，并不执行，这也得益于人家包的编写，去监听了这个ctx 的变动，ctx 本质上只是一个连接串，携带着一些request id 这样的东西）


// Wait blocks until all function calls from the Go method have returned, then
// returns the first non-nil error (if any) from them.
func (g *Group) Wait() error {
	g.wg.Wait() // 很普通的wait
	if g.cancel != nil {
		g.cancel()  // 正常执行完，释放掉cancel 
	}
	return g.err
}

// Go calls the given function in a new goroutine.
//
// The first call to return a non-nil error cancels the group; its error will be
// returned by Wait.
func (g *Group) Go(f func() error) {
	g.wg.Add(1)

	go func() {
		defer g.wg.Done()

		if err := f(); err != nil {
			g.errOnce.Do(func() { // once
				g.err = err
				if g.cancel != nil {
					g.cancel()
				}
			})
		}
	}()
}
```



看下b站版本的



b站支持两个版本的errgroup ，一种是原生的，我们不要调用 GOMAXPROCS(n int) ，方法

另一种是利用缓冲channel， 实现协程数量的限制。

```
// Copyright 2016 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Package errgroup provides synchronization, error propagation, and Context
// cancelation for groups of goroutines working on subtasks of a common task.
package errgroup

import (
	"context"
	"fmt"
	"os"
	"runtime"
	"strings"
	"sync"
	"time"

	
)

// A Group is a collection of goroutines working on subtasks that are part of
// the same overall task.
//
// A zero Group is valid and does not cancel on error.
type Group struct {
	err     error
	wg      sync.WaitGroup
	errOnce sync.Once  

	workerOnce sync.Once // 设置最大协程数的时候只执行一次
	ch         chan func(ctx context.Context) error // 有缓冲的chan， 缓冲数对应设置的最大协程个数， 里面的元素很容易看出来，就是我们 go 调用时候的func
	chs        []func(ctx context.Context) error  // chan 中存不下的func， 先用数组存着，后面从数组里面取出往 chan 里面塞

	ctx    context.Context
	cancel func()
}

// WithContext create a Group.
// given function from Go will receive this context,
func WithContext(ctx context.Context) *Group {
	return &Group{ctx: ctx}
}

// WithCancel create a new Group and an associated Context derived from ctx.
//
// given function from Go will receive context derived from this ctx,
// The derived Context is canceled the first time a function passed to Go
// returns a non-nil error or the first time Wait returns, whichever occurs
// first.
func WithCancel(ctx context.Context) *Group {
	ctx, cancel := context.WithCancel(ctx)
	return &Group{ctx: ctx, cancel: cancel}
}

func (g *Group) do(f func(ctx context.Context) error) {
	ctx := g.ctx
	if ctx == nil {
		ctx = context.Background()
	}
	var err error
	defer func() {
		if r := recover(); r != nil {
			buf := make([]byte, 64<<10)
			buf = buf[:runtime.Stack(buf, false)]
			err = fmt.Errorf("errgroup: panic recovered: %s\n%s", r, buf)
		}
		if err != nil {
			g.errOnce.Do(func() {
				pl := fmt.Sprintf("%s ERROR:errgroup call panic: %v\n", time.Now().Format("2006-01-02 15:04:05"), err)
				//忽略panic告警
				if !strings.Contains(err.Error(), context.DeadlineExceeded.Error()) {
					fmt.Fprintf(os.Stderr, pl)
				}
				xlog.Error(pl)
				g.err = err
				if g.cancel != nil {
					g.cancel()
				}
			})
		}
		g.wg.Done()
	}()
	err = f(ctx)
}

// GOMAXPROCS set max goroutine to work.
func (g *Group) GOMAXPROCS(n int) {
	if n <= 0 {
		panic("errgroup: GOMAXPROCS must great than 0")
	}
	g.workerOnce.Do(func() {
		g.ch = make(chan func(context.Context) error, n)
		// 这个地方起对应协程个数
		for i := 0; i < n; i++ {
			go func() {
				for f := range g.ch {
					g.do(f)
				}
			}()
		}
	})
}

// Go calls the given function in a new goroutine.
//
// The first call to return a non-nil error cancels the group; its error will be
// returned by Wait.
func (g *Group) Go(f func(ctx context.Context) error) {
	g.wg.Add(1)
	if g.ch != nil {
	 // 限制协程数的时候走这里
		select {
		case g.ch <- f:
		default:
			g.chs = append(g.chs, f) // 装不下的塞到数组里面
		}
		return
	}
	// 如果非限制协程数的时候走这里，类似原生errgroup
	go g.do(f)
}

// Wait blocks until all function calls from the Go method have returned, then
// returns the first non-nil error (if any) from them.
func (g *Group) Wait() error {
	if g.ch != nil {
	  // 这块会一直堵塞着，直到所有程序都被执行
		for _, f := range g.chs {
			g.ch <- f
		}
	}
	// 原生的errgroup
	g.wg.Wait()
	if g.ch != nil {
		close(g.ch) // let all receiver exit
	}
	if g.cancel != nil {
	    // 我们新生成的context， 记得都要cancel
		g.cancel()
	}
	return g.err
}
```

