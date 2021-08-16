---
title: Linux常用命令总结
date: 2017-11-28 01:20:28
tags: [Linux,命令,工具]
categories: Linux
---

1. netstat -an  查看网络连接，-a 表示包括tcp和utp，-n表示不用域名代替ip地址 
2. ps -ef | grep httpd  查看httpd进程是否在 (注意这个筛选是没有端口号的，你想啊，如果一个脚本就监听一个端口还好，要是类似 nginx apache ,listen 那么多端口，你怎么显示呢)
3. ll -a ls-la  展示该目录下所有文件包括隐藏文件
4. cd  进入目录，~ 普通用户的家目录 /home/vagrant, root用户 /home
5. pwd 展示路径
6. mkdir  创建目录
7. chmod -R (文件夹)  755 修改权限
8. chown root:root 文件名 修改文件所有者和组
9. chgrp  修改组
10. useradd  添加用户
11. passwd 修改密码  （切换到当前用户下就可以修改了哦，不知道怎么修改别的用户的）
12. groupadd 添加组 
13. touch 新建文件
14. echo 11 >> 文件名 把这个内容输出到文件中
15. vim  i 修改模式 ：wq 保存  ：q 退出不保存 ：q！强制退出  /  查找  ：num 多少行  ：$末尾  
16. cat  查看
17. echo 输出，比如输出当前进程号 echo  $$ (PID的值代表的就是进程号)
18. export 设置环境变量 export FOO = foo ,让当前环境变量立即生效，source ~/.bash_profile 执行这个脚本文件（source在当前进程中直接执行而不是复制子进程执行）感觉直接在命令行这样export FOO = foo 好像就直接生效了，但是如果修改bash_profile 文件的话，要执行source才能生效
19. env 查看环境变量，比如查看刚刚的环境变量是否生效 env | grep FOO 
20. cp 复制文件  cp 源文件 目标文件 ，如果是文件夹 记得带上 -r 参数
21. mv 剪贴文件
22. rm -rf 删除文件
23. lsof -i|grep 3306 查看端口号
24. tar -xjvf  解压
25. wc -l 查看行数
26. awk   cat /etc/passwd | awk -F ':' '{print %1}'  筛选内容
27. cut  切割, cut -d ':' -f 1,2   -d 按照什么进行切割，-f 要获取哪些行数，一般这两个搭配着用
28. sed  sed -n '2p' /etc/passwd  修改内容，类似程序用的vim
29. sort 排序
30. unique  经常结合sort 使用，因为unique 只能合并连续的两行
31. find  / -name 'php.ini'  查找文件
32. locate  查找文件之前 updatedb
33. ln -s  创造软链接
34. tail -f 
35. top 查看cpu啊内存啊之类的占用情况
36. df -h 查看服务器硬盘占用，itbasic 服务器就经常磁盘被写满了
37. wget 下载
38. curl 下载


常用端口号 ：

http 80

 https 443

mysql  3306

redis 6379

ssh 22



~代表家目录，root用户 /root,普通用户/home/用户名