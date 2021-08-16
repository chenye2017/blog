---
title: Linux常见问题
date: 2018-01-09 00:17:31
tags: [Linux,常见问题]
categories: Linux
---

因为linux平时用的不是很频繁，所以经常会遇见一些问题，然后很容易忘记，然后还经常遇到，所以记录下来。

目录：

1. yum源替换
2. 修改密码
3. locate 查找文件
4. 添加环境变量
5. ~ 普通用户和root用户





<!--more-->

1.yum源的替换
我们每次安装centos的时候，如果不是打包安装，是从新安装的话，为了安装软件速度快点，我们通常会替换yum源，把国外的镜像替换成国内的yum源镜像。常用的有163，阿里云啊。
首先我们进入 /etc/yum.repos.d 文件夹下面，查看repo文件，把centos-base 这个yum源文件改名，作为备份文件（mv,剪贴，又能当做重命名来用，注意不是复制哦）。然后wget，国内的yum源地址，yum clean all 清除缓存，yum makecache 在本地生成源的软件信息，方便软件的查找。
总结：
1. cd /etc/yum.repos.d  进入目录
2. mv centos-base.repo centos-base.repo.bak  备份文件
3. wget -nc http://mirrors.aliyun.com/repo/Centos-7.repo   下载原文
4. yum clean all  清除缓存
5. yum makecache  生成缓存



```
*   阿里yum源:[http://mirrors.aliyun.com/repo/](http://mirrors.aliyun.com/repo/)
*   163(网易)yum源: [http://mirrors.163.com/.help/](http://mirrors.163.com/.help/)
*   中科大的Linux安装镜像源：[http://centos.ustc.edu.cn/](http://centos.ustc.edu.cn/)
*   搜狐的Linux安装镜像源：[http://mirrors.sohu.com/](http://mirrors.sohu.com/)
*   北京首都在线科技：[http://mirrors.yun-idc.com/](http://mirrors.yun-idc.com/)
```
2.修改密码
passwd 用户名 root用户，非root用户只能修改自己的密码哦，所以passwd后面不能加参数

3.locate查找文件

locate 命令的使用
之前一直使用find / -name 文件名，太慢了，使用locate吧。
首先yum install mlocate,然后updatedb ,就可以使用locate 文件名称。每次使用前locate 文件名称。（如果是安转好了，每次使用前只用updatedb,然后再locate 寻找就好了）

4.添加环境变量

lamp环境如果通过源码编译安装，安装的位置一般是/usr/local,命令一般在/usr/local/软件/bin/命令，每次使用起来这么一大串，是在麻烦，加入环境变量效果棒棒的
1. echo $PAHT 查看环境变量

2. 编辑/etc/environment文件，在里面更改PATH环境变量。

   例如：向环境变量中添加 /home/YRS/Nim/bin

   /usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin:/home/YRS/Nim/bin，保存后记得source /etc/environment 。重新启动后就不用source 。

   ​


5.普通用户的 ~代表家目录，比如/home/vagrant,但是root用户的这个~代表根目录 /home

6.别名 alias 这个还是挺重要的，编辑环境变量可以帮助我们添加一些可以直接执行的命令（类似封装了 /opt/php/bin/php -> php），别名其实和这个作用类似，但是如果我们安装了两个版本的php, 都添加到环境变量里面，容易分不清楚，我们可以取别名，比如 （alias php7.1='/opt/php7.1/bin/php' ）

```
// 实际例子
alias composer7='php7.2 /usr/bin/composer'
// 我们在使用php 进行composer 安装的时候提示php 版本过低
composer install 其实是  php composer.phar install
所以我就想到了
alias composer7='/opt/php7.2/bin/php /usr/local/composer.phar'
```

alias -p 查看所有的alias

vim ~/.bashrc  // 修改alias

source ~/.bashrc  // 让alias 生效




​     

