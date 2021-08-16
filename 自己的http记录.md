---
title: 自己的http记录
date: 2018-03-07 17:49:37
tags: http
categories: http
---

上周老师让我们写一个关于http的总结，记录下这些天来的自己关于http的认识。

<!--more-->

1.http请求动作

移动互联网时代，不像之前web时代，实现了前后端的分离，前端请求后端接口从之前基于soap变成调用基于rest的restul api接口，restful api是基于资源，他对于不同的http请求方式有不同的含义，最常见的get，post，put，delete, option,。传统的get和post区别主要是一个是url明文传递，另一个是加密后传递更安全。但是restful api中常见的4中请求动作分别代表一下含义
get ： 对于资源的请求
post ： 对于资源的创建
put ： 对于资源的更新
delete ： 对于资源的删除
例如 : get  /users/1  代表获取用户1的信息
post /users/1 创建
put /users/1  更新
delete /users/1  删除
这样的好处之一就是更有利于语义化，而且更能指定统一的标准, 比如laravel里面Route的resource方法就能根据资源users统一生成一堆的url，而不是通过不同的方法名来区分。
但比如传统表单提交只能实现get 和 post方式，如果我们直接修改提交方式会报错

```
<form action="/posts/62" method="POST">
```
我们只需要在里面增加一个隐藏的input
```
<input type="hidden" name="_method" value="PUT">
```
就可以改变原先的form表单提交方式了

2.content-type的作用

* 对于接口返回数据的处理

  以python flask为例

  ```
  from flask import Flask, make_response

  app = Flask(__name__)

  @app.route('/hello')
  def hello():
      response = make_response('<html></html>')
      return response

  app.run(host='0.0.0.0', debug=True)
  ```

  默认浏览器是按照html的格式解析的，上面当我们访问/hello的时候输出是空，因为html标签里面什么也没有

  ```
  app = Flask(__name__)

  @app.route('/hello')
  def hello():
      headers = {
          'content-type':'text/plain',
      }

      response = make_response('<html></html>')
      return response
  app.run(host='0.0.0.0', debug=True)
  ```

  但这样就能输出 

  ```
  <html></html>
  ```

  因为我们让浏览器按照字符串的形式来解析

  itbasic中也对header信息做了设置

  ```
  $response->header('Content-Type', 'text/json');
          $response->header('Access-Control-Allow-Origin', '*');
          return json_encode($res);
  ```

  这就是为什么我们每次调用接口的时候返回的都是json数据的原因

  但是当我们输出模板的时候，观察控制台返回的

  ```
  Content-Type: text/html; charset=UTF-8
  ```

  应该是模板在哪里进行了处理

- 常见的输出图片

content-type 属于返回内容的一部分，主要是用来告诉浏览器返回的数据应该怎么解析，比如你访问 homestead.app/1.jpg,并不是根据文件的后缀名解析的成相应的文件，而是根据resresponse headers里面的内容，[常见的mini type](http://tool.oschina.net/commons)。例如php输出一张图片

```
<?php
$fileres = file_get_contents('./1.jpg');
header('Content-type: image/jpeg');
echo $fileres;
```
- 不使用插件导出excel文件

其实这个作用还是挺大的，比如我们经常输出的excel，虽然php有个很强大的插件phpexcel，但是那种东西经常就内存溢出了，我们可以通过修改header，直接输出csv文件，而不用使用任何插件

```
/**
 * 导出excel(csv)
 * @data 导出数据
 * @headlist 第一行,列名
 * @fileName 输出Excel文件名
 */
function csv_export($data = array(), $headlist = array(), $fileName) {
  
    header('Content-Type: application/vnd.ms-excel');
    header('Content-Disposition: attachment;filename="'.$fileName.'.csv"');
    header('Cache-Control: max-age=0');
  
    //打开PHP文件句柄,php://output 表示直接输出到浏览器
    $fp = fopen('php://output', 'a');
    
    //输出Excel列名信息
    foreach ($headlist as $key => $value) {
        //CSV的Excel支持GBK编码，一定要转换，否则乱码
        $headlist[$key] = iconv('utf-8', 'gbk', $value);
    }
  
    //将数据通过fputcsv写到文件句柄
    fputcsv($fp, $headlist);
    
    //计数器
    $num = 0;
    
    //每隔$limit行，刷新一下输出buffer，不要太大，也不要太小
    $limit = 100000;
    
    //逐行取出数据，不浪费内存
    $count = count($data);
    for ($i = 0; $i < $count; $i++) {
    
        $num++;
        
        //刷新一下输出buffer，防止由于数据过多造成问题
        if ($limit == $num) { 
            ob_flush();
            flush();
            $num = 0;
        }
        
        $row = $data[$i];
        foreach ($row as $key => $value) {
            $row[$key] = iconv('utf-8', 'gbk', $value);
        }

        fputcsv($fp, $row);
    }
  }
```
- 文件的上传

之前在调用datrix的上传接口的时候，他要求传过来的数据文件得是form表单的g格式，可以通过

```
var writeData = new FormData();
```

3.http是无状态的协议（cookie 和 session）
无状态的协议代表我们在登录网站后，浏览器不知道我们在不同的网页间是同一个个体，这就得靠session和cookie了。
session和cookie都能保存用户信息，只是一个存储在客户端，另一个存储在服务器端。存储在客户端就说明用户可以进行修改，不安全，所以一般开启session后，服务器返回给用户一个session id,下次用户登录的时候，拿着这个session id 去服务器端获取session文件。
php里面默认的cookie 名字是PHPSESSID 里面装的是sessionid的内容(这个名字是可以改的，比如laravel框架里面默认的是laraval_session,这个也是可以通过配置文件改的)，只要拿着这个去服务器端sess_ + sessionid的内容（这个存储地址在php.ini里面有配置），就能找到对应的session，session里面的内容是序列化后的（如果客户端禁止了cookie，也可以把session id 放在url里面带过来）。默认这个session id的cookie是种在 域名下。比如 itbasic.datatom.com,这个域名下的所有网页都可以访问（只有同一个浏览器，如果是不同的浏览器，就获取不到这个cookie了··）

文件1 ：
```
<?php

session_start();

$_SESSION['username'] = 'cy';
```
文件2
```
<?php
session_start();
var_dump($_SESSION);
```
notice: **这个获取session的时候也要调用session_start（）函数，有时候因为框架的或者函数的封装，很容易忽略**

4.http 状态码
http状态码表示这个请求的状态，常用的有200成功 ，3xx 资源的重定向，4xx客户端的错误（404 请求的文件找不到 403 没有权限 401 没有登录）， 5xx服务器端错误（500 经常代码的错误  502 网关，经常nginx出错）。然后现在的restful api在我们返回客户端的请求内容的时候经常也会携带着一个状态码，比如 

```
return json_encode([
      'errorCode'=>10001,
      'errorMsg'=> 'xxx',
      'result'=>[]
])
```
errorCode一般是我们内部定义的错误状态码，可以展示给用户看，让用户更能清楚的了解问题的所在，举个例子就是文章的创建，可能title不合法，过长了，文章删除的时候没有传递id，都可以以一个具体位数开头的错误码，归于一类。
不管是http状态码，还是我们传递给用户的状态码，都是可以改变的，比如tp里面的json方法，第一个参数就是http状态码的参数，第二个参数才是要返回的具体的内容。

location也是返回信息中的一个字段，虽然状态码和我们的返回信息没有关系，即使正常调用接口我们返回一个404也是可以的，但是location必须得配合30x才能实现页面的跳转。

5.无连接
无连接的含义是限制每次连接只处理一个请求。服务器处理完客户的请求，并收到客户的应答后，即断开连接。采用这种方式可以节省传输时间。记得之前好像看过一篇文章说现在的http协议支持长连接了

```
Connection:keep-alive
```
在请求头里面包括这个东西，虽然自己平时好像从来也没感受到，但还是记录下

6.url

- url组成

统一资源定位符。比如http://itbasic.datatom.com:80。
http 是网络协议，默认端口80，如果不是80，需手动添加，比如我本地的xampp修改了http默认端口8080，访问的时候itbasic.app:8080才能访问，除了http，还有https 443， ssh默认端口22，访问这些服务的时候都要加上端口号，除了域名之外。其实域名只是为了让人们更好的记住，其实它本质上是翻译成ip的，打开cmd，ping itbasic.datatom.com 就能看到ip地址。ip地址代表服务器，一个服务器上可能提供多种服务，比如web服务，为了识别出不同的服务，我们用端口号进行标识，这和之前说的刚好吻合。

- 域名解析

其实当我们在浏览器中输入itbasic.datatom.com 的时候，浏览器会先去找比如windows host文件（经常我们本地测试的时候，瞎写的域名能访问就是这个道理，他不会去网络上进行解析，而是根据hosts文件中的配置进行ip地址转换，可以测试一下把www.baidu.com ip地址换成itbasic的，当然只有自己这台电脑访问百度会显示的是itbasic的主页），如果找不到，寻找浏览器缓存，再找不到会进行DNS域名解析，最终找到服务器，给他发送http请求，如果是传统的lamp架构，apache接受到请求会把通过模块加载的形式把关于php的内容传递给php进行解析，php会把其中关于数据库的内容，交给数据库处理，返回来的数据处理后统一返回给apache，再返回到浏览器进行渲染。现在lnmp，利用nginx，fpm对php请求进行统一处理，自己关于这方面内容现在还不足，了解还不是很深。

- 跨域

itbasic.datatom.com中 com是一级域名，datatom是二级域名，itbasic是三级域名，同属于datatom,这个二级域名下还会有很多，比如blog.datatom.com,51dan.datatom.com,域名的不同会导致出现跨域的问题，之前看网上处理方式主要分为两大类，一种是jsonp,另一种是返回的请求头中添加

```
'Access-Control-Allow-Origin', '*'
```

7.请求头
```
GET /562f25980001b1b106000338.jpg HTTP/1.1
Host    img.mukewang.com
User-Agent  Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.106 Safari/537.36
Accept  image/webp,image/*,*/*;q=0.8
Referer http://www.imooc.com/
Accept-Encoding gzip, deflate, sdch
Accept-Language zh-CN,zh;q=0.8

```
第一行get代表请求方式， 后面那个是请求的资源，最后是http的版本号，话说http 1.0 和 http1.1 使用时间好长
第二行host 代表主机名
第三行 表示 访问的主机的系统，还有浏览器，经常通过这个来判断是手机还是电脑登录。其实这个在服务器端也是可以获取的，通过打印 $_SERVER 数组。
第四行
第五行 上一个链接地址，经常用来进行回退。
这个请求头和后面传递的数据需要空一行。
这个请求头里面还可以放置很多别的数据，比如laravel里面表单为了放置csrf攻击，在里面放置了额外的参数
```
<meta name="csrf-token" content="{{ csrf_token() }}">
```
请求的时候建立连接还会有4次握手
1. 你在吗
2. 我在
3. 那我给你发数据了
4. 好的，你发吧
   虽然到今天还没用上过

8.返回信息
```
HTTP/1.1 200 OK
Date: Fri, 22 May 2009 06:07:21 GMT
Content-Type: text/html; charset=UTF-8

<html>
      <head></head>
      <body>
            <!--body goes here-->
      </body>
</html>

```
这个和上面的请求消息差不多，断开链接需要3次握手
1. 我要断开链接了
2. 好的 （我不发送数据）
3. 我也要断开连接了（这下两条都断开了）




9.https

https 默认端口443，其实他不是一种新的协议，他还是http协议，只是在外层包裹了ssl。

https传输数据比http安全大家都知道，但原因呢，是由于数据进行了加密。很早之前有种协议talent，他也是通信协议，默认端口23，但现在这个端口默认都被禁止掉了，原因就是他传输数据的过程是明文传输，导致别人用抓包工具抓到了请求，或者监听这个端口，直接就能获取客户端传递过来的数据。想象一下，钓鱼网站如果用这个，很容易就能获取到用户的密码，太不安全了，于是ssh诞生了（我猜的）。ssh用的地方很多，比如我们用xshell时候建立的连接，还记得我们第一次连接新的服务器的时候的弹窗吗

>一次性接受密钥

这是什么东西？这其实就是服务器的公钥。你发给服务器的内容，用服务器的公钥进行加密，服务器接收到你的内容用私钥进行解密，保证了数据传输的安全性（你发送的数据只有服务器能查看到，别人永远不能查看，因为你加密的媒介是计算机的公钥，能解密的只用服务器的私钥，而服务器的私钥只有你自己有，so，数据绝对安全。但有一点，数据的不可变性，就是数据的被篡改性不能得到保证）。或者你是否记得第一次对github进行提交的时候，或者换新的机器对github进行提交的时候，需要在github上贴你的公钥，为什么呢？因为我们git clone代码的时候，需要让github用我们的公钥加密，然后我们用自己的私钥进行解密（git clone 的时候的地址有ssh和https之分）

上面说的公钥加密，私钥解密舒服非对称加密，记得有RSA算法之类的。但是我们在数据通信的过程中如果一直用非对称方式进行数据传输，消耗会很大。于是我们用了另一种对称加密。对称加密顾名思义就是：同一种密钥进行加密解密，简单。可问题来了，这个密钥如果被获取到，加密就毫无意义了，我们该如何传递这个密钥给客户端。我们不能预先在所有的客户端都放置所有的密钥吧，这样不现实，也不能变动。

上面说的第一种非对称加密是公钥加密，私钥解密，保证数据的安全性。其实我们还可以通过私钥加密，公钥解密的方式，这样虽然不能保证数据的安全性（指的是被别人获取，因为公钥的公开性），但可以保证数据的不被篡改，因为私钥的私密性，一旦篡改内容，我们没法复原到私钥加密前的状态，我们只需要在私钥加密的内容中添加一些公共的东西，比如一个值，一个算法，这个值经过算法加密后的值，当我们通过公钥解密后这这个值加密后不等于这个算法加密后的值，那我们就能发现文件内容被篡改。
























