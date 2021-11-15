---
title: apache简单配置
date: 2018-04-16 21:38:54
tags: [apache,linux,框架]
categories: apache
---

关于apache的配置文件。不管是xampp安装的lamp环境，还是自己在linux下面搭建的lamp环境，之前关于apache的配置都是直接copy网上的，用的虚拟域名，本质是还是不够理解的，关于.htaccess 也只是刚开始学习tp的时候看到过，后来因为配置虚拟域名能解决问题，就没有过多的了解这个.htaccess了。

<!--more-->

首先说一下之前遇到的问题。某次，帮助交付的一个同事配置xampp环境，他当时对多站点的配置文件都写在了apache的主配置文件中，httpd.conf,当时因为已经了解了虚拟域名，所以我在同级目录extra下面的httpd-vhost.conf中帮他写了新的配置，当时用的是不同域名分发到不同文件夹下的方式，其实这在实际生活中也是这样的，因为如果靠的是端口号的话，因为web服务默认是80端口，如果用了别的端口号，比如http:\\\www.baidu.com:800, 这样访问百度的web服务，这对广大用户是不友好的，因为对于非程序员来说，根本不知道:800是什么。如果不想用域名开进行分发，端口号也是可以的，但为了不让用户去记忆端口号，可以通过nginx来帮忙。比如itbasic.datatom.com和bbs.datatom.com虽然绑定的是同一个服务器，但是通过nginx的转发，到了不同的端口，一个因为是从itbasic.datatom.com+ 80端口来的，所以转发到本服务器的8080端口，一个是从bbs.datatom.com+80端口来的，所以到了8081端口。我就是通过上述两种方式之一的域名不同来帮那个同事写的vhost配置的，可是呢，当我输入域名的时候，他虽然能访问，但是访问的还是之前他的一个网站，fuck！！！算了，配置文件写的这么乱，我就没找原因了，直接帮他copy了以前主配置文件里面写的一个站点配置。

其实后来我差不多知道原因了，大概是直接在主配置文件里面写了

```
ServerName localhost:80
DocumentRoot "F:/www"
<Directory "F:/www">
    #
    # Possible values for the Options directive are "None", "All",
    # or any combination of:
    #   Indexes Includes FollowSymLinks SymLinksifOwnerMatch ExecCGI MultiViews
    #
    # Note that "MultiViews" must be named *explicitly* --- "Options All"
    # doesn't give it to you.
    #
    # The Options directive is both complicated and important.  Please see
    # http://httpd.apache.org/docs/2.4/mod/core.html#options
    # for more information.
    #
    Options Indexes FollowSymLinks Includes ExecCGI

    #
    # AllowOverride controls what directives may be placed in .htaccess files.
    # It can be "All", "None", or any combination of the keywords:
    #   AllowOverride FileInfo AuthConfig Limit
    #
    AllowOverride All

    #
    # Controls who can get stuff from this server.
    #
    Require all granted
</Directory>
```

问题的关键就是那个server name，因为是localhost：80，但凡是host（windows下）里面127.0.0.1 用的80端口，统一转发到这个目录下，造成了上述结果。其实itbasic的服务器配置上也是这么写的，但之外的那些bbs，51dana用的却是虚拟域名，为啥那个能正常访问呢，在于配置文件上面定义了server name itbasic.datatom.com。其实itbasic那个也是可以通过域名来进行分发的，主要在每个vhost里面加个server name 就可以了。itbasic那个完全也可以不用端口进行转发，完全可以通过域名，在每个vhost里面加上server name就可以了

先说一下关于vhost的配置方法吧

打开apache的配置文件 htppd.cnf。分别打开重写扩展和虚拟主机扩展：

> LoadModule rewrite_module modules/mod_rewrite.so 这句前面的 注释 # 去掉
>
> Include conf/extra/httpd-vhosts.conf 这句前面的 注释 # 去掉

但现在一般apache都是默认开启的

```
<VirtualHost *:80>   //包裹的内容代表虚拟主机，端口号80
    DocumentRoot "D:/wamp/www/testphp/"  //项目的根目录
    ServerName php.iyangyi.com   //域名
    ServerAlias www.pptv.cn #可省略  //这个是我们的虚拟域名的别名，可以不要，他的出现场景就是我们希望另外一个域名也往这个目录下调整。比如 www.pptv.cn 我们也希望跳到这里来，就可以这样做，但是前提是 www.pptv.cn 也要绑定host 127.0.0.1
    ServerAdmin stefan321@qq.com #可省略  //这里填 服务器管理员的邮箱，也可以不要，当服务器出现故障后，如果提前有配置邮箱的话，会往这个邮箱发邮件，或者是显示在网页的错误信息当中。一般我们可以不填。
    ErrorLog logs/dev-error.log #可省略  //当访问出现错误的时候，就会记录到这里，注意：logs/dev-error.log 这个文件路径是apache的安装目录下的logs 目录 。可以不要。
    CustomLog logs/dev-access.log common #可省略 //这里填 访问日志，用来记录每一次的请求访问，可以不要。注意：logs/dev-access.log 这个文件路径是apache的安装目录下的logs 目录 。记住：路径后面加common。
    ErrorDocument 404 logs/404.html #可省略 //这里填 403,404等错误信息调整页面，用来访问出现404页面等情况时的错误页面展示，比较有用，也可以不要。注意：/404.html 这个文件路径是项目的根目录，不是apache的目录。
    <Directory "D:/wamp/www/testphp/">
        Options Indexes FollowSymLinks
        AllowOverride All
        Order Allow,Deny
        Allow from all
        RewriteEngine on
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteRule ^(.*)$ index.php/$1 [QSA,PT,L]
    </Directory>
</VirtualHost>
```

一般配置的时候，上面写了可省略的都不会配置。

```
<Directory "D:/wamp/www/testphp/">
```

是最重要的一步了，这里也是填本项目的路径，然后所有的`rewrite`规则都是在里面完成。所以这个是很重要的。

```
Options Indexes FollowSymLinks
```

作用我访问php.iyangyi.com，如果文件根目录里有 index.html(index.php)，浏览器就会显示 index.html的内容，如果没有 index.html，浏览器就会显示这文件根目录的目录列表，目录列表包括文件根目录下的文件和子目录。

到底是优先显示index.php还是index.html 有apache的配置决定的：

```
<IfModule dir_module>
    DirectoryIndex index.html index.htm index.php index.php3  
</IfModule>
```

这个是单独写出来的，而不是写在上述标签之内。这个感觉应该是统一定义一个就行了，或者是在vhost里面直接写directoryindx 这个标签，如果没写默认是index.html或者是index.php

之前遇到一个问题就是public文件夹因为是统一对外开放的嘛，不做处理的话，比如itbasic.datatom.com/userpic,可以访问服务器上public下面这个userpic文件夹，为了不让用户访问，我们可以去掉

```
Options Indexes FollowSymLinks
```

的indexes,用户就不能访问了。

`Order Deny,Allow Allow from all`这2个一般是组合在一起用。用来`设置访问权限` ，设置哪些ip可以访问这个域名, 哪些ip禁止访问。

所以order是设置这2个的组合排序, 不区分大小写，中间用`,`分开，中间不能有空格。 

所以order是设置这2个的组合排序, 不区分大小写，中间用`,`分开，中间不能有空格。 
`Order Deny,Allow` ：表示设定“先检查禁止设定，没有设定禁止的全部允许”

`Order Allow,Deny` : 表示设定“先检查允许设定，没有设定允许的全部禁止”

**而且最后的访问结果有第二参数决定！**

`Deny from All` `Deny from 127.0.0.1` 禁止访问的ip， all 表示全部 

`Deny from All` `Deny from 127.0.0.1` 禁止访问的ip， all 表示全部 
`Allow from All` `Allow from 127.0.0.1` 允许访问的ip， all 表示全部

我们看几个他们2个组合的例子。

这个例子：

```
Order Deny,Allow
Deny from All

```

表示先检查允许的, 没有允许的全部禁止。但是下却没有Allow，那么就表示是无条件禁止了所有的访问了。

```
Order Deny,Allow
Deny from all
Allow from 127.0.0.1

```

上面表示 只允许127.0.0.1访问

```
Order Allow,Deny
Allow from all
Deny from 127.0.0.1 192.168.1.51

```

上面表示禁止127.0.0.1和192.168.1.51访问，其他都可以！

所以这个的组合就可以达到很多的过滤访问效果。

但现在一般不用那个了

```
Require all granted
```

才能解决，要么一直都是403。

## RewriteCond 与 RewriteRule 指令格式配置详解

上面花了大量的时间讲述`VirtualHost` 里面的一些配置参数的写法和作用，接下来就是rewrite的重点了，3个核心的东西：**RewriteEngine，RewriteCond，RewriteRule**

**RewriteEngine** 

**RewriteEngine** 
这个是rewrite的`总开关`，用来开启是否启动url rewrite，要想打开，像这样就可以了：

> RewriteEngine on

**RewriteCond 和 RewriteRule** 

**RewriteCond 和 RewriteRule** 
表示指令定义和匹配一个规则条件，让RewriteRule来重写。说的简单点，RewriteCond就像我们程序中的if语句一样，表示如果符合某个或某几个条件则执行RewriteCond下面紧邻的RewriteRule语句，这就是RewriteCond最原始、基础的功能。

先看个例子：

```
RewriteEngine on
RewriteCond  %{HTTP_USER_AGENT}  ^Mozilla//5/.0.*
RewriteRule  index.php            index.m.php
```

上面的匹配规则就是：如果匹配到http请求中HTTP_USER_AGENT 是 Mozilla//5/.0.* 开头的，也就是用FireFox浏览器访问index.php这个文件的时候，会自动让你访问到index.m.php这个文件。

**RewriteCond 和 RewriteRule 是上下对应的关系。可以有1个或者好几个RewriteCond来匹配一个RewriteRule**

RewriteCond一般是这样使用的

```
RewriteCond %{XXXXXXX} + 正则匹配条件

```

那么RewriteCond可以匹配什么样的数据请求呢？ 

那么RewriteCond可以匹配什么样的数据请求呢？ 
它的使用方式是：`RewriteCond %{NAME_OF_VARIABLE} REGX` FLAG

```
RewriteCond %{HTTP_REFERER} (www.test.cn)
RewriteCond %{HTTP_USER_AGENT}  ^Mozilla//5/.0.*
RewriteCond %{REQUEST_FILENAME} !-f
```

上面是常见的3种最常见使用最多的`HTTP头连接与请求`匹配。

**HTTP_REFERER** 

**HTTP_REFERER** 
这个匹配访问者的地址，php中$_REQUREST中也有这个，当我们需要判断或者限制访问的来源的时候，就可以用它。

比如：

```
RewriteCond %{HTTP_REFERER} (www.test.cn)
RewriteRule (.*)$ test.php

```

上面语句的作用是如果你访问的上一个页面的主机地址是www.test.cn，则无论你当前访问的是哪个页面，都会跳转到对test.php的访问。

再比如，也可以利用 HTTP_REFERER `防倒链`，就是限制别人网站使用我网站的图片。

```
RewriteCond %{HTTP_REFERER} !^$ [NC]
RewriteCond %{HTTP_REFERER} !www.iyangyi.com [NC]
RewriteRule \.(jpg|gif) http://image.baidu.com/ [R,NC,L]
```

NC nocase的意思，忽略大小写。第一句呢，是必须要有域名，第一句就是看域名如果不是 www.iyangyi.com 的，当访问.jpg或者.gif文件时候，就都会自动跳转到 [http://image.baidu.com/](http://image.baidu.com/) 上，很好的达到了防盗链的要求。

**REQUEST_FILENAME** 

**REQUEST_FILENAME** 
这个基本是用的最多的，以为url重写是用的最多的，它是匹配当前访问的域名文件，那哪一块属于REQUEST_FILENAME 呢？是url 除了host域名外的。

```
http://www.rainleaves.com/html/1569.html?replytocom=265
```

这个url，那么 REQUEST_FILENAME 就是 `html/1569.html?replytocom=265`

看个例子：

```
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^room/video/(\d+)\.html web/index\.php?c=room&a=video&r=$1 [QSA,NC,L]
```

`-d` 是否是一个目录. 判断TestString是否不是一个目录可以这样: `!-d` 

`-d` 是否是一个目录. 判断TestString是否不是一个目录可以这样: `!-d` 
`-f` 是否是一个文件. 判断TestString是否不是一个文件可以这样: `!-f`

这两句语句`RewriteCond`的意思是请求的文件或路径是不存在的，如果文件或路径存在将返回已经存在的文件或路径。一般是这样结合在一起用的。

上面`RewriteRule`正则的意思是以 room开头的 room/video/123.html 这样子，变成 web/index.php?c=room&a=video&r=123

`$1` 表示匹配到的第一个参数。

## RewriteRule 写法和规则

RewriteRule是配合RewriteCond一起使用，可以说，RewriteRule是RewriteCond成功匹配后的执行结果，所以，它是很重要的。

来看一下 `RewriteRule`的写法：

```
RewriteRule Pattern Substitution [flags]

```

`Pattern`是一个正则匹配。`Substitution`是匹配的替换 `[flags]`是一些参数限制；

我们看几个例子：

```
RewriteRule ^room/video/(\d+)\.html web/index\.php?c=room&a=video&r=$1 [QSA,NC,L]
```

意思是 以 room开头的 room/video/123.html 这样子，变成 web/index.php?c=room&a=video&r=123

```
RewriteRule \.(jpg|gif) http://image.baidu.com/ [R,NC,L]
```

意思是以为是访问.jpg或者gif的文件，都会调整到 [http://image.baidu.com](http://image.baidu.com/)

所以，掌握正则级是关键所在了。以后，我会专门搞一个正则的篇章来学习下。

我们再看看`[flags]`是什么意思？

因为它太多了。我就挑几个最常用的来说说吧。

`[QSA]` qsappend(追加查询字符串)的意思，次标记强制重写引擎在已有的替换字符串中追加一个查询字符串，而不是简单的替换。如果需要通过重写规则在请求串中增加信息，就可以使用这个标记。上面那个`room`的例子，就必须用它。

`NC` nocase(忽略大小写)的意思，它使Pattern忽略大小写，也就是在Pattern与当前URL匹配时，"A-Z"和"a-z"没有区别。这个一般也会加上，因为我们的url本身就不区分大小写的。

`R` redirect(强制重定向)的意思，适合匹配Patter后，Substitution是一个http地址url的情况，就调整出去了。上面那个调整到image.baidu.com的例子，就必须也用它。

`L` last(结尾规则)的意思，就是已经匹配到了，就立即停止，不再匹配下面的Rule了，类似于编程语言中的`break`语法，跳出去了。

其他的一些具体的语法，可以参考以下资料：

[http://www.skygq.com/2011/02/21/apache%E4%B8%ADrewritecond%E8%A7%84%E5%88%99%E5%8F%82%E6%95%B0%E4%BB%8B%E7%BB%8D%E8%BD%AC/](http://www.skygq.com/2011/02/21/apache%E4%B8%ADrewritecond%E8%A7%84%E5%88%99%E5%8F%82%E6%95%B0%E4%BB%8B%E7%BB%8D%E8%BD%AC/)

[http://www.2cto.com/os/201201/116040.html](http://www.2cto.com/os/201201/116040.html)

[http://www.cnblogs.com/adforce/archive/2012/11/23/2784664.html](http://www.cnblogs.com/adforce/archive/2012/11/23/2784664.html)

## .htaccess文件的使用

`.htaccess`文件是啥呢？我们前面说了这么多的配置和修改，都是针对于apache的配置文件来修改的。.htaccess文件就是它的一个替代品。为啥呢？因为你每次修改apache的配置文件，都必须重启apache服务器，很麻烦不说，有些共享apache的服务器，你还没权限修改和重启apache。所以，.htaccess文件就应运而生了。（不用重启apache,是真的挺方便的）

`.htaccess`分布式配置文件。它文件名字比较奇怪，没有文件名，只有一个文件后缀就是`.htaccess`。所以一般在windows下还没法新建这个文件，因为windows不允许文件名是空的，比较蛋疼。但是我相信你总归会有办法新建这个文件的。（在linux下新建一个，下载到windows中呗）

`.htaccess`同时是一个针对目录的配置，你可以把它放到项目的根目录下，那么它就多整个项目其效果，如果你把它放到一个单独的子目录下，那么它就对这个子目录其效果了。

**.htaccess**文件如何生效呢。上面讲配置的时候，我讲过了`AllowOverride All`这个配置，它就是启动`.htaccess`文件是否可以使用的。`AllowOverrideAll`表示可以。`AllowOverride None`表示禁止使用。还是蛮简单的。

**那.htaccess文件里的语法是怎么写额呢？**

其实和上面说的一模一样的写法。可以完全的搬过来用。没问题。

```
 <Directory "D:/wamp/www/testphp/">
    Options Indexes FollowSymLinks
    AllowOverride All
    Order Allow,Deny
    Allow from all
    RewriteEngine on
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^(.*)$ index.php/$1 [QSA,PT,L]
</Directory>

```

上面的apache的``里的这一块就可以完全的搬到`.htaccess`文件中来，且效果一模一样。



上面就是vhost文件里面的配置的大致写法，其实httpd.conf里面也是这么写的，毕竟vhost最后还是包含到主配置文件里面。

documentroot 其实自己一直好奇为啥和directory参数一样其实本质上该项目文件夹下只有这个document root能访问，所以重写规则也好，针对的都是这个document root，导致了二者之间的相同。



有一点要注意的就是，如果通过开不同的端口号转发到不同的文件夹，需要早配置文件的开头写上Listen 127.0.0.1：8080 这样，开启端口号。







































































