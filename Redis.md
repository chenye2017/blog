---
title: Redis
date: 2018-04-23 20:54:46
tags: [PHP,Redis,缓存]
categories: Redis
---

像虎扑这类电商网站，基本上所有的数据都塞到redis 里面 ！！！

缓存，对于现代web很重要，首先他比较快嘛，因为是在内存中的，像很多功能也是通过缓存来实现的，比如点赞功能之类的，记录下常用的redis命令。因为是写php的，所以举得例子大部分还是以php的redis扩展为参照，其实大部分都一样。php 利用redis 有两个扩展，一个是php redis ,用c 写的，比较常用。一个是 predis , 用php写的一个redis 扩展包。

<!--more-->

端口号：6379

密码：可以设置，也可以不设置。itbasic上没有设置，但只允许本机才能连接服务器上的redis。 $redis->auth(); 一般好像不需要用户，或者用户名: 密码 这样。

启动：安装目录下的redis-server,最好后台挂起，要不然这个cmd就没法用了

连接： 安装目录下的redis-cli , 这个客户端可以连接各地的redis-server, 

redis-cli  -h  配置ip地址 -p 配置端口号  -a 配置密码

常见数据类型：1.string 2.hash 3. list 4. set 5.sorted set

通用命令：

keys * : 获取所有的key



### String

get(),根据key值获取

```
class \Lib\Redis extends \Redis  //因为这个继承于php自带的扩展，所以命名空间在\下
{
  public function __construct()
  {
   建立连接
	}
}

$redis = new \Lib\Redis();
$redis->get($key);
```

set,设置值

```
$redis->set($key, $value,)
```





### hash

hash类型不能像string的set方法中设置系统key的过期时间，可以通过调用expire来设置时间。

```
$redis->expire($key, 7*24*60*60);
```

hset: 设置hash的filed的值,这个key意思就是这个hash对象的唯一标识

```
$redis->hset($key, $filed, $value)
```

hmset: 给这个对象进行赋值, $arr是关联数组

```
$redis->hmset($key, $arr)
```

hgetall: 获取这个hash对象所有的内容

```
$redis->hgetall($key)
```



坑：

之前很傻，因为接触到redis的时候，听到的好处就是相比较于mecache这种，多了很多的数据类型，memcache这种只有string，然后redis又在原先5中数据类型的基础上增加了表示地理坐标的新的数据类型。当时遇到一个问题就是想一个键值既能对应string，也能对应hash，想有没有这种数据类型。其实当时的思路是完全跑偏的，完全可以通过先构造一个比如token=>string,string=>hash两个类型，第一个是string，第二个是hash，通过token既能得到简单的string，也能得到hash。