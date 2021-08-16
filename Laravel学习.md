---
title: Laravel学习
date: 2018-07-29 23:41:50
tags: [PHP,Laravel]
categories: PHP
---

关于Laravel 的一些学习，主要是对一些知识点的理解，不包含用法，具体用法参考laravel-china, wiki ,真的是太详细了

<!--more-->

# 目录结构

传统的mvc 中只要注意下面几个文件夹就好了，可能很多人和我当初刚接触laravel 一样，对于 controller和model不放在一层感觉很奇怪，routes 在更外层，其实我们这里面可以这样理解，http更相当于我们传统的controller，为了实现控制器中逻辑处理和参数验证分开，我们分出了request 和 controller，中间级。和 http 同层的还有model (模型)， jobs(队列任务)，listeners之类的，暂时用的还很少，以后再补充

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20190726100149.png)





# 设计模式

依赖注入: 我们平时写的代码大部分包括以下几种模式

1. 我定义的函数，我调用
2. 框架定义的函数，我调用
3. 我定义的函数，框架调用

第一种很好理解，第二种我们在调用框架定义的函数的时候注意一定要传入完备的参数，否则会报错（很多时候我们在直接修改框架源码的时候一定要注意这点，因为框架定义的函数，如果你没搞清楚框架在哪些地方调用了，直接修改，也没给参数默认值的话，很容易出错，我们可以通过phpstorm去查找这个函数的调用地址，当然如果是那种没有命名空间的函数的话，很容易找不到，比如itbasic中）

我们这里主要讲一下第三种，我定义的函数，框架调用，主要有两种。一种是路由中callable, 比如klein 路由中都是匿名函数执行的形式，他默认有4个参数，\$request, \$response, \$service, \$app, 剩下的就是路由参数了，我对这种执行方式的猜测就是执行的时候直接传入这4个类实例，再merge 路由参数 一起传入我们的匿名函数中，因为php在定义的函数中没有对应参数，执行的时候传入，不会报错，这就方便了我们可以随意的传入前面4个参数，用到的就传入，不用到就不传入，但是坏处就是对于后面的路由参数，只能在前面传入4个参数之后加上，参数值和参数顺序完全对应！！！！

而依赖注入就不需要，laravel的路由有两种方式可以实现，一种和klein一样是匿名函数的形式，另一种指定函数中方法名，注意这和我们itbasic中klein一点都不一样，itbasic中调用控制器中方法本质上还是在匿名函数中实例类，再调用方法，并传入参数，你会发现itbasic控制器中参数的顺序并不能改变，但是laravel中却可以，只要类名对应上，底层应该是用反射类获取参数名称，然后实例化，传入，所以依赖注入的核心应该是就是反射 ，之前我的想法总是总是容器，或者是方法名中可以直接使用request 和 response对象就是依赖注入，其实这都是错误的想法，你随便在一个不是路由对应方法的函数中定义request和response对象，调用的时候一定会报没有传入对应参数的错误



# Router

laravel 包含了两种请求方式的处理，一种是类似前后端不分离的形式，传统的web开发，一种是专门用来处理api,  两种请求处理过程有哪些不同还没有研究，现在发现的是一个http请求，会先走web.php, 再走api.php

```
匿名函数形式
```


```
// 类名，方法名
<?php
Route::get('/', 'StaticPagesController@home')->name('home');
Route::get('/help', 'StaticPagesController@help')->name('help');
Route::get('/about', 'StaticPagesController@about')->name('about');
```

这个name是用来命名的，命名有什么用，看下面

route('help')  => 这个help 是路由命名，user.help 更为准确

route('help',[1,2])  => 多个参数传递数组



这个函数的参数，通过下面这个函数，配合配置得环境变量，可以生成前端可以访问路径的，如果很直白的写url，以后url改动的时候但凡用到的地方，都需要改动

# Input



# Validator

Validator 调用的是facade ， facade 调用是 validator 的factory 类， 其中的make方法生成一个validator 实例，我们可以调用validate 方法去验证，抛出Validation 异常，我们可以通过对validate方法捕获异常，或者判断validator实例执行validate方法之后， 再调用errors()->all() 方法获取错误信息

```
$validator = Validator::make($request->all(), [
   'title' => 'bail|required|string|between:2,32',
   'url' => 'sometimes|url|max:200',
   'picture' => 'nullable|string'
], [
   'title.required' => '标题字段不能为空',
   'title.string' => '标题字段仅支持字符串',
   'title.between' => '标题长度必须介于2-32之间',
   'url.url' => 'URL格式不正确，请输入有效的URL',
   'url.max' => 'URL长度不能超过200',
]);
try {
$validator = $validator->validate(); // 注意不要链式调用，我们需要的是实例对象
} catch (ValidationException $e) {
  
}
// dingo api 对这个异常处理了
// 默认的web 也会有自带的异常处理机制，默认最基础的是绑定信息到模板变量$errors 中， 这个变量内容其实就是 $validator->errors(), 通过 $errors->all() 我们可以获取所有的错误信息
```

为了实现请求验证和控制器的解耦，laravel 把请求单独成requests，直接在controller中注入这个 request 就能实现自动验证，可这样的话，就不知道request 验证什么时候执行的，怎么捕获这个异常（后面可以看request源码中的验证过程），那只能在统一定义异常处理那进行处理 (web 和 api 还不在同一个位置好像，还有怎么在除了控制器的地方直接返回数据给前端，而不接着往下走return)



# Exception handle







