---
title: 关于git
date: 2018-09-07 15:51:06
tags: [git]
categories: git
---

公司一直用svn, 怕git 的相关内容以后用到，所以学习了，只是简单记录git 使用过程中自己的问题，不是教程，欢迎一起探讨

<!--more-->

* 首先是分布式（不需要联网，本地就是一个完整的环境）和集中式（需要联网，否则运行不了）的区别

  分布式就是在每个人的个人电脑上都有仓库，称作本地仓库，本地仓库也会保存整个项目的提交记录，让我们在不连网的情况下也能进行代码的提交，当我们可以连网的时候，再把代码推送到远程仓库，远程仓库只是一个让我们各个工作者可以相互交换提交记录的中转站，我们的提交在git commit 的时候已经完成(我们唯一的提交信息就是在commit 时候写的)，也就是在git log 历史里面已经占据了位置，我们在push 的时候只是用本地的提交记录去覆盖远端的提交记录。而集中式代表着我们在不联网的情况下完全工作提交不了代码，那个中央仓库才是我们的唯一仓库，往里面提交代码才会产生log。所以我们在用集中式的时候，查看log都是通过查看远端log，对比diff也是，所以断网的情况下完全用不了,而分布式完全没有这个问题。

  查看分支也是同样的道理，git branch -all , 可以查看本地的所有分支，包括远程分支 。这里提到的远程分支并不是远程仓库上的实际分支，而是你上一次git pull 的时候下载到本地的分支，如果这时候别人新推送了分支到远程仓库，你必须git pull 重新下载一次，否则还是看不到他提交的远程分支。

* git add .

  我们每次修改代码之后都需要git add 文件名(. 代表所有文件)添加到暂存区，当我们添加一次之后，又修改了文件，还得git add 一次，因为git add 只是把当前文件现在的修改添加到了暂存区，如果后面修改了，还得重新添加一次，这个可能和svn 的只需要add 这个文件到管理文件夹内，后面修改直接commit 有点不同， git add 每次添加的是文件修改内容，而不是整个文件

* up-to-date

  最新的，有的时候可能显示before one  commit 这种，代表你要给远程分支push啦、

* git init

  把本地文件夹变成git 仓库

* git remote add origin  git 地址

  为本地仓库添加远程仓库

* git remote -v

  查看配置的远程仓库

* git clone

  做了两件事情 下载了.git 本地仓库，然后把当前目录当做工作目录，把master 最新的提交内容填充当前目录

* .git 

  .git 就是本地仓库，下面的index 代表了暂存区

* git comment -m 

  git comment 是实际的提交，会产生log 日志，像我在做itbasic 的时候经常一个功能提交多次，因为有的代码忘记提交了，可以把comment 合并，产生一个log

  git comment --amend ,修正上次的提交，很方便，可以合并多个提交内容到同一个comment (也就是同一个log 点)

* git push 推送本地log 覆盖线上log (覆盖的时候会自动检测，如果没有冲突，直接合并。如果有冲突，比如 节点不能往前推进，会让先git pull)

* git 提交流程

  git add .

  git commit -m

  git pull 冲突发生的地方，pull 只会把需要的内容拉下来，并不能把你之前提交到本地仓库的内容push, 所以pull 之后还是需要push的（感觉这个pull还是挺智能的，在pull的时候如果有冲突，比如修改了同一个文件，还是没有提交到暂存区的那种，需要你先commit 或者废弃（感觉不要动不动就pull, 现在每次开发新的内容，都自己新建一个分支，然后直接在当前分支开发，开发完了把当前分支推送到远程，最后统一合并到master上）。想象一下pull 中包含merge 操作，merge 操作的对象是线上的commit 和 你最新的commit，所以如果有未提交的内容，一定要注意啦，别被覆盖了，但是如果没有关联的文件修改，pull的时候有没有commit 没关系）

* git branch

  branch 字面上是分支的意思，其实就是一次提交的引用，类似head, master （远端分支head 一定指向master， 可是我们本地head 可以切换到别的分支上呀，而且对应的好像是这个分支的最新提交）

  因为branch 是一个引用，所以我们删掉任何的branch ，都不会删除commit

  自己的分支，因为回退之类的不能修改，那就强行 ， git push origin branch1 -f ，但这种只针对自己吧 （千万别在master 上强行push）

* git checkout

  git checkout -b  feature, 我们新建一个分支并切换到这个分支上面，想把本地分支提交到远端只需要git push origin feature：feature 就好了

  还有就是注意我们在切换分支的时候一定要commit ,想象一下，如果我们没有commit ，在下次切换回来的时候，很多内容是不是就没有了，因为切换到的是这个分支最新的commit 上面  (我还是觉得切换分支的时候一定要commit， 想象一下切换分支的那个动图)

  git checkout -b feature origin:feature , 注意一下这个和上面的区别，上面的feature 其实还是基于master搞出来的，但是这个是基于远程的feature 搞出来的，有本质上的不同

* git rebase
  主要是用来分支合并的，比如之前在master 分支上开出来的feature 分支，开发完了再合并到master分支上，这时候master分支已经不是原先的master了，可能有向前的提交，就会出现分支的合并，如果不希望出现多条分支合并的情况，可以通过切换到feature分支上，然后git reabase mater, 以新的master分支作为起点 （git rebase 之后，只会把master 分支多加commits, 并不会切移动head, 但是能把当前分支的点移动到master 上，也不会把master  主动移动，需要我们主动切换head 到master 上，master merge 提升head 和自身（master））后续操作：git checkout master, git merge feature,

* git reset --hard commitId

  git reset --hard 目标commit 到达任意一个commit ，我觉得直接用sha-1码很方便，小册中说的回退，但是用sha-1码的话，我感觉能到任意一个commit （回滚）


* git 和 github 的互动

  git 在我们日常生活中用的最多的就是github，当我们一台崭新的电脑希望从github 中clone 代码的时候（比如git clone xxxx），我们需要ssh 连接到github 上，不做任何处理的情况下会被告知ssh 连接失败，我们需要做的就是在本地电脑上先生成ssh 公钥和私钥（如果没有的话），然后把公钥贴到github 上，公钥的生成过程中需要一个email， 这个其实随便填都是可以的（既然email 是随便填的，但怎么识别这个公钥对应这个用户呢，我觉得应该是这个公钥被贴到对应用户的账号setting处，所以github能识别这个ssh key 对应这个用户，[这里有篇文章写得https 和 ssh 之间的区别](https://debugtalk.com/post/head-first-git-authority-verification/)）。(这个email便于区分贴的ssh public key 是谁的)。

  我们在仓库的提交过程中也需要配置email 和 用户名，其实这个只是代表着你显示在git log 日志上的用户名和email ，并没有实际的作用

- 今天给easy swoole 提交pr ,详细过程[阮一峰](http://www.ruanyifeng.com/blog/2017/07/pull_request.html) (要注意的核心就是我们提交的pr 具体是哪一个分支，不能只会使用master 分支)

  首先是从fork代码从easyswoole 到自己仓库中的时候，注意fork的是哪个分支（好像不能把所有的分支都fork下来），然后git clone 到本地仓库，这时候通过编辑器打开，可以把fork 的所有版本都打开，通过

![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/4524515645624759596.png)

编辑器右下角的切换，可以在本地创建对应分支

```
git branch   本地分支
git branch -a  远程分支
git branch -vv 分支对应情况
```

通过编辑器切换可以自动添加分支关联，如果想手动关联不同分支，查看这篇[文章](https://www.jianshu.com/p/fc433b1686bd)

总结：所以 git push origin master , 这个master代表的就是分支名称喽 （git push origin master:master 其实是这样的，origin 代表的是远端仓库名称，两个master 分别代表本地的和远程分支名称）

* git  只能记录纯文本的修改，不能记录word 这些二进制文件的修改，对于这些二进制文件的修改，只能知道他大小的具体变化，不能知道具体修改的内容，所以我们平时的文档都用md格式？
* git 除了支持 ssh 协议，还支持 https协议 （但是https 协议下载的时候总要输入用户名和密码，所以一般还是走ssh吧）




虎扑代码发布 提交流程

- 自己本地新建分支用于开发，基于远程master 分支, 写完了，推送到远程分支，和develop 分支合并（自己有合并权限）

// git checkout -b  cheney/test  origin:master  通过远程master 分支新建分支 （因为我本地master分支有些没用的东西， 所以本地master 分支不要做任何的事情吧）

- 还是不要用上面的新建分支的命令吧，容易出错。用git fetch origin chenye/test:chenye/test, 新建本地的分支


- 对于分支切换的时候的错误，是在不行就 git add . git commit -m  。但这样以后千万不要随便提交，比如我本地的master分支

对于我熟悉的分支切换时候的错误，就git stash ,git stash pop 恢复吧

- 给master 分支提交的时候一定要记得merge， git pull origin master:chenye/test, 我曾经收到一个这样的提示

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20191012121216.png)

这个原因是master 和 我本地分支完全一样，没必要merge (而且本地有提交， 如果本地也没有提交，如下)

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20191012121521.png)



- git  push origin chenye/test:chenye/test 推送本地分支到远程分支



- 测试完develop 分支，再把自己的分支和realease分支提交合并请求，此时并不用拉取 master分支



- realease 分支测试完，给master分支提交，一定要提前和远程master分支合并，也是给master提交 pr （并不用和release 还有 develop 合并，因为我们只需要把我们的分支提交给master就好了，release 还有 develop 上面可能有很多别人尚不稳定的内容）

git pull origin master: chenye/test   拉取远程master 分支和本地分支合并 （git fetch 更安全点，和 pull 的区别 ？）



- ！！！注意，我们在用sftp 连接远程代码的时候，本地分支的切换并不能保证远程代码的切换，所以为了不出错，需要我们自己手动切换分支
- 本地某个分支写了很多，但没有提交，所以切换的时候就容易出错，首先如果可以的话，我们git stash 暂存，如果是untracked ，可以通过 git add. ,再git stash，按理说前面两种就能解决所有问题了，实在不行最后只能git add ., 然后 git commit ，但这样以后千万不能在这个分支是 git pull


所有的代码拉取都是 可以跨分支，分支的推送都不跨分支，所以合并都是通过pr 方式，



git 撤销操作

对于已经add 的内容 （一般显示绿色）

通过 git reset 撤销，撤销完了之后还需要通过 git checkout (因为上面的操作只是撤销暂存区内容，所有内容都返回工作区，所以还得对工作区内容进行处理)



对于 没有add 的内容（一般显示红色），但是 tracked 的 文件，通过

git checkout . 撤销  git checkout  文件名，可以撤销具体的文件

对于没有add, 也没有 tracked 的 文件，通过

git clean ， git clean -n  先预览删除哪些文件，防止误删除， git clean -f  删除具体的文件。（clean 的话下面这些蛮好用的，注意加上n, 查看会删除哪些文件，对于编辑器产生的内容，不想删除，可以直接添加到.gitignore, 提交之后看看有什么变化，其实编辑器产生的内容没啥影响，直接删除就好了，下次又会自动产生）

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20191206111607.png)

git 文件在checkout 或者提交的时候，如果是跨平台，可能遇到 \r\n (win) 和 \n(linux) 的问题，一般情况下 git bash 都配置好了，只有那些比如我远程代码linux 下，sftp 到本地 win 下，才会出现这种问题 （git 自身的兼容只有在 checkout 或者 pull的时候，如果是通过别的方式比如sftp，好像不行），目前尚未解决 



原先我调试代码的方式是虚拟机里面clone， 然后phpstorm 远程连接上去，本地代码拉取一份，这样本地代码修改，远程可以直接自动修改，唯一缺点就是上面的crlf 不能实现兼容（本地拉完代码，git status 就会发现大量代码的modifyed），如果是mac 的话，应该蛮好处理上面的问题, 因为他们的换行符是一致的

后面我是本地clone代码，然后sftp 到远程，这样就能避免上述换行符问题。唯一需要注意的就是本地分支切换和远程分支都得手动切换



关于phpstorm 切换分支，远程目录也切换分支，如下

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20191017100223.png)

所以我们就把 sftp 和 ftp 当做一种远程同步代码的方式就好了

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20191017100319.png)

还有要注意的点就是我们配置了Configuration（远程文件夹位置的配置） ，也要注意一下options， options 对我们的保存操作（是否改动就上传），哪些文件需要被同步会有配置（.git 文件一定要上传，方便我们切换分支）