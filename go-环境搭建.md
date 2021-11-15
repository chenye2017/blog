---
title: go 环境搭建
date: 2020-06-06 16:24:09
tags: [go,环境搭建]
categories: go
---

go越来越流行了，加上公司的业务需求，迫使我必须得开始学习go了

<!--more-->

学习任何一门语言，刚开始面对的第一个问题就是环境的搭建吧，相较于php的编译安装，go的环境搭建更加简单。

首先要明确的一点是go 是交叉编译，所以我们无论在win 还是 mac 还是 linux 下开发都是一样的。php 有很多扩展比如redis ，在win下版本比较少，所以我们不得不需要一个linux 环境， 我们平时开发php ，是把远程服务器的代码sftp 到本地，然后本地代码的修改实时上传到服务器上，在服务器上跑。go不需要 （goland 甚至连着个功能都没有，需要装插件）。

1. 下载go的安装包，因为墙的原因，我们可以去 studygolang 上面下载，win和 mac 都是那种点击下一步傻瓜般的安装，linux 下安装也很简单，下载解压完之后就是一个编译好的二进制包（就像php 编译完成那样）。我们只需要把 我们环境变量的 path ，添加上 go下载包中的bin，就能直接使用go的命令了。linux 下可以直接编译 /etc/profile, source /etc/profile 生效。保存好后 echo  $PATH，查看path 。

2. go 安装需要注意的几个路基

   goroot  就是我们（linux 下解压完go 安装包的地址，win 和 mac 下是我们自己制定的go 安装路基）

   gopath 是我们自己制定的一个文件夹，需要在这下面手动建立 bin， pkg， src 三个目录，其中bin 是我们日后编译生成二进制文件的地方，pkg 很重要，里面的mod 文件就是我们安装的第三方库所在的目录，src 就是在 go mod 没出现前，我们存放我们项目代码的地方(之前就觉得我们的代码只能存放在一个地方，很不方便，go mod 的出现就不需要管这个文件夹了）

   go env 我们可以查看和 go 相关的 环境变量，必要重要的有  GO111MODULE, 怎么设置  go env -w 

    GO111MODULE="on",  还有 GOPROXY=https://goproxy.cn,direct， goproxy.cn 就是 七牛云搞的golang 下载包中国镜像，后面的direct 代表如果找不到，就直接按照网址下载。go 的第三方包的命名很好玩，基本就是这个仓库所在的可以访问的url 路径，比如 code.chenye2017.cn/xxx/xxx, 不需要像composer 那样中间走一层代理，那层代理还要给你开访问权限，才能下载。

3.在用goland 编辑的时候，会遇到包找不到的请款，我们需要手动 seting 去配置 goroot  gopath ，还要开启vgo（就是gomod）

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20200606164735.png)

这个地方要选空，否则还是会出现包找不到的问题。

4.go mod 的使用，go mod init xxx 初始化项目，生成 go mod 文件，类似composer init 生成 composer.json，go run main.go 开始跑项目啦。会自动下载需要的安装包，这个相比于composer install 更加的方便。



今天在初始化别人项目的时候发现goland 的 go vgo 已经设置好了，但是还是找不到go module 的位置，仔细一看发现是自定义的package name 找不到。当我们想用go module 的时候，我们一定要 go init 起一个默认的命名空间，然后当我们使用自己定义的包名的时候，都会用到这个这个命名空间做前缀。



以上就是总结的 go 环境安装，完结~~