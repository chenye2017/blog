---
title: 日志 awk sed
date: 2018-11-28 18:32:20
tags:
categories:
---

日志很重要
> 日志可以进行数据挖掘，分析用户的喜好，比如什么资源被用户频繁访问（对应面试题：取出访问量前20的网址）  --来自百度某朋友



> 你线上要是有错误怎么办，你们平时是不是都不看日志，只有错误了才解决，··balabal （一顿嘲讽）  --来自平安好房某专家表哥



> 为啥本地开发的时候，错误信息能输出在页面，线上环境直接就是个500页面，好奇怪，好奇怪     --来自17年的me

<!--more-->
###了解日志
我们itbasic用的其实是lamp, 通过apache的模块处理php, php的错误日志可以在虚拟域名里面进行配置
lnmp,nginx 起到静态文件处理的作用，php 转给php-fpm 进行处理，虽然网上说nginx 记录不了错误日志，但其实当我在虚拟站点里面配置error-log的时候，也是有记录的，和php-fpm中是一样的。
php-fpm 中开启日志需要修改php.fpm 的配置文件（还可以添加慢查询日志，[据说很重要](https://juejin.im/post/5b9394a0e51d450e686747e3)，有空的话可以把itbasic改成lnmp,有空的话看看php-fpm 中各个参数的作用，）

![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/5401815645624805999.png)
特别是这块，php_flag和php_value 的作用是一致的，不知道为什么要搞两个一样的，还有就是他们的使用是这样
```
php_flag[error_reporting] = 0
```
来覆盖代码中的error_reporting ,····原先以为还有什么作用呢

###分析日志

我不会告诉你最开始接触sed和awk 是因为面试题，很多时候，当我们分析一个大日志文件的时候，我们甚至都打不开这个文件，因为太大了， 但是我们可以通过这些文本处理工具对数据加以筛选。

awk（参照[阮一峰](http://www.ruanyifeng.com/blog/2018/11/awk.html)老师，很可惜，没有出sed 入门）

**基本用法**
```
# 格式
$ awk 动作 文件名

# 示例
$ awk '{print $0}' demo.txt

$ echo 'this is a test' | awk '{print $0}'
this is a test
```
print 和 echo 都是标准输出，stdin ,这个东西最近经常看到，感觉就是输出在控制台上，比如php cli 模式下，普通php 脚本中 echo,  print,  var_dump都可以打印在控制台上（但好像记得之前子进程中输出好像只有父进程中才能看到，不记得是return 还是 输出了），这种标准输出好像都可以通过管道进行连接然后进行处理
上面的分割， \$1 代表第一个元素，$0 代表整体元素
```
# 练习（test.log）：
root:x:0:0:root:/root:/usr/bin/zsh
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync

# 想输出第一个字符的话
cat test.log|awk -F ':' '{print $1}'
```
 想一下，我们通过cat ,是不是能把文件默认输出在页面上，然后我们通过awk去处理，设置分隔符号 ':',默认的分隔符符号是 ' '.

```
# 练习：提取nginx 500 日志，并统计总数
# 78.181.37.209 - - [28/Nov/2018:15:01:56 +0800] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.103 Safari/537.36"
cat access.log|awk -F ' ' 'if ($9 == 500) print $0' | wc -l
```
注意这个分隔符号是空格，即使双引号里面有空格，也会把切割开， wc -l 可以统计行数

**变量**
内置一些变量，比如行数NR，这一行有多少个字段NF(这两个应该是最常用)， 
```
FILENAME：当前文件名
FS：字段分隔符，默认是空格和制表符。
RS：行分隔符，用于分割每一行，默认是换行符。
OFS：输出字段的分隔符，用于打印时分隔字段，默认为空格。
ORS：输出记录的分隔符，用于打印时分隔记录，默认为换行符。
OFMT：数字输出的格式，默认为％.6g。
```
```
练习：
      2 345
      2 123
      1 789
      1 1234

awk '{print NR" " $2}' log  # 行数  第二个字段中间用空格连接
awk '{print NR" " $(NF)}' # 行数 最后一个字段中间用空格连接，运行的时候中间的值会被替换
```
**函数**
```
tolower()：字符转为小写。
length()：返回字符串长度。
substr()：返回子字符串。
sin()：正弦。
cos()：余弦。
sqrt()：平方根。
rand()：随机数。
```
**条件**
```
awk '条件 动作' 文件名
awk '条件 动作' 文件名awk -F ':' '/usr/ {print $1}' demo.txt
awk -F ':' 'NR % 2 == 0 {print $0}' demo.text #输出偶数行
awk -F ':' '{if (NR % 2 == 0) print $0; else print "error" '}' demo.text
```
```
练习
[例]：有如下文件test,请统计每个网址出现次数，用一句shell实现。
a [www.baidu.com]  20:00
b [www.qq.com]  14:00
d [www.baidu.com] 23:00
e [www.qq.com]  20:30
f [www.360.com] 20:30

 cat url.txt|awk '{print $2}'|sort|uniq -c|sort -rn
# cat 查看文件内容， 输出第二行的内容，排序，因为 uniq 只能把连接在一起的文件重复文件合并，然后再倒序
```
uniq -c  把重复的行数显示在前，但是只能合并连续的
sort 排序， -rn   r是倒序，n是按照数字处理
head -n 5  取出前五行，这也是查询的时候经常要用到的

补充一下
```
# 取出cpu 或者内存占用前五，top 查看占用
ps -aux|sort -k3rn|head -n 5
# sort 自带切割，
# ps -aux 可以查看所有进程， 比如之前我想查看我的一个脚本 server.php,可以通过 ps -aux|grep server.php, 相比较netstat -antp|grep 9000,查看端口占用情况， ps -aux 查看更加清楚，比如swoole 的进程，1个 master ,1个manage,剩下的work ，所有的进程都能查看到，但是netstat 只能查看到一个
```
统计某接口的调用次数

```
cat access.log| grep 'GET /app/kevinContent' | wc -l
```
统计报错接口

```
cat access.log| awk '{if ($9 == 500) print $0}'
```



