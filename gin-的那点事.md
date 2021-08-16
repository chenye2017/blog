---
title: gin 的那点事
date: 2020-07-17 10:45:57
tags: [go,gin]
categories: go
---

最近沉迷王者荣耀，好久没有认真学习go了，不应该，记录下gin 学习的那点事

<!--more-->

abort ，next 方法

gin 的中间件和别的中间件还不一样，gin 的中间件没有next 也能正常执行，next 只是控制自己的代码和中间件的代码执行顺序。中间件中用return 不能停止中间件的执行，只能用abort 方法才能停止后面程序的执行



