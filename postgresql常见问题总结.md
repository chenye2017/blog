---
title: postgresql常见问题总结
date: 2018-04-17 15:47:01
tags: [postgresql,数据库]
categories: postgresql
---

本来是不想写这篇文章的，因为感觉自己对于postgresql理解不够深入，但因为生活中对于postgresql有些问题总是遇到，所以记录下来，方便以后查找。

<!--more-->

1.navicate连接postgresql

第一次建立连接肯定会报错啦

![navicate连接postgresql属性](http://ozys8fka7.bkt.clouddn.com/TIM%E6%88%AA%E5%9B%BE20180417155030.png)

上面初始数据库的意思并不是数据库的名称。毕竟这次连接成功后，里面有一堆的数据库

> The server doesn't grant access to the database: the server reports 
> FATAL: no pg_hba.conf entry for host "192.168.0.123", user "postgres", database "postgres" FATAL: no pg_hba.conf entry for host "192.168.0.123", user "postgres", database "postgres"
>
> 

谷歌翻译

> 服务器不授予对数据库的访问权限：服务器报告
> FATAL：主机“192.168.0.123”，用户“postgres”，数据库“postgres”没有pg_hba.conf条目FATAL：主机“192.168.0.123”没有pg_hba.conf条目，用户“postgres”，数据库“postgres”

其实字面意思就很容易理解，虽然我还是百度的错误答案。我自己理解就是入口不被允许,就觉办法如下

> PostgreSQL数据库为了安全，它不会监听除本地以外的所有连接请求，当用户通过JDBC访问是，会报一些如下的异常：
>
> org.postgresql.util.PSQLException: FATAL: no pg_hba.conf entry for host
>
> 
>
> 要解决这个问题，只需要在PostgreSQL数据库的安装目录下找到/data/pg_hba.conf,找到“# IPv4 local connections:”
>
> 在其下加上请求连接的机器IP
>
> host all all 127.0.0.1/32 md5 //坑爹，那里面对于192.168.23.10当做192.168.23.1
>
> 32是子网掩码的网段；md5是密码验证方法，可以改为trust
>
> 

一个服务器，有内网ip（类似192.168这些，有外网ip 211.这种，有127.0.0.1 这种，如果要包括所有的，就得用0.0.0.0，感觉如果你访问一个服务器用的是内网ip，应该意思就是你们在同一个局域网中吧。很奇怪的一点是，之前说vagrant会把ssh的22端口映射到host的2222端口，导致了127.0.0.1:2222或者192.168.23.10:22都能访问，其实不太懂这个原理，但好像这是由两个不同的网卡决定的，我好像把itbasic那上面一个网卡禁止了，导致127那种ssh访问不了，感觉就是虽然内网ip和外网ip可以对应同一个端口，但也可以对应不同的端口）

2.导入导出数据库

导出：/opt/pgsql/bin/pg_dump -U postgres itbasic > /home/itbasic/itbasic.sql

pg_dump 是postgresql安装之后的一个命令，-U 指定用户 postgres, itbasic,是数据库名，postgresql好像只能用postgres这个用户执行操作，itbasic这个数据库名一定要指定，还有pg_dump 这个命令所在目录没有加入系统变量的话，一定要加上路径，后面 >/home/itbasic/itbasic.sql,代表数据导出到具体的文件中。

导出

数据库的导入必须要先创建这个数据库

postgresql安装之后的命令会有createdb，但用这个创建数据库感觉挺麻烦，直接

```
1. /opt/pgsql/bin/pgsql -U postgres itbasic  //进入pgsql 客户端
2. create database project x   //创建数据库
3. \q   //退出pgsql客户端
4. /opt/pgsql/bin/pgsql -U postgres projectx -f /home/itbasic/projectx.sql   //导入数据库
```

3.导入导出表

导入导出表和导入导出数据库的最大区别是导入导出数据库需要先建立一个数据库，而表则不用，最好还先把这张表删除了

导出

/opt/pgsql/bin/pg_dump -U postgres itbasic  -t dt_admin > /home/itbasic/admin.sql

导入

/opt/pgsql/bin/psql -U postgres itbasic -f /home/itbasic/admin.sql

4.数组类型

array_remove  会把出现的元素全都移除掉，而不是只移除一个   love=array_remove(love, chen.ye)

array_append   love= array_append(love, chen.ye)

如果元素不存在，也不会报错，只会接着往下执行

any()  chenye = any(love)  love这个字段中是否有陈野

5.今天在到处数据的时候发现了一个不得了的问题，就是通过命令导出的数据的时候当文件一定大的时候，会自动停止，但是导出语句不会报错，这就坑爹了，也就是说我每天备份的那些文件都没有任何的意义

可以通过

```
/opt/pgsql/bin/pg_dump -U postgres itbasic -t '(dt_user|dt_user_login|dt_userinfo)' > /home/itbasic/t4.sql
```

分表导出

表名的话可以通过

```
select tablename from pg_tables where tablename like 'dt_%'
```

筛选出来再连接，果然定义成有前缀好处颇多，要不然筛选出好多自带的无用的表

6.感觉要多看看别的项目是怎么进行数据备份的，之前的那种一次性导出的方式不行，当数据量大的时候，单表备份可能都不行。