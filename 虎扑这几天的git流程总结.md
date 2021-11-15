---
title: 虎扑这几天的git流程总结
date: 2019-11-12 16:04:56
tags: [git]
categories: git
---

首先说一下虎扑的代码开发流程（包括提交，发布流程不太明白，对金啃死了解太少）。公司给每个人分了台虚拟机(类似云主机)，

我原先的想法是通过在虚拟机上clone 代码，然后通过phpstorm 的远程连接主机的功能，实现在本地开发，然后映射到虚拟机上代码改变。可是这样有一个问题就是虚拟机linux上clone的代码的换行和我本地win上的换行不一样，虽然开发的时候没有影响，但是要是在win上提交代码就会出现很多没有修改的代码也会有modifyed（其实git 本身做了兼容，在clone的时候，但主要是我现在把linux代码拉取到本地，所以才会这样）。而且我也不可能在linux上提交，毕竟提交的时候需要通过一些工具观察文件的改变，所以改成了第二种方式

通过在本地和远程都clone 代码，然后通过sftp实现两个文件夹的关联，本地代码的修改也能直接映射到虚拟机，而且可以在windows上提交代码，因为是在win上clone迁出远程仓库的代码，所以提交的时候不会出现换行(crlf)的问题。

<!--more-->

首先是开发分支的命名。 

姓名/feature(bugfix) 特征/ 自定义名称 （这样切割之后，在一些git 工具上可以发现是以文件夹方式显示的）



开发完了，推送到远程

 git push origin chenye/feature/dgtag(本地分支):chenye/feature/dgtag(远程分支)

然后给develop 提交一个pr，再合并（最好不要直接给develop提交，就算提交也是通过git fetch origin develop(远程分支):deve1 去提交, 千万不要把自己的分支推到远程的develop分支上了，防止覆盖）

当我们提交的时候出现conflect 的时候，我们可以 git fetch origin develop:dev1, 然后在dev 上merge 我们的分支 chenye/feature/dgtag, 合并完了，再推送到远程新建分支，再把这个分支和dev分支合并。

当我们merge本地分支的时候肯定会出冲突，这个冲突的来源有两种可能，一种自己的，这个很好办，把head 到 ==== 中间删除，替换成自己最新的。

还有种可能是别人的，要是别人的呢，也好办，最暴力的方法就是用dev替换自己分支上最新的，这样自己的分支和dev合并就没有冲突了，但这个只适用于dev哈，如果是release 和 master ，一定要问一下出现conflect文件的那个人！！！



远程仓库更新完了，我们需要更新服务器上的代码，这和itbasic 一样，直接去28机器上git clone 就好了

唯一有区别的就是release， 我们这边release 的发布类似以前，每次发布之后都会在服务器上产生一个新的文件夹，通过改变软链接指向的文件夹，保证每次发布成新的代码（而不是通过代码覆盖，感觉这样是不是发布更快速，如果是clone的话，有时候需要很久才能全部完成）

进入release 环境，我们第一个选2， 第二个选项是不同的php， 7.1 和 7.2，7.2的count() 真的很坑呀，要注意，其中3环境我们每次执行完脚本要及时删除 文件夹下的cache， 否则脚本执行不了 （很奇怪~~）。

有时候我们在release 环境修改文件会发现没生效，这可能是因为有人发布代码了，所以你后面再次进入修改的文件，可能对应的文件夹已经不是之前你修改的那个了。



这几天git常用的几个命令

git pull origin master:master1 

注意这个和 git pull origin master 不一样，后者会下拉master 并和 当前分支合并，前者即使当前在master1 分支上，也是用master 强制覆盖master1 ，类似push 。

git fetch origin master:master1 

下拉远程master 在本地新建master 1

git reset   commitId  回退本地 commit 的内容到工作区（并不是删除）

(如果没有 --hard， 内容会再次回到工作区，需要 git checkout . 抛弃， 默认是 git reset --soft)

git checkout  .  删除工作区的内容

git clean  删除添加的文件，或者直接 rm  删除文件

git branch --delete  分支名 删除本地分支

git  push origin  --delete 分支名 删除远程分支



git merge  ing 的时候如果想放弃，也可以通过删除merge的文件，这样是不是能少记一个命令呢，哈哈哈



我们在解决冲突的时候，千万不要把develop，或者release 上面的代码合并到自己的开发分支上， 因为自己的开发分支最终是要合并到master上的，所以我们对这两个分支解决冲突只能通过新拉分支来解决，解决完也不要推送到自己的开发分支上，可以推送到远程的临时分支上，与dev和release合并



- 今天在给一个新项目做远程sftp配置的时候（本地clone，远程也是clone）,始终配置不上server， 

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20191112232234.png)

options 这块总是显示mapping 不对，打开 configuration

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20191112232326.png)

注意那个地方一定要有个 / !!!!



- 今天用sftp 发现，为了删除文件，只能在phpstorm上面删除，如果自己打开文件夹删除，是更新不到服务器上的



- 今天用git 的时候，想给部署多个仓库

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20191205161629.png)

小垃圾们看一下，~是不是对origin 有更深刻的理解了 ，地址[https://segmentfault.com/q/1010000008366409](https://segmentfault.com/q/1010000008366409)



- 今天在推送本地分支到远程的时候推送错了，导致远程分支提前自己的开发分支，所以只能强推，当然只有自己的分支能这么坐哈，~公共分支千万别，因为感觉强推就是覆盖

git push origin dev -f



- 今天在提交代码的时候，发现想删除某个提交，其实这个情况蛮常见的，比如之前某个大佬把release合并到master上，我们只需要



git rebase -i  还是蛮重要的，目前我就用在删除当前分支的某次提交。变基的时候如果没有修改同一个文件的话，没有冲突的话，改起来话挺快的，可以快速删除某个不想要的提交，比如：[https://www.36nu.com/post/275](https://www.36nu.com/post/275)，

但是如果改了同一个文件的话，可能存在某种冲突，所以变基操作会每个commit 慢慢操作，只需要 git rebase --continue 这样就行了

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20191206114054.png)



- git reset 的后遗症蛮严重的，特别是自己开发分支修复不当，所以我们每次提交一定要严格执行合并dev， release，master 流程，一旦开发分支修改过，都要重走一遍。今天有人合并分支出现冲突，强推了merge，导致我的代码被覆盖，但归根结底都是之前提交代码不规范，~一定要引以为戒



最近虎扑多了个新的git tag的功能，主要是为了发布代码的稳定性，方便回滚。

git tag  [https://www.zhihu.com/question/28784805](https://www.zhihu.com/question/28784805)  这篇文章讲的蛮好。

需要注意的就是我们要在自己想要的branch 上打tag （就是对于自己想要的commit 取一个别名）



- 重温了git 小册，看打了git rebase ，其实这个功能大家不常用很正常，因为我们现在往往提交代码都是通过pr的方式，合并代码都是在gitlab 上手动执行就好了，很少关注git merge 这些事情，本地也很少merge 代码 （我一般开发一个功能就是fetch 一个 master as other branch，然后去开发）