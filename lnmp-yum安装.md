---
title: lnmp 环境搭建
date: 2018-01-19 15:21:24
tags: [Linux,lnmp,环境]
categories: Linux
---

关于lnmp环境的yum安装。作为一个phper，肯定得熟悉。之前写过关于在windows下php环境的搭建··咳咳咳，其实那根本算是搭建，像xampp都是把环境搭配好了，自己只是做了配置虚拟主机，然后运行框架。
框架在我的理解中本身除了yaf之类的安装，主流的框架tp，yii，laravel更像是一个文件夹，代码的集合体，这个文件夹中各个子文件夹联系紧密，成为一个整体。当外来请求通过lamp进入到入口文件，路由进入到控制器，模型层，数据库，变成一个有用的数据返回出来，再通过lamp返回给浏览器，上面的流程就很容易发现，简单的像apache,nginx里面的配置文件定义，只是让外来请求找到这个入口文件，进入到入口文件里面之后就靠框架的路由寻找具体的控制器了。
lamp环境方便的让外来请求比如对于index.php的请求，如果是直接在linux中，我么需要php (编译器) 可能还需要 -c指定配置文件的位置 文件.php,这样。但是如果是lamp，我们可以通过浏览器直接访问这个php文件。
<!--more-->

首先是yum源的安装，更新。

```
[root@localhost ~]# rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
[root@localhost ~]# rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
[root@localhost ~]# rpm -Uvh  http://dev.mysql.com/get/mysql57-community-release-el7-9.noarch.rpm
```
先说这个mysql yum安装，现在要通过这个社区版。

然后剩下两个是为了安装第三方软件用的。这个epel以后可能经常用到，虽然没lamp好像只要这个webtatic就可以了。之前添加源的时候看过一篇文章，好像是说最好不要和官方源里面有内容冲突。这个epel很合适，大家都在用，当然自己也就用咯。



yum安装nginx，mysql，php

```
[root@localhost ~]# yum -y install nginx
[root@localhost ~]# yum -y install mysql-community-server
[root@localhost ~]# yum -y install php70w-devel php70w.x86_64 php70w-cli.x86_64 php70w-common.x86_64 php70w-gd.x86_64 php70w-ldap.x86_64 php70w-mbstring.x86_64 php70w-mcrypt.x86_64  php70w-pdo.x86_64   php70w-mysqlnd  php70w-fpm php70w-opcache php70w-pecl-redis php70w-pecl-mongo
```
其实这里面比如php7安装的那几个我也不懂，据说把70w改成71w就能装7.1版本了。



yum安装的一个好处是通过systemctl enable 能控制软件的开机启动 关闭，通过systemctl status 能看软件的状态，启动，关闭。默认安装的mysql没有密码，可以直接登录进去。据说要配置默认编码，utf8，但我好像没有配置，它自动就是utf8。

```
[root@localhost ~]# vim /etc/my.cnf
[mysqld]
character_set_server=utf8
init_connect='SET NAMES utf8'
[root@localhost ~]# systemctl restart mysqld    # 重启 MySQL
```


nginx的配置

```


location / {
    #定义首页索引文件的名称
    index index.php index.html index.htm;   
}

# PHP 脚本请求全部转发到 FastCGI处理. 使用FastCGI默认配置.
location ~ .php$ {
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_index index.php;
    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    include fastcgi_params;
}
```
注意这块document root的位置，要卸载location外面，我之前写在上面那个location里面，下面那个location获取不到。

重启nginx，重启php-fpm.

测试是否成功。
在nginx根目录下 <?php echo phpinfo();die;

看能不能输出php信息咯。其实还可以检测下和mysql的连接。

 

