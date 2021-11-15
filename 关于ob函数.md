---
title: 关于ob函数
date: 2019-09-30 19:40:50
tags: [PHP, 缓存]
categories: PHP
---

在看源代码的时候还有phpexcel 的时候经常都会看到ob 函数，ob函数主要是用于处理php 的缓存的，默认cli 下ob 是关闭的，需要我们手动打开

[ob_flush和flush](http://www.laruence.com/2010/04/15/1414.html)

[echo](http://www.laruence.com/2010/12/17/1833.html)

[加速echo](http://www.laruence.com/2011/02/13/1870.html)

(之前看到的fastcgi-finish-request 和上面的缓存也是有一点关系的哦), 每当php 返回数据的时候，我们可以这么看待传输过程，首先是php把数据给webserver， webserver 再把数据给cli ，webserver 在传输给cli 的过程中php的处理进程并没有结束，所以这之间的传输效率也会影响php进程的存活时间，当这之中传输过慢的时候，就会导致大量的php进程存在（php维护的的数据库单例连接也就存在），有时候我们想尽快的结束这个php进程，可以让php 传输给webserver 的时候开启一个缓冲池，当缓冲池满的时候一次性推给webserver，然后fastcgi_finish_request ,这之后就和php进程没关系了，php就可以接受下一个请求或者断开都可以

<!--more-->