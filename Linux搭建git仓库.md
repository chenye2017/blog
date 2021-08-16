---
title: Linux搭建git仓库
date: 2018-07-24 19:52:40
tags: [git]
categories: git
---

！！！ linux 搭建git 仓库最方便的就是gitlab了，因为以前不知道，所以自己搭建了linux git 仓库，也没个图形化界面，是真的难用，但对git 的理解更加深刻了。（强烈推荐还是直接用gitlab 吧）

故事的发生背景：项目前端走了，前端界面没有人修改，只能先做后端，但是没有测试环境，而且这个bbs项目还被包含在itbasic里面，一旦代码提交到svn上，当我想更新服务器上的itbasic代码的时候，我一般直接在根目录下进行svn up来更新代码，这样会把bbs提交到svn上的代码也拉下来，所以本地的代码没法提交。以前我和前端镜像代码对接的时候，一般我都是先更新到svn上，然后前端进行下拉代码测试，当没问题的时候再去服务器上拉取svn的代码。但现在前端寻找不到，如若我更新bbs上代码，必然论坛会崩溃。
<!--more-->
后来老大希望测试下我写的代码，看看需要哪些参数，给了另一台服务器。
刚开始想到直接把修改好的代码打个包放上去，后来想了下不是长久之计，因为我的水平肯定要经常修改代码，不能每次修改下就打包一下，太麻烦了。
又想到可以通过配置itbasic服务器上的svn，当svn up 的时候忽略掉特定文件夹的更新，百度了下方法，大部分以下几种情况：windows下忽略文件夹，提交代码的时候忽略掉文件夹，很少有更新代码的时候忽略掉文件夹的解决办法。（有，但自己试验失败）。
后来想到不如利用git吧，首先大部分都是我一个人操作，我经常向github提交代码，操作已经很熟悉了，再者毕竟用git也能跟上时代的潮流嘛。


先来介绍一点基础知识：
当我们想下载github上的代码（或者说远程仓库的代码），我们可以通过
```
git clone git@ip地址：git远程仓库的地址
//例如
git clone git@github.com:chenye2017/zerg.git
//当需要下载github上的仓库的时候，直接去这个项目首页，会有个clone or download的选项，注意不要选择use https，因为选了那个每次提交代码额时候都要输入密码
//winodows用户可以直接下载zip
git clone git@192.168.56.10:/home/resposity/bbs.git
```
新建一个目录，自定义名称
进入到这个目录内，执行上述操作，就能把项目目录中的代码下载下来。

但上面仅仅只是下载项目，我们为了和远程仓库进行联动，我们需要在本地的这个文件夹建造一个git仓库，然后和远程仓库的某个分支比如master进行绑定，当我们修改完成的时候进行代码推送到远程的时候，别人下次再次clone或者pull就能收到我们提交的修改。
下面就是创建本地仓库的实例代码

```
cd dir
//进入上面执行git clone的文件夹
git init
git add.  //把修改的文件夹都添加进来
git commit -m 'init'  //提交到本地代码库
git remote -v //查看远端仓库
//如果有远程仓库，可以进行删除
git remote rm origin
git remote add origin git@192.168.50.16:/home/resposity/bbs.git //添加远程仓库
git push origin master  //和远程仓库的master分支绑定，以后提交只需要git push
git pull origin master //同上，以后使用也只要git push 就好了
```

当我们提交代码的时候肯定是不成功的，提示我们需要输入密码，···而且每次都会提示，怎么办呢，可以通过一个命令
```
ssh-keygen -t rsa -C "your_email@example.com"
```
在windows中，生成密钥位置会在C:\Users\admin 下面，这个下面有.ssh文件夹，进去复制末尾是.pua 也就是公钥的文件，粘贴到你的git仓库所在的服务器的git 主目录下，比如我的是 /home/git/.ssh/authorized_keys,粘贴到文件末尾即可，如果还不能提交代码，重启下sshd服务，netstat -antp|grep sshd, kill, /usr/bin/sshd 启动
如果还不行，可以查看这篇文章修改下配置 https://blog.csdn.net/dreamstone_xiaoqw/article/details/78355873

一直提交需要填写密码其实是个很常见的问题，收集了几篇文章，因为之前一直不成功，但后来成功了，但其实不是下面这几个问题，是我对git远程仓库理解出现了问题，所以操作不对.
https://www.jianshu.com/p/9dbb1dea5929， （文件夹权限），
https://ruby-china.org/topics/14182 （ssh真的出问题了，需要查看日志）

最后也是最终的要一步，按理说这应该是第一步的，就是linux上远程创建git仓库。
本身应该 1. 创建远程仓库 2 git clone 3. git push
这也是我们平时使用github 的主要步骤，但因为这一步最重要所以我放在了最后。

git创建远程仓库 http://www.runoob.com/git/git-server.html 菜鸟教程上已经说得很清楚了，只是我们需要区别一下两行代码
```
git init
git init --bare
```
第一个我们经常用到是实例化本地仓库，第二个是创建一个裸库，什么意思呢，就是这个裸库就是我们平时那个 git clone ip地址后的路径，这个仓库装的都是我们的提交记录，本身是并没有代码的，但我们通过git clone能生成最新的代码，以我的服务器为例
```
/home/resposity  //专门用来放远程仓库的文件夹
/home/resposity/bbs.git  //我的bbs远程代码库
/home/code/bbs  这是我的服务器上实际项目代码，也是被人访问我的项目的路径
```
当我本地修改好了代码（windows下），提交到远程仓库中，然后在服务器的/home/code/bbs (线上项目文件夹)下进行更新git pull最新的代码,别人就能访问到我本机的最新代码了。

对了，还记的我们当初svn代码更新提交失败，提示svn clean也不行的时候怎么处理的吗，就是清除.svn 下面两张表里面的数据 (通过Navicat建立连接).当时刚开始是不会这种方法的，同事告诉我直接删除.svn这个文件夹，说svn管理这个文件夹都是通过他，现在想起来挺有道理的，但没有试过（以前直接是删除了svn这个软件，然后肯定不成功啦，现在想想这和svn半毛钱的关系都没有撒，但感觉不能删除.svn,因为删除了，那些diff都找不到了）git应该也是通过.git这个文件管理的。

对了，关于免密码登录，当我们通过命令生成的ssh密钥linux下，一般在这个用户目录下面，比如root用户，在/root/.ssh 下面，我们需要把这个公钥粘贴到需要进行提交的git远程仓库的服务器中git的安装目录下面的ssh的auth···——key，里面，追加在文件末尾，注意着这两个地方文件名虽然类似，但一定要分清把那个粘贴到哪个上面，分清主从关系。



关于忽略文件夹，编辑.gitignore 文件，在项目根目录下面，因为windows下面不能这样命名文件，可以在linux下面编辑保存，编辑之后这个文件就是untracked形式,以后的提交就不会把这个文件保存在内，即使用了git add.

但是之前把.idea这个文件夹包含进去了，怎么办呢，git remove -n -r --cached

--cached  只是git仓库删除了，本地不会删除，-r 类似删除文件夹的时候递归删除， -n 先列出来要删除的文件，真正想删除的时候不要加这个参数。







