---
title: Laravel 关于blade
date: 2018-10-10 16:45:23
tags: [PHP,Laravel]
categories: PHP
---

其实现在都是前后端分离的，应该蛮少用到模板引擎这种东西，作为上一个web时代的产物，其实模板引擎开发起来还是蛮快的，而且挺简单的。

<!--more-->

itbasic 用的也是模板引擎，名twig，其实和blade没有多少不同的地方，只是自己发现的twig新功能还比较少，只是一直在原先的基础上堆代码，感觉上在模板引擎里面还是不要做过多的逻辑判断，否则后面会越写越复杂，而且难于维护。



对于web网站，我们有很多页面，但这些页面大部分都是相同的？因为我们的布局需要相同，这个其实在写itbasic的时候应该能感觉到，我们似乎只需要填充中间的content部分就好了

```
<html>
    <head>
        <title>@yield('title','测试')</title>
    </head>
    <body>
        @yield('content')
    </body>

</html>
```

什么是yield ,就是让继承者可以填充的东西，继承者通过

```
@extends('layout.blade') //继承上一个模板
@section('title', 'cy')  //填充数据

@section('content')   //这种方式填充数据要有结束符号
	<h1>222</h1>
@stop
```



```
@include('default.ce')  // 包含额外的页面
//传递变量，include('default.ce', ['user'=>$user])
```



```
//判断
@if ($a > 0)
	<h1>cc</h1>
@endif

//循环
@foreach($counts as $count)
	<li></li>
@endforeach
```

