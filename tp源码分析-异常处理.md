---
title: tp源码分析-异常处理
date: 2019-08-30 13:05:13
tags: [PHP,异常处理]
categories: PHP
---

[参考文章](https://segmentfault.com/a/1190000014977430)

上面这篇文章写得很好，让我对异常处理除了 try catch 之外有了更深的认识，大致记录一下对php 异常处理的理解以及结合tp框架的源码分析

<!--more-->

### PHP 处理错误的两大类方式

首先我们要明白php 的异常处理分成两个独立的大块，一个是php自身的标准处理，就是我们经常error_reporting ， display_error 的设置（呈现形式就是界面上的php warning 信息， php fatal error 这种）；另一个是我们通过try catch 捕获异常进行处理（我们可能在catch 中 记录 $e->getMessage()） . 我们要清楚的是这两块对错误的处理级别是相互独立的，互不影响。比如error_reporting(0), 不对任何异常进行处理（ps: -1 ,类似 E_ALL ）,try catch 还是能捕捉到我们的exception。

```
<?php
error_reporting(0); // 0 的时候没有任何错误信息， -1 的时候会打印出 //fatal_error
test();

try {
  throw new Exception()
} catch (Exception $e) {
  var_dump($e->getMessage())
}

```

### PHP 标准错误处理 error_reporting() 参数的意义

error_reporting(), 不带任何参数，返回的是错误处理级别，带参数意思是设置错误处理级别

error_reporting(E_ALL ^ E_NOTICE) , ^ 是异或，代表除了E_NOTICE 都进行处理， 或者

error_reporting(E_ALL & ~E_NOTICE), & 是同或， ~代表取反，和上面是同一个意思，E_ALL是一个常量，数值是 E_NOTICE 那些的结合体

error_reporting(E_NOTICE | E_WARNING), 包括notice 和 warning 错误  



### PHP的错误等级

```
# 系统级用户代码的一些错误类型 可由 try ... catch ... 捕获
E_PARSE          解析时错误 语法解析错误 少个分号 多个逗号一类的 致命错误
E_ERROR          运行时错误 比如调用了未定义的函数或方法 致命错误

# 系统级用户代码的一些错误类型 可由 set_error_handler 捕获处理
E_WARNING        运行时警告 调用了未定义的变量
E_NOTICE         运行时提醒                  
E_DEPRECATED     运行时已废弃的函数或方法

# 用户级自定义错误 可由 trigger_error 触发 可由 set_error_handler 捕获处理
E_USER_ERROR      用户自定义错误 致命错误 未处理也会导致程序退出
E_USER_WARNING
E_USER_NOTICE
E_USER_DEPRECATED

==========================开发中常遇到/不常遇到分割线=======================

# Zend Engine 内部的一些错误 应该也能通过 try ... catch ... 捕获 略难测试
E_CORE_ERROR
E_CORE_WARNING
E_COMPILE_ERROR
E_COMPILE_WARNING

#编码标准化警告(建议如何修改以向前兼容)
E_STRICT          部分 try ... catch ... 部分 set_error_handler
E_RECOVERABLE_ERROR
```

这个地方需要我们注意的是结合PHP7 的 error 和 exception 来看，php7 的error 和 exception 都来自throwable， 所以我们在 catch  throwable都能捕获到，他和上述php 标准错误级别的关系如下

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20190830133738.png) 

也就是说我们平时throw 的都是实际的error， 需要注意的是上面的 E_USER_NOTICE,并不指的是我们用户写的代码触发的notice， 或者throw 的 notice (用户throw的都是error)， 这个实际上我也没见过，但我们可以通过trigger_error 函数模拟出来



### PHP 各种错误的触发

```
<?php
error_repoting(0)
aa
```

为什么上述代码还是会抛出parse error， 因为我们在当前脚本中存在着错误，还没有执行error_reporting 就会报错，但有个方法可以让上述代码不报错，就是把 aa 写在另一个文件中，然后require 这个文件 。我们php-fpm 模式的本质，其实就是 php index.php, 所以我们在 index.php 中可以写上很少的代码，至少保证这个index.php 的正确性。

（可能是因为php是解释性的代码，只会检测require 的文件是否存在，不会检查被包含的该文件正确性，但是会对require 的文件中 require 语句正确性检测，蛮奇怪的,而且require 的文件不存在会抛出两个错误，一个是warning ，一个是fatal error， 这个warning 会被set_error_handle 处理，这个fatal error 不能被捕获， 而且竟然能通过error_reporting 屏蔽，感觉就是严重性介于 文件本身 parse error 和 try catch 捕获的 error 之间）



### 关于PHP 异常处理的几个函数

+ try catch , 不用做过多的介绍了，老朋友，可以捕获 throw 的所有内容，相当于 php 标准错误的fatal_error, 但不是fatal_error 就等于他，比如 require 的文件错误不存在，就不能被捕获


+ set_error_handle, 和上面的try catch 互补，try catch 捕获的都是 E_ERROR 级别，对于那些warning ,notice 就靠这个东西来处理了，他有个返回值，当返回false 的时候，错误还是会提交个php 标准错误处理的 （也就是处理错误的流程是先php代码处理，然后php 自身处理， 我觉得php自身处理应该就是日志记录，还有display error 。 php 的错误日志一定要注意位置，很多时候不是php.ini 里面配置的位置，而是虚拟主机 apache  或者 nginx 里面配置的日志位置）

+ set_error_handle要比想象中的强大多了，我们可以在这之中接受 erroNo, erroStr, 封装成 exception 抛出去，那么就会在当时被set_error_handle 捕获的地方抛出 （注意几个常用的 error的数字 ，比如 e_error 1, e_warning, e_notice.  -1 代表 e_all,  0 代表关闭所有的错误报告）（adodb即使开启了错误处理，也只能触发 e_user 级别错误，所以只能被set_error_handle 处理，set_error_handle 再抛出异常，才能被捕获）

+ set_exception_handle, 对于没有用try catch 捕获的错误（注意是没有用！！，而不是不能捕获的，捕获的内容等同于try catch）, 会走到这个里面进行处理，但是这个里面处理后程序就停止了，而try catch 处理之后程序还能接着往下走（ps:其实就我们平常而言，很多时候我们捕获了异常，除非在foreach 中，如果在平时逻辑中，也就直接return 给客户端，停止后面逻辑了）

+ register_shutdown_function， 注册一个程序结束的时候执行的函数，就是无论是崩溃啊，还是正常结束，都在最末尾执行的方法

+ error_get_last, 如果一个错误走到了php标准处理那块，就能通过这个获取错误内容，如果一个错误提前被捕获处理了，那这块就是空的

+ throw new Exception , throw new Error, 通过throw 异常，测试try catch 和 set_exception_handle

+ trigger_error, 对于 warning ，notice 那些错误的触发，测试set_error_handle(当然也可以通过 var_dump 一个不存在的变量来触发)

  ​

  tp 所有的异常处理都会走到Error 的 appException 方法中,最后给到 getExceptionHandler方法

  1. 首先最基本的提点try catch 的，会被 set_exception_handle 捕获，包括 \Exception 和 \Throwable, 交给 appException
  2. warning 和user 级别，会被当做 errorException 抛出，转而到 appExcepiton 当中
  3. register_shutdown_function , 那些没有被捕获到的错误，通过error_get_last 捕获，然后判断是否要塞入 appException 中


### TP 框架中的异常处理

我们对于异常的处理最好写在代码的最前面，否则error 触发之后，该错误之后的代码都不会生效。

tp 在base.php 中 Error::register();  ，注册了代码的错误处理。(核心文件 Error.php)

我们大致测试一下cli 模式下的异常处理

+ register

```
		error_reporting(E_ALL); // php 标准错误处理级别
        set_error_handler([__CLASS__, 'appError']); // 对于warning 级别的处理
        set_exception_handler([__CLASS__, 'appException']); // 用户自定义的try catch , 会在执行完之后停止，但是不会交给php 标准错误处理
        register_shutdown_function([__CLASS__, 'appShutdown']); // 感觉就是只要注册了，不管怎样都会执行
```

+ appError 

接受php 传递过来的参数，生成ErrorException 交给 appException 处理

+ appException

首先判断捕获的异常是error 还是exception， 如果是error ，统一转换成 errorException 处理

然后获取异常处理类，官方是Handle.php, 因为tp 允许用户重写这个类，如果重写了，需要继承官方的Handle 类，重写的方法是render， 框架会自动调用（为什么要重写这个类呢，因为比如记日志这种统一的方法就不需要重写了，框架会自动调用原先自己的方法）。如果没有重写，单单的给了匿名函数，会把这个匿名函数保存到 handle 的render 属性中，后期执行。如果匿名函数都没有，那就执行官方自己的render 方法

+ appShutDown

如果这个错误没有被处理直接交给了php标准错误，会抛出一个ErrorException, 交给自定义的异常处理，否则就记录下日志就好了

+ isFatal

需要抛出的错误级别

+ setExceptionHandle

设置自定义异常处理级别







