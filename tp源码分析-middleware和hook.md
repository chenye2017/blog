---
title: tp源码分析-middleware和hook
date: 2019-08-09 09:48:42
tags: [PHP,TinkPHP, 中间件, 钩子]
categories: PHP
---

这块的中间件主要值的是route 中间件，对于请求发送到controller之前进行一系列的判断，是否缺胳膊少腿不符合要求，如果不符合，返回false, 符合 next($response)

钩子就是流程执行到某个点，触发用户绑定的方法，之所以这么做是为了用户可以在框架的层面做某些修改，而不用修改框架源码。

<!--more-->

### Hook

```
echo 'start'."<br>";
 
        // 在触发钩子之前，绑定行为到指定钩子
        Hook::add('test_1','app\\test\\behavior\\BehaviorTest'); // 添加钩子处理
        Hook::add('test_2','app\\test\\behavior\\BehaviorTest');
 
        // 调用设置了钩子的函数来触发钩子，进行测试
        Hook::listen('test_1',['name'=>'cy']); // 设置钩子，并执行钩子绑定的函数方法，第二个变量是数组方式传入钩子函数需要的变量
         Hook::listen('test_2',['name'=>'cy']);
    
 class BehaviorTest {
    public function test_1($param) {
 
        var_dump($param['name'])
       
    }
 
    public function  test_2($param) {
   		var_dump($param['name'])
        
        }
     }   
       
```



php 因为是同步的，所以他这个钩子函数和别的语言理解起来还不一样，比如这个listen， 在监听的同时并且执行了后面类中绑定的方法，而不像一些异步语言，listen之后等待着事件的触发。

还有就是我们绑定的是类，一个类中会有多个方法，默认的初始方法是run ,这个类中还可以有多个方法，多个方法对应多个钩子，方法名和钩子名一样

分析一下Hook

tags   核心属性，不同的钩子对应不同的类（可以是多个类组成的数组）

bind   别名，也是给类绑定别名

portal  入口方法名称，默认run

app 应用对象，App的实例，也是单例模式



__construct

构造方法，注入App 实例 $app



portal

设置入口函数名称



alias

修改bind 属性，给行为（对应某个类中特定方法，也是唯一一个方法，所以这个behavior 方法就是是class）绑定别名



add

添加点对应的行为，行为可以是关联数组，key _overlay 会让添加的value 覆盖之前的value, key  first 会让添加的value 排在第一位



import

批量导入 点对应的行为，可以是让import 和之前的tags 标签合并，也可以是让之前的tags 覆盖import 导入的内容



get

获取tags 属性



listen 

其实就是执行某个点中绑定的类中的方法



exec

执行某个类中方法（方法名确定，就是portal）



execTag

执行绑定的类中的方法，先去执行listen 的点对应的method是否存在，如果能执行，执行默认方法 



### middleware

中间件用的设计模式是装饰器模式，每个装饰器类中只有一个方法，handle，第一个参数是Request请求，第二个参数是匿名函数 next， 还可以接受额外的参数，他这块也用的是反射，所以你如果没有添加第三个参数，在方法内部用func_get_args() 是获取不到额外的参数的，即使你在路由上middleware 传入了这个参数。因为反射没有获取到第三个参数的存在，所以执行的时候就不会传入。

```
<?php

namespace app\http\middleware;

class Check
{
    public function handle($request, \Closure $next, $pa1 = 1, $pa2 = 2) // 难道只能装一个参数吗
    {
        var_dump($pa1,'|', $pa2);exit;
        $request->name = 'cy';
        return $next($request);
    }
}

Route::get('/testhook', 'index/testhook')->middleware(['Check:name']);
```

如果我们在application下面的middleware.php 中写入，那么全局所有的route都会经过这个中间件













