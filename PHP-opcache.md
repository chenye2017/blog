---
title: PHP opcache
date: 2018-05-25 15:38:57
tags:
categories:
---

opcache是什么东西，我也不知道，我只知道开启他能加速我接口的返回速度。

<!--more-->

原理：大概的意思就是把php的脚本一次性全存入内存中，然后每次读取脚本直接从这里面取，而不用每个连接都读取一份脚本。

但这样就存在修改数据后服务器识别不到，可以配置参数，多少秒服务器检查一次，但一般生产环境直接把配置成0，就是间隔时间是0，单位是s，然后停止检查，而开发环境参数配置成1，一直检查，这样就不会存在修改文件，没有效果了。

还可以手动清除，必须脚本中写入

```
opcache_reset()
```

但需要注意的就是不同模式下储存脚本是在不同的内存空间中，比如cli下执行脚本文件，和通过浏览器这种mod_php清除是不一样的，所以浏览器访问的话就通过浏览器的方式去执行脚本吧。

当然，如果和我们swoole那样，每次修改代码后重启web服务器，那是根本不存在这种情况的。

话说偶然间发现代码的不同方式发布

```
1.类似前端代码那种，每次新代码的发布都会重新打个包，然后把站点目录通过软连接指向这个新包
2.我们现在用的，通过在服务器上更新svn上的代码，来发布新的代码
（傻孩子，其实真正的发布应该都是前端那种发布方式，我们这种git 直接更新的方式也不能做到平滑重启）
```



首先运行环境是php7.014,线上环境是php5.6,效果不是十分明显，还有opcache_reset() byte-code不是十分好用。

首先得开启这个扩展，在phpinfo中查找opcache或者 php -m 中查找opcache，其实直接php -m就好了，会有个很显眼的ZendOpcache,

安装方法就是因为默认编译的时候大部分都有--enable-opcache,或者phpinfo页面上可以看见编译的参数，直接在php.ini 里面开启zend_extension=opcahce.so就好了，或者下载文件源码编译。

或者重新编译一下php（这个忘记了）

或者可以像pgsql那样去php源码包里面找源文件进行编译？这样可行吗，没有试过。

参数

```
opcache.validate_timestamps=1 //是否检测修改
opcache.revalidate_freq=0//单位时间
opcache.memory_consumption=64  //memory_consumption 这个参数很好理解，代表这块内存区开辟的大小，另外需要注意不同 PHP SAPI 内存区不是共享的，就是说同一个 PHP 文件，运行在命令行模式或者 PHP-FPM 模式下，对应的 byte-code 会存储在不同的内存区中。
opcache.max_accelerated_files=4000 //如果命中率不搞，可以适当提升这个值，直到1。
opcache.opcache.fast_shutdown=1  //modern php上面说这个写1就可以了
```

（opcache_get_configuration()和 and opcache_get_status()）获取配置信息和运行信息，比如了解那些文件被缓存了、使用了多少内存、内存命中率等等。

感觉就是基于这两个函数出了两个opcache的项目，一个是单页面，一个比较复杂

1.[PeeHaa / OpCacheGUI](https://github.com/PeeHaa/OpCacheGUI)

2.[rlerdorf / opcache-status](https://github.com/rlerdorf/opcache-status)

单页面的部署直接放在网站根目录下就好了，就理解成phpadmin那种网点就好了，利用的就是php的函数获取php的运行环境，然后可视化展示出来



opcache_invalidate()，这个函数就是更新特定的文件缓存，没去试验过，因为项目小啦，直接重启web server。

