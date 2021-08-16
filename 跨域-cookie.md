---
title: 跨域 cookie
date: 2018-06-06 18:30:54
tags:
categories:
---

cookie，对于传统的php开发web必不可少，为什么呢，因为http是无状态协议，为了让浏览器在不同的tab页面记住我这个人，我们通过让同一个浏览器在访问同一个网址的时候都使用同一个钥匙，这个钥匙就是cookie，这个钥匙能打开服务器端特定的箱子，就是session，这个session里面保存了完整的用户信息，这样就能让浏览器记住我的不同tab页对应的是哪一个人了。

<!--more-->

cookie,对于传统的lamp架构，在服务器端想获取可以通过$_COOKIE[]数组获取到，但是对于swoole这种自己搭建web服务的东西，只能通过request请求获取，其实lamp里面本质也是这样，也是把apache或者nginx获取到的请求内容放到了php的\$_COOKIE里面，类似的\$_SERVER, \$_GET,这些在swoole里面都是永不了得，你可以把http->request->get 的值在请求到来的时候给与\$_GET数组，这样就能在swoole搭建的web服务里面和普通的lamp架构一样使用\$__GET数组了。正因为swoole的这些特性，像如果在swoole里面使用session登录机制需要自己去实现，还有对于一些框架的集成，因为外部传入的参数不在那些数组中了，所以需要对框架进行修改，让框架能获取到对应的参数，保证框架的运行，比如：swoole集成slime框架(swoole在客户端设置cookie也得用它自带的方法而不是setcookie)。

回归正题，通过上述，我们也只cookie是一个数组，那其中哪把钥匙是用来打开session的呢，答案是PHPSESSID,这个值是可以设定的，在php.ini,或者通过php的session函数来获取到。

对于session，我觉得他更像是一个模块的扩展，他不仅仅是提供一系列的简单的函数，这些函数其实还有具体的活动的封装。比如开启session的时候，session_start(),他做的东西有很多，首先检测客户端是否有 phpsessid，如果没有，通过函数setcookie，在客户端种植，其实底层就是通过在返回的header头信息中加入set-cookie，如果已经有phpsessid，会通过寻找客户端对应的session，获取里面的信息。那问题又来了，这个phpsessid的cookie是怎么传送给服务端的呢，我不记得我有加入这个cookie啊，一般情况下，在发送http请求的时候，默认都会带上浏览器的cookie。如果禁用了还可以通过url传递哦，话说从url中获取phpsessid也是在session_start()函数中实现的，可见这个函数的复杂性，和普通的函数还是不一样的，里面有具体的逻辑。

对于cookie的设置，也是有跨域问题的，a.test.com下的cookie,在b.test.com下面不能获取到，但是如果在test.com下面就能获取到了，为什么呢？看看跨域吧。不信？你可以设置下，在/path页面下设置另一个path的cookie，然后去对应的path下刷新，看是否有，答案是肯定有的啦。所以cookie的设定。path是不影响的。

既然能设定，自然就能删除，设定和删除可以归为一类操作（直接把过期时间设定成当前时间-100就可以删除了，注意cookie有max-age存在的时间和expire过期时间之分，php的setcookie设定的时间是expire时间，所以需要在过期时间+time()!!之前就犯了这个错误）

其实php的setcookie也能验证这点。在控制器中调用setcookie，如果不传path，默认是当前path，但也可以加path,比如/。

但需要注意的是不同path下面不能相互读取!!!怎么试验呢，可以在chrom的application下面写,然后在当前path写同一个域名下的另一个path，刷新cookie，发现另一个页面下面的有这个设置的cookie，请求的时候也能带上，但是自己这个设定页面没有，所以有时候如果setcookie没错误，但也没出现cookie，可以考虑下是不是设置路径错误了，虽然设置上了，但是看不见。这种情况特别容易出现在前后端分离的情况下，前端在80端口下，后端在8080端口下，后端api设定cookie是装在了8080那个域名下面，所以你无论怎么刷新前端页面都不会有这个cookie，但是没关系，当你请求后端接口的时候，请求会自动加上！！但要注意，请求的时候，http请求只能把他当前能看到的cookie带上，对于他看不见的肯定不会带上。

