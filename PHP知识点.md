---
title: PHP知识点
date: 2018-03-26 14:08:38
tags: [PHP]
categories: PHP
---

PHP中很多稍微高级的知识点自己都不是很熟悉，trait, 闭包，又或者composer等， 记录下来，成为现代phper。

<!--more-->

-  [PHP中闭包](https://chenye2017.github.io/2018/03/26/PHP%E9%97%AD%E5%8C%85/#more)


- PHP 时间处理(DateTime)

  ```
  // 打印当前时间
  $a = new DateTime();
  echo $a->format('Y-m-d H:i:s');

  // 打印特定的时间（接受一个特定的时间，反应出周边信息，感觉通过DateTime 这个类就能获取一个时间点的所有资源）
  $a = new DateTime('2019-02-21');
  echo $a->format('U'); // 转换成一个unix 时间戳
  $a = new DateTime('tomorrow'); // strtotime 的所有功能都能实现(既能接受日期，也能接受string)
  echo $a->format('Y-m-d')
  $a = new DateTime('+2 days')
  echo $a->format('Y-m-d H:i:s')

  // 还可以在获取到一个事件之后进行时间的修改
  $a = new DateTime();
  $a->modify('+1 day')
  echo $a->format('Y-m-d')
  $a->setDate(2018,1,19)
  echo $a->format('Y-m-d')
  $a->setTime(9,9,9);
  echo $a->format('Y-m-d')

  // 转换时间戳
  $date = new DateTime('@1408950651');
  $date->setTimezone(new DateTimeZone('Asia/Shanghai')); // 这个时区很重要，影响转换的内容
  echo $date->format('Y-m-d H:i:s');
  echo "\n";

  // 比较日期差额
  $date1 = new DateTime();
  $date2 = new DateTime('2014-09-15');

  $diff = $date1->diff($date2);
  echo $diff->format("The future will come in %Y years %m months and %d days");

  // 比较日期大小
  $date1 = new DateTime();
  $date2 = new DateTime('2014-09-15');

  if($date1 < $date2) {
      echo $date2->format('Y-m-d H:i:s') . ' is in the future';
  }

  总结对比：
  这个new DateTime() 能接受的参数有很多，很方便
  $a = new DateTime('2019-2-21'); // 在这之前想实例化任意一天需要 strtotime(),普通的date('Y-m-d', time()), 后面的参数默认是当前时间戳，time() 获取的也是当前时间戳，不能根据自定义获取时间戳
  echo $a->format('Y-m-d'); // 格式化时间
  echo $a->format('Y-m-t'); // 某个月结束日期，开始日期直接 —01-01
  ```

  ​

  todo: 时间戳转换，格式化日期，当前时间，修改当前日期，每个月结尾（30号还是31号），循环每周四，时间前后比较，

- PHP7 新特性的使用

  ```
  $a = $a ?? 9;
  $a = isset($a) ? $a : 9;
  // 上面的三元表达式在变量的判断中使用的还是比较少的，更多的是用在判断数组的key 是否存在，因为如果不存在的话是可以直接使用的，但是会给warning,这样也是不好的
  $a = $a ?: 9;
  if ($a) {
     $a = $a;
  } else {
    $a = 9;
  }
  // 这个是很久就出来的写法，三元表达式的简写

  ```

- PHP 数组操作

  array_slice() 截取数组，对于关联数组和索引数组都能截取，因为关联数组没有顺序，所以array_pop 那些数组头部尾部的操作不能生效，这时候array_slice 就有用了，比如获取排行榜前5位

- PHP 常用函数

  sprintf('%02d', 9) 对于 1 显示成01 这样很方便

  number_formate()  1234.45 显示成 1,234.45 很方便


* 感觉除了foreach 外，while 比for 好用

  ```
  $month = 5;
  $num = 5;
  $res[$month] = $num;
  $tmpM = $month;
  while ($tmpM - 1 > 0) {
      $join = 2;
      $quit = 1;
      $num = $num - $join + $quit;
      $tmpM = $tmpM - 1;
      $res[$tmpM] = $num;                
  }
  var_dump($res);exit;  // 虽然tmpM 也可以设置成临界变量            
  ```

* continue 调出本次循环， foreach 继续； break 跳出(终止)整个foreach 循环

* ​



