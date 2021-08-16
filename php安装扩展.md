---
title: php安装扩展
date: 2018-01-18 18:35:02
tags: [PHP,Linux,扩展]
categories: PHP
---

好久没有写文章了，今天来说一下关于php的扩展，虽然自己理解的不是很深，也没有仔细去研究，只是把平时工作中遇到的情况，解决途径写下来，还有很多不足的地方，以后去补充
<!--more-->


php的扩展大部分都是c编写的，php -m 可以列出我们php中安装的扩展
![image](http://upload-images.jianshu.io/upload_images/5525740-a0582293f271520d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中swoole就是现在很火的php扩展。以这个为例子
Linux下安装
1. 获取tar.gz压缩包（可以去github上直接下载，或者去pecl（php 扩展库里面搜索））。
2. 解压缩  tar xvf 
3. 进入解压缩目录（如果进入解压缩目录内部文件是so结尾的，不是.c 之类的结尾的，不用进行后面的操作）
4. phpize  (这个命令在php的安装目录下，bin目录下，和php命令在一个目录下)
5. ./configure 生成配置文件，之前安装swoole的时候需要指定php-config,这可不是php的配置文件，也是一个命令，和上面的php，phpize在同一个bin目录下，如果想看更多参数可以 --help,
6. make
7. make install (安装失败make clean，重新再来呗)
   8.生成的so文件自动放到了/usr/local/php/lib/php/extensions/no-debug-zts-20160303，然后在php.ini 文件里面加上一句extension = swoole.so就可以了
8. 可能需要ldconfig下，http://man.linuxde.net/ldconfig

windows下安装
1. 大部分都是xampp的这种集成环境，首先去pecl上下载dll文件，··像swoole我是没找到windows下的扩展文件，需要根据php版本 vc版本 TS Windows版本（windows一般是x86 和64，操作系统位数） 来选择 扩展文件的版本
2. 解压后，把dll文件放到php的扩展目录下ext下面，然后在php.ini里面加上zend_exten```= 这个扩展文件名。就可以了

之前安装redis扩展的时候，···当时用的是xampp，··然后安装好了redis扩展，可是使用不了。坑爹啊，window下我都没安装redis，光安装redis扩展有什么用，····其实这个扩展知识为了让php能操作redis，像apache，在编译安装php7的时候，也要指定apache的apxs之类的位置。



坑：

1. 机器上安装了挺多php的，有5.6，有7，因为7编译安装的时候没有填写配置文件的位置，默认应该是放在/usr/local/lib目录下（php -r "phpinfo();"|grep configure, 注意一定要在phpinfo();后面加引号，查看编译安装参数，像apache，直接去安装目录下的build文件夹里面，查看就可以了），而我的机器上一直还有/etc/php/php.ini 这个配置文件，因为php安装的时候是不存在配置文件，讲道理编译的时候没有写配置文件的位置，应该放在/usr/local/lib目录下，但他还是放在了/etc/php/php.ini,聪明的apache自动去找到了这个配置文件，但当我在默认/usr/local/lib目录下放置了php.ini的时候，他是去这下面寻找的，以前的那个就废弃了。apache可以指定php配置文件，只需要在apache的配置文件里面指定下，地址：[http://www.jb51.net/article/106111.htm]
2. php -m 的好处就在于不用重启服务器，就能看到已经加载的php模块，php -r “phpinfo();” 执行后面的函数返回结果。 php -c 指定配置文件的位置，注意 php -m -c  加载扩展的时候默认使用的是默认配置文件的位置，php -c 文件位置 -m 这里才是新的配置文件位置方式加载扩展。



遗留的问题：

1. php -m显示出扩展swoole ， phpinfo 页面没有这个扩展