---
title: Laravel migrate 数据迁移
date: 2018-07-29 23:42:19
tags:
categories:
---

虽然表哥说他们都不用这种方式，但这种方式确实很方便的解决了团队合作的时候数据库表的同步问题，像我们itbasic 就每次上线或者给队友同步数据库的时候都得自己手动去处理，而且这种方式更多的把sql语句转换成php语言处理，不知道建索引那些会不会很方便.

<!--more-->

文件位置和命名：

- database/migrations/2014_10_12_000000_create_users_table.php

通过artisan 命令能直接生成对应目录下的文件

注意up 和 down 方法，分别是的执行的时候，还有回滚的时候的执行

注意create 和 update 表结构的时候的区别

自增：increment 

字符串： string  ， 方法的第二个参数限制最大长度

唯一： unique，感觉就像数据库的唯一索引

创建时间和修改时间，通过 timestamps 就能生成

记住我： rememberToken, 虽然一直不太清楚这个干啥的



create =>  dropIfExists() , 创建 回滚的时候对应 删除

