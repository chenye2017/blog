---
title: 关于nginx的那些事
date: 2018-08-20 11:42:26
tags: [nginx]
categories: 运维
---

lnmp 是php最常用的模式，工作中经常要自己搭建环境，nginx 的学习少不了，记录生活中经常用到的，方便查找

<!--more-->

常用命令

```
nginx -s reload   让配置文件生效，并重启
nginx -s stop   快速关闭
nginx -s quit  平滑关闭
nginx -t -c  nginx.conf  测试配置文件是否有语法错误
nginx -v  查看版本
nginx -V  查看编译信息，可能有时候要添加一些新的扩展
```



   http

​       server



复杂均衡  http层upstream (session 需要分布式，可以考虑用redis ,或者jwt token)

server 层 proxy_pass

轮训  权重都是1

加权轮训  weight

最少连接

ip hash

普通hash 



虚拟主机， 所有匹配的location 是同一级别，都需要配置root



nginx 处理跨域，其实等同于php 处理跨域，只是nginx 那个语法还不太熟悉

允许访问的域名

允许访问的方法

允许携带的参数

对于options 请求的处理



nginx 的调试

通过写日志  // 这个比较方便，return 的话很多内容不能记录

通过return 内容



curl 直接调试  -H 'host: xxx.com' 添加请求头

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20190718185551.png)



其实这个比较容易看出来的，host 在寻址找到服务器之后，域名已经不重要了，只是起到header 中一个变量的作用

默认第一个server 是匹配不到走的server (当然可以指定default server)







