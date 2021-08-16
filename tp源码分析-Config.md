---
title: tp源码分析-Config
date: 2019-08-09 09:47:22
tags: [PHP,ThinkPHP,Config]
categories: PHP
---

thinkphp config 源码阅读，为了更好的使用~

<!--more-->

首先 config  implements ArrayAccess, 所以我们可以这样

```
$z = new \think\Config();
$z->load(__DIR__.'test.ini','test');  // test 作为前缀，用来分割不同文件的配置项，如果不加，会放在一级目录，不太合适
// test 内容
[boy]
name=cy
sex=1

var_dump($z['test']['boy'][name]); // cy
```

分析Config的几个属性

```
config . 核心，配置文件内容加载进来的存放
prefix   默认前缀，set的时候，如果key 是个string , 默认前缀， 注意set 的内容如果是个数组，那就直接合并
path 初始化构造函数，配置文件的位置
ext  初始化构造函数，配置文件的类型
yaconf 布尔值或者字符串，用来支持yaconf
```

__construct 

 配置一些基本的参数，比如path 配置文件路径，ext 配置文件后缀，yaconf 判断是否安装了yaconf扩展

__make 

获取注入类app 中config文件路径和ext 文件后缀，初始化一个config类



setYaconf

注意这个一定要在属性yaconf true 的时候才能设置



getYaconf 

获取



setDeafultPrefix

 设置前缀，比如app



parse  

把一个配置文件（config driver 下面支持的类型，默认ini, json, xml）加载到 \$this->config 属性中



load 

加载文件(先判断该文件是否存在，不存在添加上文件夹前缀和文件后缀)解析放到config中，调用parse，并添加支持php,yaml.



getYaconfName  

Yconf::get(key), 获取的就是这个key



yaconf 

 获取yaconf 中内容，感觉比如直接调用yaconf 的get



loadfile

load = loadfile + yaconf



has 

判断是否有某个key , 如果没有添加前缀，会添加默认前缀. (默认所有的config 都得挂在某个 一级key 下面，比如app(默认)， 比如 log)



pull

获取所有的config 中一级配置，或者yaconf 中的某个属性 + config 中的一级配置



get

获取配置，key 中没有 . ,默认添加app 前缀，支持获取多层嵌套



set

只能支持两级key ,不包括文件名，如果传入数组，合并原先 config属性（只有在key 是string 的时候才会添加默认prefix ）



remove 

移除key



reset

重置某个key



剩下arrayAccess 需要实现的方法，方便数组方式调用





### 缺点

config 文件中支持 yaconf 和 \$this->config, 可有的方法是从二者中某个取值(一般都是yaconf优先)， 有的是yaconf 和 config 合并。

yaconf 的key中，第一个肯定是文件名

config key 中，如果没有. 默认都会拼上prefix , 想获取一个文件中所有内容，key 可以这样 app. , 感觉没有yaconf 直观，规范



### 关于yaconf(具体可以参考鸟哥博客)

yaconf 是鸟哥写的一个常驻内存的配置文件扩展，（windows下使用有些问题）

很简单，默认两个配置项，一个是dir (把该文件夹下的所有的ini文件都加载进内存，不支持文件夹是为了简单，不支持别的类型文件是为了简单),一个是check 更新时间(每隔比如2s检测一下文件是否改动, 只能用mv, 不能用cp, 可能是根据文件修改时间来进行判断是否改动)

api 也只有两个，

// test 是文件名，name是属性

get('test.name')  // 获取内容

has('test.name') //  判断是否有内容

我们来想想yaconf 解决的问题：我们在开发itbasic 的时候经常被这样的问题困扰，对于配置项，线上环境和本地环境不一样，所以导致我们每次提交代码都很麻烦，但凡用到配置参数的文件都会和线上文件冲突。后来我们把配置项都写到一个文件中，通过config 去获取，并且不把config文件纳入版本库，其实yaconf的本质也是这样，只是他分的更彻底，把config 完全移除项目代码，和项目代码分离，放到一个单独的文件夹中，可以让运维人员或者任何人员去修改。



### 关于配置文件格式

tp 中有ini, json, xml, yaml, php 文件

对于ini 文件（.env文件也是ini ）可以直接调用方法解析

json ,我们直接读取文件内容，然后json_decode 

xml ,我们直接调用方法解析

yaml ,我们需要安装yaml 扩展（centos7 中安装yaml 工具）

php, 我们直接include , 文件中的return 内容会被include 返回

