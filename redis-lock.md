---
title: redis lock
date: 2021-07-07 18:31:02
tags: [go, redis]
categories: go
---

分布式锁在目前我们多pod 的代码运行环境中经常用来控制并发问题，kafka 的reblance 的时候中也经常会因为同一条消息被多消费者消费到，需要分布式锁去控制，简单的分布式可以通过redis set 命令的 nx 去实现，但有一些问题是这单条命令不能解决的

<!--more-->

#### 任务执行时间过久，redis key 过期

我们可以通过 在lock 方法的时候通过启动一个timer 定时器，在ttl 过了2/3 后主动续约ttl， 让其不过期

