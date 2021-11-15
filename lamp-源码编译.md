---
title: lamp 和 lnmp 源码编译
date: 2018-01-19 15:21:41
tags: [Linux,lamp,环境]
categories: Linux
---

lamp的源码编译安装。！！！注意哦，这里面是lamp，不是lnmp。之前的lamp中，apache和php的交流主要是apache把php作为自身的一个模块，而ngixn是通过cgi，php-fpm进行管理进程（这段话有待考证，因为apache也能使用php-fpm进行管理，并不是php-fpm和apache绑定的。我们那个lamp环境搭建的时候安装好apache,编译php 的时候要指定apxs的位置，--with-apxs2=/app/httpd24/bin/apxs ,这之间的交互过程还不是特别清楚）。

所以lamp的编译安装是有顺序的，因为php在编译的过程中需要指定apache的apxs模块位置。
<!--more-->

### apahce 的编译安装

apache2.4之后编译安装之前需要apr和apr-util的支持（这两个文件下载位置，百度，好像在apr.apache.org下面，首页上面只是展示最新的，注意有个archive.download.sit,点击他，你会发现新世界的大门）apache在httpd.apache.org下面下载（讲道理我不知道他们之间的搭配有没有版本限制，so 下载的时候都下载最新的？比较靠谱的办法就是网上再找一份教程上面有apr apr-util apache的版本，对照着做就好啦··我的文章只是自己的思路呢）。

```
[root@localhost src]#pwd
/usr/local/src
[root@localhost src]#tar zxvf apr-1.6.2.tar.gz 
[root@localhost src]#tar zxvf apr-util-1.6.0.tar.gz 
[root@localhost src]#tar zxvf httpd-2.4.27.tar.bz2
```
 安装apr-util之前得先安装apr，因为需要配置 （--prefix==/usr/local/apr），上面的三个编译安装或者	说所有linux的软件编译安装都是一个套路 :
   1.进入解压目录，./configure 配置参数，百度吧 
   2.make
   3.make install
   4.查看

需要注意的就是可能安装的时候要安装额外的包，其实刚开始没安装也没什么大事情，后面编译肯定通过不了，所以就百度，一下子就能出来答案。

还有就是 configure 文件不存在，怎么办？一般目前我遇到这种情况主要是在编译安装php扩展的时候，可以通过 phpize(这个命令在php安装生成的bin目录下的一个指令),运行后会生成configure脚本，然后再像上面一样执行

参考文章：[centos7 源码编译httpd7](http://blog.csdn.net/leshami/article/details/50144179)
其实这一块都不用lamp中安装apache和单独安转apache没啥区别。

notice :

- （而且后来发现一个问题 apachectl start 这些用不了，应该是按照上篇博客中安装出现的问题，其实还是要解决下，因为在lamp中，刚开始修改php.ini 是很常见的，每次修改都需要apachectl restart 一下还是很方便的)

  httpd -k start 这戏好用

  ​其实ps -ef|grep httpd ,也能看到这个httpd 的命令



- 把apachectl 添加到环境变量中，我一般通过的方法是

```
 vim /etc/profile 
 HTTP=/usr/local/apache/bin
 $PATH=$HTTP:$PATH
 export $PATH
 需要注意的是我们在使用变量的时候会从前往后找的，比如http 和 path 中都有 apachectl,但是http 在最前面，会使用前面的http中的apachectl, 通过 which apachectl可以查看这条命令来自哪里（yum安装的软件可能把命令自动加到/usr/bin 里面去，卸载的时候也删除不掉，这个很坑爹，需要注意下，经常用的时候不是你想用那条命令，所以一直加在前面可能会避免一些这样的事情发生）
 source /etc/profile 让他生效
 echo $PATH 看是否包含了你需要的路径
```


- 我在正常启动apache 之后，外网总是访问不了，通过查看配置文件的日志 access_log 是空的，error_log中有内容，但其实是notice 级别的错误，一直以为是这个原因导致外网不能访问，百度了好久这个错误信息，都没有找到解决办法，问同事，演变成apache正常启动，外网访问不了的问题，先通过curl 127.0.0.1:80 来尝试访问，得到内容， it works 说明服务器没有问题，想到防火墙，看了下服务没有开启，想了下selinux 也没有开启，最后是阿里云的锅，zzz,其实问题很简单，解决过程也很简单，只是自己很笨，不知道怎么去排查问题


- 虚拟主机的配置，测试外网是否能访问，需要修改配置文件，需要注意的点
  1. 开启httpd-vhost,需要包含，然后再那个配置文件中写入
  2. 开启apache rewrite模块
  3. 开启端口监听，Listen 0.0.0.0:8080，注意0.0.0.0
  4. 默认查找Index.php，除了在虚拟主机的virtual host 中配置，在外层的配置文件中也要配置
  5. 开启php文件处理，主配置文件中修改， 这时候在apache的根目录下访问phpinfo这种文件还是不幸的，需要支持对php的mime类型解析，（想了下框架不能处理json输入是因为框架原因，还是web server原因）
  ```
  AddType application/x-httpd-php .php
  ```


- 默认apache 运行用户是daemon,当我们在php 中运行写日志的比如seaslog,默认写入是/var/log,daemon用户没有写入这个文件夹的权限，我们需要修改文件夹权限或者新建新的文件夹或者修改运行用户等

### nginx 编译安装
参考网上

nginx -s reload  重启

nginx -t -c conf 检查配置文件的语法，并不会执行

### mysql 编译安装

失败，用的yum安装（直接下载编译好的二进制码比较好，比如subline下载之后直接就能使用，不用安装，这些java写的东西啊，好用，但是mysql不行，即使是编译过的二进制码，还要执行一些脚本）



### postgresql 的编译安装（网上的教程很详细）

这个下载速度很快，可以考虑源码安装

1. 有两个配置文件，一个用来进行用户权限的限制，一个好像是具体的配置
2. 有两个目录，一个专门装数据，一个装软件，比如命令这些
3. 默认root 访问不了，创建postgres用户和组（源码编译，yum的话自动创建），修改配置文件trust
4. pg_config 这个命令很重要，安装中文搜索引擎，先安装scw库文件，再去安装zhaper这个分词，多个pgsql 一定要指定pg_config（好像指定不起作用,一定要把pg_config 直接运行的这个命令对应的pgsql 对上你想要装扩展的pgsql,要不然在数据库里面创建扩展会不成功 create extension ）
5. 可以通过sql语句查询配置文件的位置（locate 不知道为啥找不到/mnt 下的文件，坑死我了）



### php编译安装

1. php.net 上下载源文件
2. 安装额外的包
```
yum install libxml2-devel bzip2-devel libmcrypt-devel
```
3.编译安装php7
```
[root@localhost php-7.1.10]#./configure \
> --prefix=/app/php \
> --enable-mysqlnd \
> --with-mysqli=mysqlnd \
> --with-openssl \
> --with-pdo-mysql=mysqlnd \
> --enable-mbstring \
> --with-freetype-dir \
> --with-jpeg-dir \
> --with-png-dir \
> --with-zlib \
> --with-libxml-dir=/usr \
> --enable-xml \
> --enable-sockets \
> --with-apxs2=/app/httpd24/bin/apxs \
> --with-mcrypt \
> --with-config-file-path=/etc \
> --with-config-file-scan-dir=/etc/php.d \
> --enable-maintainer-zts \
> --disable-fileinfo


# 进行编译安装
[root@localhost php-7.1.10]#make -j 2 && make install 
```
注意那个with-apxs2 的选项
下面那个 make -j 2 指的是用两个cpu进行编译

注意这时候时没有配置文件的，到解压的php文件夹下
```
[root@localhost php-7.1.10]#cp php.ini-production /etc/php.ini
```
notice

1. 编译php7的时候with-config-file-path 指定了配置文件的位置,这个是很重要的，因为要不然我们没有指定也没有放在默认的php文件存放位置，以后每次运行php 的时候都要指定 配置配置文件的位置很麻烦

2. 测试phpinfo， 看到了local value 还有另一行没，之前看过好像一个是apache 中指定的位置，但是本地可以通过脚本修改，所以local value 是实际的位置

3. 讲道理，上面的编译参数运行不了， --with-config-file-scan-dir=/etc/php.d \，感觉是这个参数在作怪，不太明白这个文件夹中装的啥，反正是开启了一些curl 等的扩展，但实际上好像没有开启，所以需要把屏蔽掉（上面的php 编译参数不太好用，像curl 都没有开启，在项目中用会出错）

4. php 源码包的作用。像pgsql 这些扩展，源码包中都是有的，如果想开启，不用再网上下载，直接去源码包的ext 下面找就好了，然后phpize ```编译，修改php.ini, php -m 查看是否成功。需要注意的是，一个包编译失败之后，再次编译，最好make clean一下。（pgsql 和 pdo_pgsql 是两个扩展···）(知道pgsql 和 pdo_pgsql 的关系嘛， pgsql 只是一个pgsql 的驱动，pdo_pgsql 只有在他之上，才能和php 进行交互，同理联想mysql, php的mysql引擎是 mysqlnd, pdo_mysql 是在这个引擎之上让php和mysql 进行交互的，感觉我要是用了mysqli扩展是不是不用安装mysqlnd扩展了)

   ![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20190711170145.png)

5. 关于错误信息的输出。php.ini 有不同的两个版本，development, product， 虽然我没有研究过他们的不同，但目前发现最基本的就是，开发版本会把错误信息输出在页面上，但是生产版本不会，是php.ini 中一个选项的原因，生产版本可以把错误信息输出到日志中保存，好像nginx fpm 那种比较复杂，lamp这种直接在apache 的错误日志中就可以找到。（多熟悉lnmp,最好完全切过来，熟悉swoole）

6. 关于php-fpm的配置。这里拿lnmp 来举例，nginx 接受外来请求，转发到php-fpm (php进程管理器，默认监听端口9000) master 进程上，master 进程分别把请求转发给work进程，woker 进程实现了fast-cgi协议 (默认启动几个woker进程，接受请求，请求接受完不断开链接，接着等待，但是可以设置最大请求数量，当达到最大请求数量的时候进程会重启，防止内存泄漏，感觉类似swoole 的进程模型，只是这里面每次还是会重新加载php脚本，所以每次我们修改代码不需要重启)，内嵌php解释器，需要注意的点就是在上面架构中nginx 只是起到转发的作用，我们需要把一些必要的参数携带上转发给我们的php-fpm，这样我们的php才能正常接收到外来请求（php-fpm和 nginx 通信可以通过sock ，常见 tcp 127.0.0.1:9000, 或者unix socket文件，但是unix文件不能让部署在另一台机器上的php-fpm和这台机器上的nginx通信）

   ```
   		location ~ \.php {
    36            root  /home/itbasic/new_datatalk/api/;
    37            fastcgi_pass   127.0.0.1:9000; // tcp连接
    38            fastcgi_index  index.php; 
    39            access_log logs/access_datatalk_api.log  test;
    40            error_log logs/error_datatalk_api.log  error;
    41            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name; 
    # 这个script_filename 将会被传递给php-fpm,php-fpm 也会传递给php
    # /index.php/api/leave/count, php 能识别这样的连接，并且识别index.php 是脚本,/api/leave/count 是uri
    42            include        fastcgi_params;
    43 
    44         }
    45 
    46 		# 因为前后端分离，后端是在/api下面
    			# 调试nginx 的几个方法，return '200 ok'; 注意末尾都要有分号，但是这个return 好像只能返回状态码，只能用在nginx 的正则中进行测试，看是否走到这个 location 中，如果想查看nginx 中变量的值的话，可以通过log_fmat  test "$request_filename"  access_log log/access.log test ,这样记录到日志中，在日志中查看
    			# 对于虚拟站点的请求最好都写上access_Log 和 error_log ,监控请求是否按照我们的想法走
    47         location /api/ {
    				root /home/itbasic/new_datatalk/api/;  # 每个匹配都要加上root, 因为各个匹配是平级的关系，后面判断request_filename 的时候是会拼接上这个root的
    48            access_log logs/access_datatalk_api.log main;
    48            access_log logs/access_datatalk_api.log main;
    49            root /home/itbasic/new_datatalk/api/;
    50            if (!-e $request_filename) {
    51               rewrite ^(.*)$ /index.php$1 last; # 代表在请求前面加上/index.php,last在location 里面代表直接跳出这个location，重走所有的location
    52               break;  # 感觉可有可无，这个地方是直接终止生命
    53            }
    54 
    55 
    56         }
   ```


上面我们配置了nginx 的accesslog 和 errorlog,接下来我们需要配置php-fpm 的access_log  error log 和 slow_log

access_log slow_log 在 php-fpm.d 下面的default  配置文件中，对于slow_log ,我们还要设置多长时间才记录，如果是0的话代表不开启

error_log 在 php-fpm.conf 中，我是这么理解的，上面两个代表了某个站点的，所以需要单独配置，这个php-fpm 代表整个php-fpm 软件的，所以需要统一配置，他记录的是php-fpm这个软件的错误

另外因为默认php-fpm 关闭了work进程的错误输出，我们是获取不到php的错误内容的，我们需要开启catch_work_error 这个配置，然后修改php.ini ,error_log 位置，error_reporting 级别，这样php的错误日志就能在 php.ini 中配置的地址出记录（！！注意不是php-fpm 的error日志），如果没有写权限，默认会输出到nginx 的错误日志上（注意我们的日志文件一定要有w权限，之前一直用的755 没有w权限）

如果能写上，那就不会输出到nginx 的错误日志上



我们修改了php的配置文件一定要重启php-fpm(nginx 没必要重启)

kill -USR2 pid (swoole 的重启USR1 ,并且不会修改pid, 当我们修改一些swoole的配置文件或者php自身的时候，一定要kill -9 再启动，否则上面的重启没有用)

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20190711174208.png)

上面是慢查询日志，主要看那个sleep ，剩下的函数都是因为那个sleep导致的



php 错误级别

```
error_reporting=E_ALL &  ~E_NOTICE  // ~代表出资之外，不包含的意思
```



今天在使用composer安装文件的时候提示php版本太低，其实composer 本质上可以打开看一下是一个php的脚本，类似shell脚本，在脚本开始的地方定义了解释器，所以我们直接 composer 就能运行这个脚本，类似我们平时运行 shell 脚本一样，但是那个解释器是在我们没有指定解释器的情况下默认执行的，当我们制定了解释器，那个默认解释器就不起作用了，所以这时候我们只需要 php7.1 composer install xxx 就好了，但是如果composer不兼容以前的话，我们还是得下载最新的composer喽

composer install xxx , 这个install 和 xxx 都会作为参数传进去，通过 $argv可以获取到

### redis 的安装

需要注意的就是 redis 启动服务要手动指定配置文件，

1. 遇到的问题就是估计之前别人在路径中加入过redis-server 这个命令，导致我用 redis-server /opt/redis/conf/redis.conf 不匹配，出错，
2. 还有有个配置选项是是否展示cli 下 那个命令行redis图像，果断关闭，虽然不知道他为啥报错
3. 改成守护进程模式



之前测试测试数据库总的出错···总结了下原因
1. 不能清楚的认识到mysql怎么连接数据库的。其实换一种思路，php是怎么连接redis的，我们安装好redis服务，然后安装php的redis扩展，就可以了。什么连接函数啊和连接mysql是一样的。关于php连接mysql，现在主推两种方式，mysqli和pdo，分别对应两种扩展，具体的可以参考[菜鸟教程里面的这个php连接mysql](http://www.runoob.com/php/php-mysql-connect.html)。
2. mysqli() 第一个参数'localhost' 和'127.0.0.1' 的区别，localhost 连接基于socket，这个文件的位置在php.ini有指定，如果换了位置，则找不到。127.0.0.1基于tcp，就可以啦，不用那个文件，参考链接[segmentfault](https://segmentfault.com/q/1010000000328531)
   mysql 这个status挺好用的哈哈![](http://ozys8fka7.bkt.clouddn.com/mysql-status.jpg)




