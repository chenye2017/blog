---
title: go 包的学习
date: 2020-07-03 14:41:03
tags: [go]
categories: go
---

go 语言太底层，为了方便大家的日常，出现了很多包，对底层api 的封装，产生的作用类似php 中各个函数（php 的函数相较于go，都属于高阶的），学习他们，使用他们

<!--more-->

math/rand

随机数我们经常用到，php 中rand(), 就能生成一个随机数，但是go中不行，详见 [https://blog.sqrtthree.com/articles/random-number-in-golang/](https://blog.sqrtthree.com/articles/random-number-in-golang/)

这篇文章的解释。php 中也有seed 的概念，不太清楚php中默认seed 是什么，因为一直改变，导致rand() 产生的随机数一直在变，我们可以通过函数设置seed 固定，这样rand 产生的随机数就不变了

ps: seed 并不是 rand 产生随机数从上限的概念，应该是随机数算法中用到的。还有go rand产生的随机数好像都是从1 开始的，比如我们想从5 开始，可以给产生的随机数都 + 5， 就能满足需求了。