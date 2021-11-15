---
title: robfig/cron 源码解读
date: 2021-05-30 18:21:52
tags: [go,源码]
categories: go
---

经常会有些定时任务需要执行，以前php中，我们可能就写一个脚本，然后在服务器上写好crontab 定时执行，又或者通过开源的工具 xxljob 执行php 的命令， go 因为我们可以启动一个常驻内存的服务，goroutine 中引入工具包robfig/cron 提供一个类似上面xxljob 一样的对外服务，我们可以添加兼容crontab 写法的任务到服务中，然后执行。了解一下源码是怎么写的。

<!--more-->

先声明几个用到定时任务可能遇到的误区:

1. \* */1 * * * , 分时日月周，每小时执行一次，他代表的是  1， 2， 3，4，····这些整数点执行，而不是从现在开始的 每隔一小时执行一次，下一次的运行时间是通过解析 crontab 表达式，算出下次执行的时间，算出差值，然后启动一个time.after, select {} 鉴定这些channel， 获取到值之后，遍历任务，获取next time ，判断否需要有执行的，如果有执行的直接启动一个goroutine执行。
2. 同一种类型任务，比如每秒执行，前一个还没执行完，下次是否会执行。正常情况下是会执行的，因为这些任务都是异步的，每个任务都是goroutine，所以哪怕你执行1辈子，也不会影响同一类型任务下次执行。



chain.go

```
这个文件的作用就是一些公用方法包装我们的任务

type JobWrapper func(Job) Job

其实这个类型蛮常见的，洋葱模型，web的中间件一般都需要这种类型。

// Then decorates the given job with all JobWrappers in the chain.
// 洋葱模型的核心
//
// This:
//     NewChain(m1, m2, m3).Then(job)
// is equivalent to:
//     m1(m2(m3(job)))
func (c Chain) Then(j Job) Job {
	for i := range c.wrappers {
		j = c.wrappers[len(c.wrappers)-i-1](j)  // 靠前封装的元素最后执行
	}
	return j
}


// Recover panics in wrapped jobs and log them with the provided logger.
// 因为goroutine中 的panic 外层不能捕获，还会引起服务的重启，所以但凡我们启了goroutine 最好 recover 下
func Recover(logger Logger) JobWrapper {
	return func(j Job) Job {
		return FuncJob(func() {
			defer func() {
				if r := recover(); r != nil {
					const size = 64 << 10
					buf := make([]byte, size)
					buf = buf[:runtime.Stack(buf, false)]
					err, ok := r.(error)
					if !ok {
						err = fmt.Errorf("%v", r)
					}
					logger.Error(err, "panic", "stack", "...\n"+string(buf))
				}
			}()
			j.Run()
		})
	}
}

// DelayIfStillRunning serializes jobs, delaying subsequent runs until the
// previous one is complete. Jobs running after a delay of more than a minute
// have the delay logged at Info.
// 这个就是把异步任务整成同步任务了，每个类型任务同一时间我们只能有一个执行，控制这个唯一性的就 //  mutex 互斥锁 ，如果下一次执行和上次执行中间差距1分钟，就记录条日志
func DelayIfStillRunning(logger Logger) JobWrapper {
	return func(j Job) Job {
        // 注意下变量位置	， 这个在 c.chain.Then 的时候   
		var mu sync.Mutex
		return FuncJob(func() {
			start := time.Now()
			
			mu.Lock()
			defer mu.Unlock()
			
			if dur := time.Since(start); dur > time.Minute {
				logger.Info("delay", "duration", dur)
			}
			j.Run()
		})
	}
}


// SkipIfStillRunning skips an invocation of the Job if a previous invocation is
// still running. It logs skips to the given logger at Info level.
// 注意下 这个ch 的声明时期，如果是在new cron 的时候，作为options参数带进去的，那所有的任务都  // 会变成变成同步，而且是上一个任务如果没有完成的话，下一个任务会被跳过。
func SkipIfStillRunning(logger Logger) JobWrapper {
	var ch = make(chan struct{}, 1)
	ch <- struct{}{}
	return func(j Job) Job {
		return FuncJob(func() {
			select {
			case v := <-ch:
				j.Run()
				ch <- v
			default:
				logger.Info("skip")
			}
		})
	}
}

```



constantdelay.go

```
package cron

import "time"

// ConstantDelaySchedule represents a simple recurring duty cycle, e.g. "Every 5 minutes".
// It does not support jobs more frequent than once a second.
type ConstantDelaySchedule struct {
	Delay time.Duration
}

// Every returns a crontab Schedule that activates once every duration.
// Delays of less than a second are not supported (will round up to 1 second).
// Any fields less than a Second are truncated.
func Every(duration time.Duration) ConstantDelaySchedule {
	if duration < time.Second {
		duration = time.Second
	}
	// second 只能支持到秒级别
	return ConstantDelaySchedule{
		Delay: duration - time.Duration(duration.Nanoseconds())%time.Second,
	}
}

// Next returns the next time this should be run.
// This rounds so that the next activation time will be on the second.
// 
func (schedule ConstantDelaySchedule) Next(t time.Time) time.Time {
// 减掉这个纳秒时间，为什么time.now 没有毫秒 和 微妙时间，感觉就可能是毫秒和微妙包含在里面了
	return t.Add(schedule.Delay - time.Duration(t.Nanosecond())*time.Nanosecond)
}
```



cron.go

```
核心任务管理器了

package cron

import (
	"context"
	"fmt"
	"sort"
	"sync"
	"time"
)

// Cron keeps track of any number of entries, invoking the associated func as
// specified by the schedule. It may be started, stopped, and the entries may
// be inspected while running.
type Cron struct {
	entries   []*Entry  // 单个任务
	chain     Chain // 可以统一包一层任务链
	stop      chan struct{}  // 停止channel， struct{} 是为了节省内存空间
	add       chan *Entry  // 如果任务启动了， 添加到 cron 里面就会变成线程不安全，所以我们通过chan 去操作
	remove    chan EntryID // 同上
	snapshot  chan chan []Entry // 读取此时所有的任务， 这个库功能还是蛮多的，就缺一个后台界面了
	running   bool  // 是否在执行
	logger    Logger  // 日志
	runningMu sync.Mutex 
	location  *time.Location // 时区
	parser    Parser // 解析器
	nextID    EntryID // 自增id
	jobWaiter sync.WaitGroup // 为了让任务平稳的结束
}

// Job is an interface for submitted cron jobs.
type Job interface {
	Run() // 我们添加的func 必须要实现这个方法
}

// Schedule describes a job's duty cycle.
type Schedule interface {
	// Next returns the next activation time, later than the given time.
	// Next is invoked initially, and then each time the job is run.
	Next(time.Time) time.Time // 内部自己的通过这个解析成下次任务执行的时间戳，计算出差值，然后启动一个time.after 为了执行
}

// EntryID identifies an entry within a Cron instance
type EntryID int // 任务id ，注意不是分布式的哦

// Entry consists of a schedule and the func to execute on that schedule.
// 单个任务
type Entry struct {
	// ID is the cron-assigned ID of this entry, which may be used to look up a
	// snapshot or remove it.
	ID EntryID

	// Schedule on which this job should be run.
	Schedule Schedule  

	// Next time the job will run, or the zero time if Cron has not been
	// started or this entry's schedule is unsatisfiable
	Next time.Time

	// Prev is the last time this job was run, or the zero time if never.
	Prev time.Time

	// WrappedJob is the thing to run when the Schedule is activated.
	WrappedJob Job

	// Job is the thing that was submitted to cron.
	// It is kept around so that user code that needs to get at the job later,
	// e.g. via Entries() can do so.
	Job Job
}

// Valid returns true if this is not the zero entry.
func (e Entry) Valid() bool { return e.ID != 0 }

// byTime is a wrapper for sorting the entry array by time
// (with zero time at the end).
// 其实我们这块排序这个slice 可以不用写的这么麻烦，直接 sort.StableSlice 就可以了。
type byTime []*Entry

func (s byTime) Len() int      { return len(s) }
func (s byTime) Swap(i, j int) { s[i], s[j] = s[j], s[i] }
func (s byTime) Less(i, j int) bool {
	// Two zero times should return false.
	// Otherwise, zero is "greater" than any other time.
	// (To sort it at the end of the list.)
	if s[i].Next.IsZero() {
		return false
	}
	if s[j].Next.IsZero() {
		return true
	}
	return s[i].Next.Before(s[j].Next)
}

// New returns a new Cron job runner, modified by the given options.
//
// Available Settings
//
//   Time Zone
//     Description: The time zone in which schedules are interpreted
//     Default:     time.Local
//
//   Parser
//     Description: Parser converts cron spec strings into cron.Schedules.
//     Default:     Accepts this spec: https://en.wikipedia.org/wiki/Cron
//
//   Chain
//     Description: Wrap submitted jobs to customize behavior.
//     Default:     A chain that recovers panics and logs them to stderr.
//
// See "cron.With*" to modify the default behavior.
func New(opts ...Option) *Cron {
	c := &Cron{
		entries:   nil,
		chain:     NewChain(),
		add:       make(chan *Entry),
		stop:      make(chan struct{}),
		snapshot:  make(chan chan []Entry),
		remove:    make(chan EntryID),
		running:   false,
		runningMu: sync.Mutex{},
		logger:    DefaultLogger, // 默认loger 是不输出info的
		location:  time.Local, // 本地时间，解析 时间戳成 字符串的时候，如果没有设置时区，用的就是这个时区
		parser:    standardParser,  // 标准解析， 不支持 second，可以自己构造，比如那个withsecond 方法，但如果用那个second option 更方便的感觉
	}
	
	// options 模式
	for _, opt := range opts {
		opt(c)
	}
	return c
}

// FuncJob is a wrapper that turns a func() into a cron.Job
type FuncJob func()

func (f FuncJob) Run() { f() }

// AddFunc adds a func to the Cron to be run on the given schedule.
// The spec is parsed using the time zone of this Cron instance as the default.
// An opaque ID is returned that can be used to later remove it.
// 这个蛮常见的模式，add func 其实就是一次内心转换，类似http server 添加请求
func (c *Cron) AddFunc(spec string, cmd func()) (EntryID, error) {
	return c.AddJob(spec, FuncJob(cmd))
}

// AddJob adds a Job to the Cron to be run on the given schedule.
// The spec is parsed using the time zone of this Cron instance as the default.
// An opaque ID is returned that can be used to later remove it.
func (c *Cron) AddJob(spec string, cmd Job) (EntryID, error) {
	schedule, err := c.parser.Parse(spec)
	if err != nil {
		return 0, err
	}
	return c.Schedule(schedule, cmd), nil
}

// Schedule adds a Job to the Cron to be run on the given schedule.
// The job is wrapped with the configured Chain.
 // 添加 任务 entry 进 manger
func (c *Cron) Schedule(schedule Schedule, cmd Job) EntryID {
  // 线程不安全，互斥锁
	c.runningMu.Lock()
	defer c.runningMu.Unlock()
	c.nextID++
	entry := &Entry{
		ID:         c.nextID,
		Schedule:   schedule,
		WrappedJob: c.chain.Then(cmd),
		Job:        cmd,
	}


	if !c.running {
		c.entries = append(c.entries, entry)
	} else {
		c.add <- entry
	}
	return entry.ID
}

// Entries returns a snapshot of the cron entries.
func (c *Cron) Entries() []Entry {
	c.runningMu.Lock()
	defer c.runningMu.Unlock()
	if c.running {
	   // 这块写的很绕，上面都是投递个任务就行了，就不需要后续结果了
	   // 这个需要获取结果，我们先把一个chan （其实就当做一个变量就行了塞入任务队列中, 用来装我们的结果快照）
	   //  然后等待结果内容，其实这块就是把同步用异步的方式写了，如果想支持2个并发，可以把1缓冲改成2缓冲
		replyChan := make(chan []Entry, 1)
		c.snapshot <- replyChan
		return <-replyChan
	}
	return c.entrySnapshot()
}

// Location gets the time zone location
func (c *Cron) Location() *time.Location {
	return c.location
}

// Entry returns a snapshot of the given entry, or nil if it couldn't be found.
func (c *Cron) Entry(id EntryID) Entry {
	for _, entry := range c.Entries() {
		if id == entry.ID {
			return entry
		}
	}
	return Entry{}
}

// Remove an entry from being run in the future.
func (c *Cron) Remove(id EntryID) {
	c.runningMu.Lock()
	defer c.runningMu.Unlock()
	if c.running {
		c.remove <- id
	} else {
		c.removeEntry(id)
	}
}

// Start the cron scheduler in its own goroutine, or no-op if already started.
func (c *Cron) Start() {
	c.runningMu.Lock()
	defer c.runningMu.Unlock()
	if c.running {
		return
	}
	c.running = true
	go c.run()
}

// Run the cron scheduler, or no-op if already running.
// 非协程方式，自己就能阻塞住
func (c *Cron) Run() {
	c.runningMu.Lock()
	if c.running {
		c.runningMu.Unlock()
		return
	}
	c.running = true
	c.runningMu.Unlock()
	c.run()
}

// run the scheduler.. this is private just due to the need to synchronize
// access to the 'running' state variable.
func (c *Cron) run() {
	c.logger.Info("start")

	// Figure out the next activation times for each entry.
	now := c.now()
	for _, entry := range c.entries {
		entry.Next = entry.Schedule.Next(now) // 如果不run 没有next
		c.logger.Info("schedule", "now", now, "entry", entry.ID, "next", entry.Next)
	}

	for {
		// Determine the next entry to run.
		sort.Sort(byTime(c.entries))

		var timer *time.Timer
		if len(c.entries) == 0 || c.entries[0].Next.IsZero() {
			// If there are no entries yet, just sleep - it still handles new entries
			// and stop requests.
			timer = time.NewTimer(100000 * time.Hour)
		} else {
			timer = time.NewTimer(c.entries[0].Next.Sub(now))
		}

       // 这个地方用for 循环，是因为这里面有些chan 可以直接continue ，因为任务计划 [] entries 没有改动，  而不需要重新排序浪费性能
		for {
			select {
			case now = <-timer.C:
				now = now.In(c.location)
				c.logger.Info("wake", "now", now)

				// Run every entry whose next time was less than now
				for _, e := range c.entries {
					if e.Next.After(now) || e.Next.IsZero() {
						break
					}
					c.startJob(e.WrappedJob)
					e.Prev = e.Next
					e.Next = e.Schedule.Next(now)
					c.logger.Info("run", "now", now, "entry", e.ID, "next", e.Next)
				}
			
			// 运行的时候添加任务
			case newEntry := <-c.add:
				timer.Stop()
				now = c.now()
				newEntry.Next = newEntry.Schedule.Next(now)
				c.entries = append(c.entries, newEntry)
				c.logger.Info("added", "now", now, "entry", newEntry.ID, "next", newEntry.Next)
			// 快照
			case replyChan := <-c.snapshot:
				replyChan <- c.entrySnapshot()
				continue
			// 停止chan
			case <-c.stop:
				timer.Stop()
				c.logger.Info("stop")
				return
			// 删除计划任务chan
			case id := <-c.remove:
				timer.Stop()
				now = c.now()
				c.removeEntry(id)
				c.logger.Info("removed", "entry", id)
			}

			break
		}
	}
}

// startJob runs the given job in a new goroutine.
func (c *Cron) startJob(j Job) {
	c.jobWaiter.Add(1)
	go func() {
		defer c.jobWaiter.Done() // wait group 是真的好用
		j.Run()
	}()
}

// now returns current time in c location
// 我是感觉这个地方如果不是转换成时间戳，用不用location 无所谓
func (c *Cron) now() time.Time {
	return time.Now().In(c.location)
}

// Stop stops the cron scheduler if it is running; otherwise it does nothing.
// A context is returned so the caller can wait for running jobs to complete.
func (c *Cron) Stop() context.Context {
	c.runningMu.Lock()
	defer c.runningMu.Unlock()
	if c.running {
		c.stop <- struct{}{}
		c.running = false
	}
	ctx, cancel := context.WithCancel(context.Background())
	go func() {
	   // 平滑重启，结束所有任务
		c.jobWaiter.Wait()
		// 这个cancel 主要是为了这些异步任务结束完了能拿到通知
		cancel()
	}()
	return ctx
}

// entrySnapshot returns a copy of the current cron entry list.
// 很简单的遍历
func (c *Cron) entrySnapshot() []Entry {
	var entries = make([]Entry, len(c.entries))
	for i, e := range c.entries {
		entries[i] = *e
	}
	return entries
}

// 删除任务
func (c *Cron) removeEntry(id EntryID) {
	var entries []*Entry
	for _, e := range c.entries {
		if e.ID != id {
			entries = append(entries, e)
		}
	}
	c.entries = entries
}

```



option.go  很简单，就是一些配置项

```
package cron

import (
	"time"
)

// Option represents a modification to the default behavior of a Cron.
type Option func(*Cron)

// WithLocation overrides the timezone of the cron instance.
func WithLocation(loc *time.Location) Option {
	return func(c *Cron) {
		c.location = loc
	}
}

// WithSeconds overrides the parser used for interpreting job schedules to
// include a seconds field as the first one.
func WithSeconds() Option {
	return WithParser(NewParser(
		Second | Minute | Hour | Dom | Month | Dow | Descriptor,
	))
}

// WithParser overrides the parser used for interpreting job schedules.
func WithParser(p Parser) Option {
	return func(c *Cron) {
		c.parser = p
	}
}

// WithChain specifies Job wrappers to apply to all jobs added to this cron.
// Refer to the Chain* functions in this package for provided wrappers.
func WithChain(wrappers ...JobWrapper) Option {
	return func(c *Cron) {
		c.chain = NewChain(wrappers...)
	}
}

// WithLogger uses the provided logger.
func WithLogger(logger Logger) Option {
	return func(c *Cron) {
		c.logger = logger
	}
}

```



loger.go 这个很经典的loger

```
package cron

import (
	"io/ioutil"
	"log"
	"os"
	"strings"
	"time"
	
)

// DefaultLogger is used by Cron if none is specified.
var DefaultLogger Logger = PrintfLogger(log.New(os.Stdout, "cron: ", log.LstdFlags))

// DiscardLogger can be used by callers to discard all log messages.
var DiscardLogger Logger = PrintfLogger(log.New(ioutil.Discard, "", 0))

// Logger is the interface used in this package for logging, so that any backend
// can be plugged in. It is a subset of the github.com/go-logr/logr interface.
type Logger interface {
	// Info logs routine messages about cron's operation.
	Info(msg string, keysAndValues ...interface{})
	// Error logs an error condition.
	Error(err error, msg string, keysAndValues ...interface{})
}

// PrintfLogger wraps a Printf-based logger (such as the standard library "log")
// into an implementation of the Logger interface which logs errors only.
// 包装成一个interface
// loginfo 如果是true，才会打印info 内容，
// 默认的包装器，不打印info 内容
func PrintfLogger(l interface{ Printf(string, ...interface{}) }) Logger {
	return printfLogger{l, false}
}

// VerbosePrintfLogger wraps a Printf-based logger (such as the standard library
// "log") into an implementation of the Logger interface which logs everything.
func VerbosePrintfLogger(l interface{ Printf(string, ...interface{}) }) Logger {
	return printfLogger{l, true}
}

type printfLogger struct {
 // 这个接口 代表了printf 方法， go 官方库实现了这个方法
	logger  interface{ Printf(string, ...interface{}) }
	logInfo bool
}

// 自带的结构体，实现了额info 和 error 方法
func (pl printfLogger) Info(msg string, keysAndValues ...interface{}) {
	if pl.logInfo { // default printf 因为是false , 所以不打印
		keysAndValues = formatTimes(keysAndValues)
		pl.logger.Printf(
			formatString(len(keysAndValues)),
			// 这块这有一个msg 占位
			append([]interface{}{msg}, keysAndValues...)...)
	}
}

func (pl printfLogger) Error(err error, msg string, keysAndValues ...interface{}) {
	keysAndValues = formatTimes(keysAndValues)
	pl.logger.Printf(
		formatString(len(keysAndValues)+2), // 为什么 + 2 ，是因为formatstring 构造了 key value 的形式，多了 error key 和 error value
		append([]interface{}{msg, "error", err}, keysAndValues...)...)
}

// formatString returns a logfmt-like format string for the number of
// key/values.
// 构造 formate string
func formatString(numKeysAndValues int) string {
	var sb strings.Builder
	sb.WriteString("%s") // msg
	
	// 除了第一个msg 不是成对的剩下都是成对的
	
	if numKeysAndValues > 0 {
		sb.WriteString(", ")
	}

	for i := 0; i < numKeysAndValues/2; i++ {
		if i > 0 {
			sb.WriteString(", ")
		}
		sb.WriteString("%v=%v")
	}

	return sb.String()
}

// formatTimes formats any time.Time values as RFC3339.
func formatTimes(keysAndValues []interface{}) []interface{} {
	var formattedArgs []interface{}
	for _, arg := range keysAndValues {
		if t, ok := arg.(time.Time); ok {
			arg = t.Format(time.RFC3339) // 时间都以这种格式解析
		} 
		formattedArgs = append(formattedArgs, arg)
	}
	return formattedArgs
}

```



parser.go， spec.go 这两个文件都是关于 crontab 的解析，很难看不懂啦



