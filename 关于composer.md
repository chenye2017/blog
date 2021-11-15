---
title: 关于composer
date: 2018-07-29 23:41:17
tags: [PHP,Composer]
categories: PHP
---

composer,php的包管理工具，类似nodejs的npm，类似python的pip，因为很多时候我们需要接入外来的服务，可能自己的能力不足，又或者是别人有更好的解决办法，不是有说过站在巨人的肩膀上能看的更远么。

<!--more-->

composer require  当我们需要安装一个外来服务的时候，注意这个不是安装php的扩展模块，两者有本质上的区别，php的扩展模块是c写的，需要我们源码编译到php中, 而通过composer接入的是php封装的内容。 之前我经常把需要的模块通过手动写入composer.json文件中，然后通过composer install 或者 composer update 这种方式更新，这都是不对的

composer install  会依据composer.lock 文件进行安装，如果没有这个lock文件按照composer.json进行安装

composer update 会依据composer.json 进行安装
（composer.lock 和 composer.json 的区别lock文件中会严格规定版本号）

composer init 初始化一个项目被composer 管理（很少用, 一般是手动编写一个composer.json文件，然后install）

composer 还有文件夹管理的功能，现在一般用的是psr-4,原先项目组织有psr-0,现在废弃了，二者区别就是psr-4通过命名空间更能代表文件位置
composer的自动加载有psr-4,我们那个api服务中有使用，注意定义了一个顶级之后，下面的子命名空间可以自动寻找，而不需要你的人为干预
files 可以加载配置文件之类的，不用一直include
classmap可以不遵循psr-4规范，会扫描指定文件夹，加载里面的类（一般的类名首字母大写，然后类名和文件名一样）
当我们修改了composer的自动加载，通过composer dump-autoload来更新


我们php的框架和python 这种框架有挺大的区别，写了点python，还有问了写python的同事，得到的就是他们写python的时候还用的是面向过程的方式，他们的框架flask感觉更像是php中的包的概念，通过import方式导入，没有各个文件夹的内容规定，给了很大的自由给用户，我们通过composer 安装的文件一般在这个项目文件的vendor目录下，但是我们python 通过pip安装的一般是全局安装，就算是用了pipenv 或者vrtualenv 也是在全局下面然后分成各个子文件夹（仿佛是虚拟站点让php变得这么方便？）

composer需要注意的东西：
1. 修改源，下载东西更快
2. linux 下安装（composer.phar 就是 php composer, 只是单独使用的话需要composer.phar 需要执行权限 chmod a+x composer.phar）






使用composer 发布一个自己的扩展包

编写项目用的是超哥的脚手架工具

```
composer global require 'overtrue/package-builder' --prefer-source
package-builder build ./login  ##后续查看是否要修改一些composer.json 文件
```

 可能也许是因为我在packages 上的登陆用的是github的，所以自动配置了扩展包的自动更新(可以自行测试，修改readme 文件，github上的，看packages 上是否自动修改，如果没有自动修改，谷歌自动更新的方法

[同步方法](https://blog.csdn.net/whq19890827/article/details/79705531)

[完整的发布流程](https://juejin.im/post/599fc579f265da24934b0599)

总结 ：其实就是去package 上面注册一下，然后把自己的项目地址(git 仓库地址)粘贴上去就好了，能自动识别，(而且感觉还会自动配置自动更新，否则去项目的setting 下面自己设置)

![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/2278415645624747652.png)

遇到的问题：

- package上面有，composer require 不到

脚手架自动生成的read me 上的composer require 有问题，别用那个自动生成的，（之前怀疑是源的问题， 虽然用的是中国源，但是更新的也是很快，接近实时，所以package上面有，composer require 不到，是自己的问题， 版本号写错了）

![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/919011564562475045.png)

（超哥那个自动生成的md文件会被package 上面 直接显示（嵌入的毫无痕迹，我还以为是package 网站生成的，于是我就直接使用read me 上的composer require 语句 ，报错）

我们的composer require 语句应该和composer.json 相匹配

![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/382151564562475279.png)