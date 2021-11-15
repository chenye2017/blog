---
title: docker
date: 2019-07-06 15:23:39
tags:
categories:
---

学习docker主要是因为他部署环境太方便了，相比较传统的傻瓜式yum还有稍微高大上一点的自己编译，docker在搭建环境速度上真的是太快了，而且可以避免一系列的乱七八糟的问题，就比如用安装easyswoole 这个框架，当我们用composer + easyswoole install 生成目录结构的时候或多或少出现一点问题，但是当我们下载easyswoole 提供的Dockerfile 的时候，我们只需要

```
docker build . (包含Dockerfile的文件目录)
```

就能快速生成一个运行着easyswoole框架的环境，但是我不知道怎么把宿主机也就是本机的目录映射到docker app中，虽然通过docker run -v 可以，但是当我docker cp (docker app 中的easyswoole初始目录) (宿主机目录)，我发现总是有些文件夹没有复制出来，问题还没有解决，还有docker compose， docker 多app 分工合作还是没有弄回,慢慢学吧

<!--more-->

### docker 软件的安装（修改远程仓库的名称）

docker 是 cs 结构(类似 mysql, redis 都是，只是docker 客户端和服务端连接用的restful，感觉像走的http协议)，我们安装好docker之后，通过修改docker 的国内镜像，加速docker pull 拉取镜像的速度（如果是yum，修改/etc/docker/damon.json文件，可能重启systemctl reload docker 会报错，是因为systemctl start docker 的时候 -t 指定了resposity远程仓库，我们修改一下，删除这个参数就好了）

### 镜像(images)

docker pull   imagesName  //从远程仓库拉取镜像(docker hub, 类似github的存在,像阿里云这些都是对docker hub的备份，可能有几秒的延迟，没啥事)

docker images  //  查看本地所有镜像

docker search  imagesName// 查找镜像

(关于镜像名称， 仓库名(所有者)/镜像名称:tag, 要是没有仓库名就是官方镜像，tag 是标签，比如php-7.2:cli,这种细分一下)

docker run  imagesName (镜像名称)  //  默认先从本地找，找不到去远程仓库找

(特别要注意的就是run centos 这种操作系统的时候，我么需要docker run -it  镜像名称 /bin/bash ， 这样才能让这个app一直运行，否则他会立刻断开，没有实际的意义, 像一些简版的centos 比如 alpine 版本，可能没有 /bin/bash, 我们可以 /bin/sh)

(还有就是 不同版本的centos 安装软件的方式可能不同，可能是yum,可能是agp-get install, 可能是别的，我们只需要换源，然后百度安装对应resposity库下面的软件就好了)

(-r 代表运行结束就删除了， -d 守护进程方式， -p 8000:80 端口 映射， -v  /home:/home, 文件夹挂载， -name 给容器取个名字，否则默认分配个名字)

docer cp  (从宿主机到虚拟机或者从虚拟机到宿主机)

docker inspect imagesName // 查看一个镜像的详细信息，比如挂载了哪些目录，暴露了哪些端口

dokcer rmi  imagesName // 删除镜像 



### 容器 container

docker images 启动之后就产生了 container，容器

docker ps  //查看运行的容器

docker ps -a // 查看所有运行过的容器

docker exec -it containerid(容器名称) //进入容器内部

docker rm containerId //删除容器



Dockerfile 编写

docker compose

