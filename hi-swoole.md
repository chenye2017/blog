---
title: 'hi , swoole'
date: 2018-03-23 14:02:40
tags: [PHP,swoole] 
categories: PHP
---

学习目的：想让自己从增删改查，做web，表单的生活中调出来。

<!--more-->

1. swoole的安装

swoole作为PHP的一个扩展，用c语言开发，在使用他之前我们当然需要有php的基础环境。这里我们并不需要传统的lamp或者lnmp的架构，仅仅只要安装好php就行了。网上都说php最好编译安装，虽然我倒现在还不知道编译安装的好处，我看慕课上的那个教程也仅仅只是指定了php的安装文件夹。
首先要搭建我们的虚拟环境，强烈推荐vagrant，真的很方便，虽然他把镜像都放在了c盘，我也懒得改了，但其实自己平时练习的话一个镜像就够了，剩下的只要操作这个镜像就好了，我用的centos7。
安装好vagrant和vitrual box后去网上找镜像吧，[http://www.vagrantbox.es/](http://www.vagrantbox.es/),挺快的，详细说明也有，我是先用迅雷下载到本地，然后再添加的。
```
vagrant box add centos7   d:/centos7.box      //把这个box文件装到系统中，之后这个box文件可以直接删除
vagrant box list     //查看系统中是否有这个box文件的镜像
vagrant box remove centos7   //如果这个镜像没用了，可以直接移除，节约c盘空间
```
加入镜像后，就是去一个文件夹装载你的虚拟环境，他的底层是操作virtual box嘛，我的感觉就是和vitrual box那样找个文件夹存储这个虚拟环境。
```
d:
cd virtual                       //这是我平时虚拟环境统一放置的目录
mkdir centos7             //为了区别开，这是我这次试验放置的目录，centos7代表镜像的名称
cd centos7
vagrant init centos7  //初始化虚拟环境，后面这个是上面添加进系统的镜像名称（这时候文件夹会生成vagrant的虚拟环境）
修改vagrant file      //配置共享文件夹，开启ip地址
vagrant up   //启动虚拟机（会重启加载上面配置文件的内容）
```
几个命令很长时间使用哦
```
vagrant halt    //关闭虚拟机
vagrant suspend  //挂起虚拟机
vagrant resume  //唤醒虚拟机，对应上面的挂起，这样就不用每次启动了，就和linux永不关机都可以一样
vagrant destroy  //摧毁虚拟机，没用的即使删除，虽然感觉也不占多大地方
vagrant reload //这个我用的不多，主要是不记得他是否会重新加载配置文件了
```
上面的一些坑就是：
1. 把镜像纳入系统的时候，默认会放在c盘，virtual box 也有些文件放在c盘，导致c盘越来越小。
2. 加入镜像的时候，镜像的名称最好取得有点意义，比如ubuntu1604 这种，以后用起来的时候方便，时间长了，自己都忘记了。
3. 不要用ubuntu16.04这种，之前用过这种，虽然添加镜像的时候没有啥问题，但是后面启动虚拟机的时候会报错。
4. 挂载文件夹的问题。之前下载了ubuntu的box文件和centos56 的box文件，ubuntu能挂载上，centos挂载不上，网上说是vitrualbox的原因，可我都升级到2.0了都没法解决，仔细看了报错信息，说可能是这个box文件的问题。后来新下载了centos7的box文件，reslove。

```
vagrant ssh  //进入虚拟机，注意要在上面建立的装载虚拟环境的文件夹下使用
```
上面就是基础环境的搭建，可以理解成系统的搭建，剩下的就要我们去虚拟机中进行处理了。

首先是php的编译安装
[这篇文章里面写的很清楚](https://segmentfault.com/a/1190000004123048#articleHeader0)，源码是我去php.net上面下载的。然后去swoole.php上面下载swoole的源码。phpize,这是php源码编译后的一个命令，按照上面那篇博客应该是在/usr/loca/php7/bin 下面，这是因为swoole的源码包下面没有configure文件，产生后./configure,可能会报找不到php-configure,注意这里面不是php 的配置文件，而是php7那个文件夹bin命令下面有个php-configure，直接写这个地址就可以了。

遇到的坑。编译出错，直接执行make，显示找不到configure文件，··其实./configure那时候就报错了，我不知道，直接执行make，然后显示找不到目标，注意一下报错信息，其实这些编译的时候都没什么错误，或者把错误信息粘贴去百度google，很容出来。
注意要在php.ini 里面开启扩展 extension=swoole
因为我们这没有lamp环境嘛，不要想着在页面访问phpinfo,直接php -m,查看加载的扩展。



