---
title: vagrant使用
date: 2017-12-13 21:55:52
tags: [Linux,vagrant,虚拟环境]
categories: 工具
---

使用vagrant的原因？

其实主要是因为来公司的时候用的这个东西，相比于传统的xampp这种window下的一体化工具，vagrant其实是在vitrual box 之上的一种管理工具，他是基于虚拟机的，所以你可以在虚拟机上干的事他都能干。

首先xampp是window平台下的，他的lamp这个l不具备，所以，你xampp永远都接触不到linux，但是vagrant不同，他是基于虚拟机的，虚拟机可以装linux，所以使用vagrant，你可以使用linux。

使用vagrant首先得加载镜像，这个镜像就相当于最原始的环境，他是个box文件，想象一下，如果这个镜像是你们生产环境的打包，那你是不是就不用做别的处理就可以直接使用了，节省了新人安装环境的成本。

<!--more-->

其实vagrant 构建开发环境分为两大部分，第一部分是加载镜像，这主要和box有关，

so,先来介绍几个关于vagrant box 的相关命令

vagrant box add (镜像名自定义)  文件路径 （建议下载到本地，vagrant cloud 上面的文件下载很慢）

vagrant box  add（命名空间/文件名） 其实这样也是可以的，只是很慢

这样下载的文件放在了windows 用户 目录下面的 .vagrant 下面的box文件里面

这个可以修改的 setx VAGRANT_HOME = ""   路径名 /M 就可以了,/M代表整个系统都修改了

set 环境变量名 这样得到环境变量的值

set 环境变量=  这样是删除环境变量

vagrant box remove （box名称） 这样是删除box文件

vagrant box list 展示有哪些box文件

这样把远端或者是本地的box文件加载到系统中，如果是下载的.box文件就可以删除啦



镜像加载完成啦，接下来干嘛呢，就是在这个镜像的基础上构建虚拟环境，我们得用到vagrant的相关命令。

然后呢，新建个文件夹，建议vitrual，代表虚拟环境，再在下面新建单个项目的文件夹，这样的好处是vagrant虚拟环境不容易到处都有



进入对应文件夹（后面的这些vagrant操作命令都是在这个文件夹中进行的）

vagrant init （add的box文件名） 初始化，会生成vagrant  file 文件，修改其中的

config.vm.box = "new_itbasic"  这个代表box的名称

 config.vm.network "private_network", ip: "192.168.33.10"  私有网络只能自己通讯，相当于虚拟机里面的桥接，不能和局域网中的别的电脑通信（这个ip地址有要求吗）

public 相当于net，可以和局域网中其他通信，记得得是局域网中同一个网段哦

  config.vm.synced_folder "D:\\code\\yaf_api", "/home/itbasic" 设置共享文件夹，配置文件中这上面是两个`\\`应该是为了转义

  config.vm.boot_timeout = 100
  config.ssh.username = "root"
  config.ssh.password = "root"


这个是为了直接可以远程连接虚拟机

配置好后，vagrant up 第一次好像都是用这个，重新加载配置文件了，其实点开vitrual box，会发现自动创建个虚拟机，然后

不用的时候挂起 vagrant suspend，再启动 vagrant resume

vagrant halt 是关闭，vagrant reload 重新加载配置文件

vagrant status 查看虚拟机状态

vagrant package - -output itbasic.box 去自己的vitrual环境下，打包自己的环境成box文件，方便以后使用（好像vagrant 打包环境最好把虚拟机关掉，省略后面 - - output itbasic.box 默认生成package.box）

vagrant destroy 摧毁虚拟机镜像，这个是保存位置是第三个位置了，这个是由vitrual box决定的，在vitrual box的全局设定里面。



几个坑

1. vagrant box 可能就2.5g，可是新建的虚拟机可能要10g，这就是我之前环境一直错误，以为是虚拟机的错误之类的，其实报错信息很明显，是我的磁盘空间不足，这里不是指虚拟环境vagrant up的那个文件夹，而是c盘装的virtual box 装的虚拟环境的文件夹。
2. 新导入的虚拟机，可能没有mac地址，需要在网卡配置里面加入对应的mac地址，通过tail -f /var/logs/message 查看启动错误日志，看启动了哪些网卡，把不用的网卡onboot = no就好了，然后再systemctl start network,还可以systemctl stop networkmanger, systemctl disabled networkmanger, 这些不用的东西。
3. 关掉网卡之后可能ssh有问题。config.ssh.host="192.168.33.10" config.ssh.port="22" config.ssh.username="root" config.ssh.password="root"  把ssh先配置好，要不然改了网卡之后可能连接不上。config.vm.boot_timeout=100 配置好超时时间，这样就不用一直等了。



总结一下：vagrant构建环境其实一共有4个文件夹的位置

1. vagrant box的文件位置，box add镜像之后，.box文件就可以不要了（如果以后要用，可以直接打包生成）
2. vagrant 的虚拟环境，生成vagrant file 配置文件的目录
3. 基于box生成的供vitrual box 使用的镜像存储目录
4. 共享文件夹目录（用于写代码的目录）

上面的1，3两个大文件默认都是在c盘，坑爹啊，结果我的c盘越来越小



最近遇到的问题：

1.Homestead 需要的vagrant是2.1版本，我那个是1.9，为了升级，我直接卸载了vagrant，然后装了新的vagrant，奇迹就是 vagrant box list 的时候之前的数据还在，zz，看了下文档，vagrant分为程序部分卸载和数据部分卸载，我们单单通过windows卸载的是程序部分， 如果我们删除了用户下.vagrant，那之前的数据才会丢失

2.关于vagrant init 只是通过一个镜像文件生成一个vagrant file 文件，我们在使用homestead的时候，没有使用vagrant init ,直接是vagrant up，这样也是可以的，因为针对这个box的配置文件之前就已经存在了，homestead 中配置文件的名称是homestead.yaml

3.vagrant的那些镜像应该都有个默认用户 vagrant vagrant，虽然这个用户的权利也不是很大，但是我们能通过sudo 完成我们的绝大多数操作

4.之前那些suspend 的项目需要halt，然后up，如果直接resume 会有奇怪的问题



