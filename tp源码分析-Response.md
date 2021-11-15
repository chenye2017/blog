---
title: tp源码分析-Response
date: 2019-09-30 17:36:34
tags: [PHP,ThinkPHP]
categories: PHP
---

框架可能让你对PHP 怎么给webserver 返回内容产生了误解 ！！！

<!--more-->

PHP 是怎么把数据返回给前端的？通过response 对象？因为我们一直在用框架，框架中对于这些返回都是包裹成了response 对象，然后我们想要改变返回的内容，直接修改这个response对象就好了，比如在itbasic上

```
$router->respond(['GET', 'POST'], '/[:controller]/[:func]', function ($request, $response) {
    $res  = $obj->$func($request, $response);

    //$response->header('Content-Type', 'application/json');
    $response->header('Access-Control-Allow-Origin', '*');

    $response->json($res);
    // return json_encode($res);
});
```

上面就是一种很常见的route 组件的使用，很多时候我们可能以为代码就在这结束了，殊不知，这块只是我们定义个的一个callback， 框架代码在执行了这段之后只是完成了response 对象的封装，之后他们还会处理response 对象，比如调用php原生header 处理response的header 属性，echo response 对象的data 属性。

又或者下面这种

```
$router->respond(['GET', 'POST'], '/[:controller]', function ($request, $response, $service) {
    return $obj->$func($request, $response);
});
```

直接返回数据，并没有返回response 对象，那后面程序就可以根绝默认的配置比如content-type html, 然后echo  返回的string。



上面两种方式大概就是一般路由的使用方法，下面我们来看看tp 中到底是怎么处理的。我们查看一下入口文件index.php

```
// 执行应用并响应
Container::get('app')->run()->send();
从容器中获取 App.php 的实例，执行run 方法
```

run 方法很长，我们筛选这次用的几个重要点
```
public function run()
    {
        try {
            // 初始化应用
            $this->initialize(); // 这个里面会帮我们加载自定义的路由中间件到 middleware 的queue 属性中

····
        $this->middleware->add(function (Request $request, $next) use ($dispatch, $data) {
            // var_dump('ppp');
           // var_dump($data);
            // 没错误，直接走的run 方法
            // 这个地方因为 next 用不到了，所以
            return is_null($data) ? $dispatch->run() : $data;
        });
        // 这个地方往middleware 的queue 属性中加入一个callback， 如果之前有路由中间件，这块就是前面需要需要的参数 $next


        $response = $this->middleware->dispatch($this->request); // 中间件的执行，包括控制器的执行都在这里面

        // 监听app_end
        $this->hook->listen('app_end', $response); // 钩子 app end 执行

        return $response;
    }
```

所以呢，通过上面的代码可以看出来在tp 中路由中间件和控制器其实是同一个层面的，他们的核心就是返回一个response， 比如中间件过滤失败 ，我们也需要返回一个response，万万不可以在路由中间件里面返回false， 因为后面的send() 方法需要response 类来调用，控制器中之所以可以直接返回数据是上面那个回调函数写的好，\$dispatch->run() 中有对控制器返回内容的自适应，如果控制器返回的是一个response，那就直接让他调用send 方法，否则会根据配置文件包装一个response 方法返回。 

但凡我们想停止路由中间件的执行，我们可以不执行 $next() 方法，直接返回response， 这就他就会直接调用 send 方法，而不去处理控制器中内容。tp这块的原理和 laravel 还是不同的，laravel 中是通过array_reduce 不断的去执行绑定匿名函数，tp 中是通过对middleware的queue属性数组中的匿名函数不断shift， 每一个匿名都依赖下一个匿名，但要是当前匿名不需要比如执行下一个匿名 \$next() 的执行，那只会传入一个function， 而没有实际执行。这在我思考为什么 控制器执行的时候 ，明明queue 数组中没有办法shift 匿名函数了，却没有报错时候产生了极大的困惑

```
call_user_func('test', [$callback])  // 不执行，除非test方法中 $callback() 主动调用
call_user_func($callback, [$parama]) // 执行
```



接下来我们分析一下 Response 类，

首先是最重要的方法，

* send

hook 当中的listen_send ,属于常规操作，在我们response 解析之前

然后是

* getContent 

对于我们的控制器返回进行处理，生成最后要echo 的内容。其中的output方法就是我们response 文件夹下所有继承response 类要实现的方法，比如json 类，会json_encode, 要是普通的类， response的data 属性就直接赋值给content 属性了，默认content-type 是html.

思考一下为什么我们在控制器中返回arr 有时候会出错？因为我们最后的数据都要echo ,然后都必须要转换成string， 当我们使用默认的配置文件的时候，如果我们直接返回一个数组，默认生成的类是response 类，他的output 方法就是直接data 给content， 这样就导致我们echo 了一个arr ，能不出错嘛

后面一段是开启

* trace 调式

trace 调试我感觉方便的一点就是能看到执行的sql 语句，这在开发中是很必要的。

再后面就是头信息的处理，

* 调用php 原生header

再后面就是

*  echo data

在后面就是

* fastcgi_finish_request(); 

执行这个方法，这个方法的好处就是在我们尽快把客户端要的内容返回后，php 脚本还能继续执行， [参考鸟哥这篇文章](http://www.laruence.com/2011/04/13/1991.html)

再后面就是客户端输出完成后服务端自己的一些收尾工作



再来整体分析下response 这个类



* __construct 初始化

其中content-type charset 拼接很重要，标准的就是拼在一起，有时候当你没有识别json的，想想客户端传过来的content-type 是否多了 charset  utf8 信息



* create 生成一个response 

这个就相当于一个factory， 根据type的不同，生成不同的response， 注意我们控制器中返回的response 或者官方对于控制器中 string 这种的返回包装的response ，都是new 的，而不是从app 容器中取得



* send  解析response对象，给webserver 返回数据



* output 对于data 依据content-type 进行处理，转成对应的string, 比如 json 转string



* sendData  就是最后一步，echo 数据



* options  一些额外的参数配置，比如tp 中，可以对于json_encode 添加参数 ，json_encode 之后汉字我们仍旧能识别

```
options([
        'var_jsonp_handler'     => 'callback',
        'default_jsonp_handler' => 'jsonpReturn',
        'json_encode_param'     => JSON_PRETTY_PRINT,
    ]);
```

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20190930193953.png)



* data 设置data 属性



* allowCache  设置是否允许缓存request 内容，如果允许的话，会配置header头信息



* header 配置header 头信息



* content 把content 的内容转换成 string



* code 设置http 状态码



* lastModified 设置上次修改时间



* expires  设置过期时间



* etag  设置etag



* cacheControl  设置cacheControl



* nocache 设置不缓存



* contentType 设置contentType



* getHeader  设置请求头



* getData  获取 data属性



* getContent 类似上面的content 方法



* getCode 获取http 状态码



* debugInfo



补充：

因为今天在看response类，他对很多header头部信息处理过，比如allowCache, 大致了解几个新的header 头 [https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching_FAQ](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching_FAQ)

cache-control  这个我们经常看到，no-cache,但这个其实并不是不缓存的意思（强制确认缓存的意思），no-store 才是不缓存的意思。其中还有max-age 属性，代表这个缓存最大活多久，must-revalidate, 代表使用之前要检验是否过期了。

expires 代表缓存过期的时间



今天在控制器中设置header 头发现不生效，原因有两个，

1. php的header 蛮反人类的，这样用 header("Content-Type:application/json"),竟然是字符串。
2. 后面response send 中header 会覆盖掉控制器中的header
3. 那为啥文件结尾header 没生效呢，因为response send 方法调用了 fastcgi_finish_request(), 结束了相应





