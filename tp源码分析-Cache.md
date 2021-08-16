---
title: tp源码分析-Cache
date: 2019-10-22 09:45:16
tags: [PHP, Cache]
categories: PHP
---

需要改变的观点是

1. 这个cache 并不能完全替代redis， 这个cache 只提供一些缓存比如redis,memcache，file 方法的聚合，比如set 设置缓存，如果配置的是redis， 走的是redis string的get 方法，如果配置的是file， 则对文件名进行hash， 然后存储在对应文件下面 （redis 键的过期不用我们代码中主动考虑，当我们使用redis 的set 命令的时候自动帮我们实现（redis有自己的键过期策略），但是file 没法设置过期时间，只有在get的时候拿着文件的mktime 加上 存储的expire time 和当前时间进行对比，如果超过了就返回null，并删除文件，当然我们在调用的时候不用考虑，单纯的通过 Cache::set() 就行了，上面的操作都是 cache的driver 之一 file 帮我们封装好了）

2. 介绍一下 cache 库的基本结构

   首先 \Cache -> facade Cache -> think Cache,  所以上述三个类都能调用cache，前面两个类等同，都只能通过 静态方式调用方法 ！！！，走的是我们的默认配置。think\Cache 属于实际类，他没有 _call_static 方法，所以我们不能用这个类静态的调用方法，需要先实例化，然后去调用。

   这个库包含三个基本文件

   1. cache.php ,这个类是入口，实例化的时候走的是他的_construct, 生成handel实际处理句柄，干啥的？ 对于set， get 操作，实际发起者就是他。然后就就是一些config 的配置，具体的操作并不涉及。

   2. cache 文件夹下的 driver.php .是一个abstract class （abstract 相比较interface ，首先abstract 是类，类只能单继承，interface 是接口，可以多实现。再者abstract 中可以包含 方法实体，但是interface 中的方法都不能包含实体。这个driver.php 应该用abstract，因为他包含一些公共方法，比如 pull ->取出缓存中的一个数据并删除，可以直接通过各个cache driver 的 get 和rm 方法的组合，并不需要每个  cache driver 各自定义，所以他的pull 方法可以直接包含方法体）。

<!--more-->

首先来看一下 think\Config

instance

容器类，装着一个个 options 的驱动类，这些驱动的名字默认是配置项（options）序列化，保证同样的配置不会生成多个实例



config 

配置属性，用来选择使用哪个驱动，file ? redis or memcache



handler

 注意这个和driver 下面的驱动类的handle 区分开，那个handle 是实现具体操作的句柄，比如 驱动是 redis， 那个handle 是 new \Redis （c 实现的php扩展）， 而Cache 中的handle 是驱动类， think\cache\Redis



__construct()

初始化方法，config 属性的赋值，然后调用init() 方法



connect()

根据配置生成实际的cache 文件夹的实体类，存储在instance 属性中 （可以有多个cache配置），名字用配置项的序列化生成的字符串。返回的内容是实体类



init()

返回内容也是实体类，但操作的是 handle，handle属于instance 中的一个，是当前调用Cache 时候某个config 生成的实体类，不多于一个。因为这个条件的限制，所以只有第二个参数force 在 true 的时候，或者当前handle 是null（比如file），才能实现 cache 加载类的切换。



getConfig()

获取options, 操作的属性名叫config （不叫options）



setConfig()

设置options,操作的属性名叫config



store()

上面也说了init 方法不太方便切换当前操作的cache， 这个store很方便，通过传入不同配置项的名称，实现cache 加载的config 的切换。

```
return [
    // 使用复合缓存类型
    'type'  =>  'complex',
    // 默认使用的缓存
    'default'   =>  [
        // 驱动方式
        'type'   => 'file',
        // 缓存保存目录
        'path'   => '../runtime/default',
    ],
    // 文件缓存
    'file'   =>  [
        // 驱动方式
        'type'   => 'file',
        // 设置不同的缓存保存目录
        'path'   => '../runtime/file/',
    ],  
    // redis缓存
    'redis'   =>  [
        // 驱动方式
        'type'   => 'redis',
        // 服务器地址
        'host'       => '127.0.0.1',
    ],     
]
复合cache， 通过传入redis， file ，还可以定义更多的
```



再看一下cache 文件夹下的Driver.php

handle

实体类操作句柄，类似 redis 的 new Redis, pdo 的 new Pdo



readTimes  

读取次数，自身用的比较少 （每次读取都会增加）



writeTimes

写入次数，用的比较少 （每次写入都会增加）



options



tag

给缓存加标签，比较方便的就是 clear 的时候会清除掉tag 绑定的缓存，而不是整个redis 库的flushDB



serialize 

 一个数组，其中的四个参数分别代表，序列化方法，反序列化方法，缓存前缀，缓存前缀的字符数量



abstract  has

是否有这个缓存



abstract get

读取这个缓存



abstract set 设置这个缓存



abstract inc 新增步长



abstract dec 减少步长



abstract rm 删除缓存



abstract clear  删除缓存



getExpireTime

这个方法太傻比了，不知道咋用，传入的参数是个时间，返回的还是时间，没啥用



getCacheKey 

正如名字上写的，获取缓存的名字，比如redis 的key值，file 通过这个返回的内容加上自身逻辑，生成对应文件名（其实就是options 中的prefix 拼接上 key）



pull

get 和 rm 的组合



remember

不存在则写入，类似redis 的分布式锁，5s 中内每隔 0.2 s 执行一次 看是否能设置成功。 5s 中后直接操作，不管有没有锁 （按理说锁的存在时间不应该这么久，所以直接覆盖了之前的锁）



tag

给当前类设置属性tag.

如果传入额外的key, 则把传入的key 存入该tag 名下，这个tag 和key的对应关系如下 ： tag 名根据定义关系生成key,  把对应的key 连接成字符串当做value， 做成 key -> value 映射 （整个cache 库的映射关系都是 key value 形式，没有别的数据结构）



setTagItem

给当前tag 加入新的 key



getTagItem

获取当前tag 对应的key

```
protected function getTagItem($tag)
    {
        $key   = 'tag_' . md5($tag);
        $value = $this->get($key);

        if ($value) {
            return array_filter(explode(',', $value)); 
        } else {
            return [];
        }
    }
 发现上面的array_filter 方法用的蛮多的，用来过滤 数组中的 false， 蛮好用的   
```



serialize 

获取的上面 serialize 属性参数1对应的序列化方法序列化



unserialize

获取的是上面serialize 属性参数2对应的序列化方法反学历恶化



registerSerialize

修改serialize 属性



handler()

获取句柄



getReadTimes

获取读取次数



getWriteTimes

获取写入次数



下面是cache 文件中的驱动，随便选一个，比如 redis.php 分析 (看了这个类很容易自己封装一个 redis 操作库)

（为啥还需要自己封装redis 操作库，因为这个cache 类只是简单的使用redis 的string 类型，更多的操作类型并没有）

（封装redis lib 其实蛮简单的，把这个redis 连接抄一下，然后加入一个_call 方法就好了，最简单的redis lib）

(其他的lib 比如es lib 也可以这么考虑，重要的就是一个tcp 连接)

options  

redis连接的参数



_construct

建立redis 连接 （支持 php 扩展 redis （phpredis） 和 predis (单纯用php 实现的redis 客户端)）



has

调用redis 的 exists 方法



get

调用redis 的get 方法



set

调用redis 的set 方法，其实 setex 可以直接用set 实现



inc

redis incrby 方法



dec

redis decrby 方法



rm 

redis delete 方法



clear

redis delete 或者 flushDB !!














