---
title: window下运行各种框架总结
date: 2017-11-25 17:53:43
tags: [linux,apache,框架,php]
categories: PHP
---

这篇文章主要是**对于php运行环境的搭建中运行起来框架的总结，其实运行框架，不管是yii，还是tp，还是laravel，还是我们那个项目框架，与其说是运行起来，不如**说是找到框架的入口文件**，然后剩下的就直接交给了框架进行处理（route.php的处理，restful api等等，通过解析模块，控制器，方法），绝大部分时候就这这么简单，至少上面三个都不用对apache**进行特殊的配置（目前的这篇文章只是单纯的针对Apache）。

<!--more-->

so，简化的思考了一下，就是找public下面的index.php 文件（tp  laravel都是这个文件 yii是在web目录下，其实找一下就行了，大部分都是reqire项目的核心库这种东西。例如yii

```
<?php

// comment out the following two lines when deployed to production
defined('YII_DEBUG') or define('YII_DEBUG', true);
defined('YII_ENV') or define('YII_ENV', 'dev');

require(__DIR__ . '/../vendor/autoload.php');
require(__DIR__ . '/../vendor/yiisoft/yii2/Yii.php');

$config = require(__DIR__ . '/../config/web.php');

(new yii\web\Application($config))->run();
```
），所以只要配置好了一个环境，剩下的框架的运行基本就一样了。
上面已经说了我们这片文章的目的了，运行起来框架，通过的是直接下载代码的这种方式（现在绝大部分框架都是支持composer来安装）。我们先安装好xampp，xampp会帮我们集成安装好apache和mysql，他们和在linux下是一样的，至少我到现在还没有发现什么不同，so，因为太过相似了，导致我从前以为xampp是在window环境下集成安装了lamp这种，其实这个l（linux是没有的），他的环境准确说是wamp（我代表 window这种架构）。他把php，mysql还有apache都放在了他自己也就是xampp的安装目录下，我们找到apache的安装目录，在conf文件夹里面找到（httpd-vhosts.conf）这个配置虚拟域名的文件，我记得以前apache配置虚拟域名是在一块的，都是写在httpd.conf文件里面，现在分开了，其实是包含了conf文件下的所有文件，但这样更清楚。
接下来我们就开始配置虚拟主机了（配置虚拟主机的意思就是在一个服务器上配置多域名访问的站点，其实不通过域名通过端口号啊什么的都可以，本质上就是把不同的请求分发到不同的目录下。我们的itbasic就是通过Nginx通过域名的不同把对80端口不同的域名请求分发到不同的端口下，然后通过apache虚拟主机的配置（通过的是端口的不同）来达到发送到不同目录下）。
添加
```
<VirtualHost *:800>
    ##ServerAdmin webmaster@dummy-host2.example.com
    ##DocumentRoot "D:/xampp/htdocs/dummy-host2.example.com"
    ##ServerName dummy-host2.example.com
    ##ErrorLog "logs/dummy-host2.example.com-error.log"
    ##CustomLog "logs/dummy-host2.example.com-access.log" common
     DocumentRoot "D:\code\laravel-v5.1.11\laravel-v5.1.11\public"  
    ServerName laravel.app  
        <Directory "D:\code\laravel-v5.1.11\laravel-v5.1.11\public">  
            Options Indexes FollowSymLinks Includes ExecCGI  
            AllowOverride All  
            Require all granted  
        </Directory>
</VirtualHost>
```
简单的记录下apache的语法 DocumentRoot 代表根目录
配置虚拟域名要写在virtualHost标签之间，后面的代表端口号，servername代表域名，directory里面allow代表接受任何的请求，之前安装larave的时候代开index.php嫌弃我权限不够，其实是这个directory标签的内容写错了，注意这个如果说打开文件夹没有权限，在window下你改这个目录的什么所有者权限啊什么都是没用的，我的理解是web请求的时候打开这个文件夹的用户不是我们能预测的，应该是other什么的，因为加了这句话，所以我们在window下安装这些框架不用修改日志文件，要么在linux下安装yii，laravel都要修改log日志文件夹的权限（感觉是因为写日志的操作是来自http的请求，像我们之前安装seaslog第三方的日志的时候，也得修改那个文件夹的权限，道理是一样的）
（yii框架的话conf文件夹下web.php cookiekey要随便填一个数，）

当然还要配置个虚拟域名，因为之前apache里面填写的域名如果是假的话，在本地host文件里面配置一下（因为浏览器解析是通过缓存（贼端）host文件，dns解析来找的，配置之后ping一下能不能同，就可以啦）

大功告成。

总结一下,本机运行起来一个框架。

1.安装集成环境（xampp）
2.修改apache配置文件，配置虚拟主机
3.修改host文件，修改域名到本机127.0.0.1

注意的点：
1.配置虚拟主机之后，xampp原先那个localhost直接就访问不了了，所以把那个localhost也配置一个虚拟主机（localhost也属于一个域名）
2.如果访问不了，先ping一下域名，看是否能ping通，然后查看apache的access日志和error日志，如果没有，确认http服务是否起起来了，如果有查看日志（linux下tail -f 动态看），window下在文件的最下面，像我现在这个window下环境，就是因为端口号被占了，改成800，所以每次访问域名后面都得加 ： 800，这个Apache的错误日志里面会记录的很清楚

附上三张框架图
1.![laravel](http://ozys8fka7.bkt.clouddn.com/TIM%E5%9B%BE%E7%89%8720171125172354.png)
2.![yii2](http://ozys8fka7.bkt.clouddn.com/TIM%E5%9B%BE%E7%89%8720171125172431.png)

(其实在用外链的时候想到了树洞外链，GitHub上一个关于php的开源项目)
对了突然想起来yaf框架，其实因为yaf框架是需要扩展模块支持，所以得先安装php扩展（php扩展大部分都是c开发的，），然后原理是一样的（只是yaf框架需要手动生成基本目录）

[地址](http://blog.csdn.net/underclound/article/details/76835318)

简单记录下window下安装php扩展（phpinfo能输出php的详细信息，在框架的index.php目录下是唯一所有人都能访问的）然后去pecl下载php扩展，下载dll文件的时候要注意那个什么tc之类的，注意好，下载好放在php的扩展目录下ext，再在php.ini 里面模仿之前的php扩展，加上相应的话就好了，那个扩展还是可以配置一些属性的
```
zend_extension = D:\xampp\php\ext\php_xdebug-2.5.4-7.0-vc14.dll
```
像mysql这种，都是可以添加配置信息的
```
[MSSQL]
mssql.allow_persistent=On
mssql.max_persistent=-1
mssql.max_links=-1
mssql.min_error_severity=10
mssql.min_message_severity=10
mssql.compatability_mode=Off
mssql.secure_connection=Off
```
yaf好像不配置，不能再生产环境下用，有个参数，其实安装seaslog的时候那些日志模板啊，日志格式啊都是在这里配置的，就是初始化的感觉。linux下的安装扩展要先下载文件，然后编译成so文件放在对应目录下，然后配置php.ini，有机会再说。window下只要注意好下载对应的dll文件就可以了





