---
title: go 环境搭建
date: 2020-06-06 16:24:09
tags: [go,环境搭建]
categories: go
---

go越来越流行了，加上公司的业务需求，迫使我必须得开始学习go了

<!--more-->

学习任何一门语言，刚开始面对的第一个问题就是环境的搭建吧，相较于php的编译安装，go的环境搭建更加简单。

首先要明确的一点是go 是交叉编译，所以我们无论在win 还是 mac 还是 linux 下开发都是一样的。php 有很多扩展比如redis ，在win下版本比较少，所以我们不得不需要一个linux 环境， 我们平时开发php ，是把远程服务器的代码sftp 到本地，然后本地代码的修改实时上传到服务器上，在服务器上跑。go不需要 （goland 甚至连这个功能都没有，需要装插件）。

* 下载go的安装包，因为墙的原因，我们可以去 studygolang 上面下载，win和 mac 都是那种点击下一步傻瓜般的安装，linux 下安装也很简单，下载解压完之后就是一个编译好的二进制包（就像php 编译完成那样）。我们只需要把 我们环境变量的 path ，添加上 go下载包中的bin，就能直接使用go的命令了。linux 下可以直接编辑  /etc/profile, source /etc/profile 生效。保存好后 echo  $PATH，查看path 。

* go 安装需要注意的几个路径

**goroot ** 就是我们（linux 下解压完go 安装包的地址，win 和 mac 下是我们自己制定的go 安装路径）

**gopath** 是我们自己制定的一个文件夹，需要在这下面手动建立 bin， pkg， src 三个目录，

​	**bin** 是我们日后编译生成二进制文件的地方，

​	**pkg** 很重要，里面的mod 文件就是我们安装的第三方库所在的目录，

​	**src** 就是在 go mod 没出现前，我们存放我们项目代码的地方(之前就觉得我们的代码只能存放在一个地方，很不方便，go mod 的	出现就不需要管这个文件夹了）

go env 我们可以查看和 go 相关的 环境变量，比较重要的有  **GO111MODULE**, 怎么设置  go env -w 

**GO111MODULE**="on",  还有 **GOPROXY**=https://goproxy.cn,direct， goproxy.cn 就是 七牛云搞的golang 下载包中国镜像，后面的direct 代表如果找不到，就直接按照网址下载 (像bilibili 有自己搭建的 proxy, goproxy.bilibili.co)。

```
module：用于定义当前项目的模块路径。
go：用于设置预期的 Go 版本。这个不是强制的要注意
require：用于设置一个特定的模块版本。
exclude：用于从使用中排除一个特定的模块版本。
replace：用于将一个模块版本替换为另外一个模块版本。这个很方便，本地包调试
```

* go 安装私有模块也是很重要的一点，需要配置 GOPRIVATE= "git.chenye.cn" ，一旦遇到这个域名下的仓库，就不会走

  ```
  GONOPROXY=git.bilibili.co, 配置后对应域名代码的拉取不走goproxy.
  
  GONOSUMDB=git.bilibili.co, 私有仓库不走sum 检查。
  ```

  拉取私有仓库默认走http 方式，http 方式需要登录输入username 和 password ，可以通过 ssh code 方式，修改git 配置

  ```
  [url "git@git.bilibili.co:"]
          insteadof = https://git.bilibili.co/
  [url "git@git.bilibili.co:"]
          insteadof = http://git.bilibili.co/        
  ```

  

* 在用goland 编辑的时候，会遇到包找不到的情况，我们需要手动 seting 去配置 goroot  gopath。 goland 每个仓库感觉都要单独配置

![image-20220930155711499](/Users/yechen/Library/Application Support/typora-user-images/image-20220930155711499.png)

4.go mod 的使用，go mod init xxx 初始化项目，生成 go mod 文件，类似composer init 生成 composer.json，go run main.go 开始跑项目啦。会自动下载需要的安装包，这个相比于composer install 更加的方便。



今天在初始化别人项目的时候发现goland 的 go vgo 已经设置好了，但是还是找不到go module 的位置，仔细一看发现是自定义的package name 找不到。当我们想用go module 的时候，我们一定要 go init 起一个默认的命名空间，然后当我们使用自己定义的包名的时候，都会用到这个这个命名空间做前缀。



以上就是总结的 go 环境安装，完结~~



* go.mod 文件。原则上我们是不手动修改 go.mod 文件的，可以通过 go get 去安装。 比如 b站的 bapi 没有tag， 我们可以 直接 go get xxxc@latetest, 就能拉去最新的并更新 go.mod。 因为 bapi 没有tag， 所以都是这种自动生成的tag 名称 v0.0.0-20220920072008-4e460d33257e。 
* Go.sum 文件主要是为了帮助我们回溯， 因为会经常冲突，但问题不大。手动解决吧。如果都存在，就merge 解决吧。



比较好的文章：

* go 搭建私有仓库：https://tonybai.com/2021/09/03/the-approach-to-go-get-private-go-module-in-house/