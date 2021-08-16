---
title: PHP函数
date: 2018-03-26 14:12:29
tags: [PHP,函数]
categories: PHP
---

工作的久了，反而基本功没有原先好了，也许要记得东西很多，内存中不够放置那些无规律的参数位置，但至少不要混淆哪些是框架自带的，哪些是PHP原生的。

<!--more-->



### String

* str_repeat（'cy', 10） //把cy这个字符串重复10次

```
var_dump(str_repeat（'cy', 10）)   //注意一定要打印，这个函数的作用只是构成一个字符串
string(20) "cycycycycycycycycycy"
```
* var_dump 和 var_export
  var_dump 经常用在我们调试的时候，可以打印出参数类型
  var_export 不能输出参数类型，比如 ['a' => 1], 只能输出这个数组，不能支出 1 是 int
  var_export 第二个参数给true, 他会把这个变量的值存储起来，可以用于赋值，相当于 printf 和 sprintf ， sprintf 会把变量格式化之后保存起来，而不会输出

  但是当变量是resource的时候，返回的总是null,但是此时var_dump 虽然也不能打印出来，但是能
```
resource(2) of type (stream) 
NULL
$a = var_export([2,3], true);
```


+ strtr  经常用在简单替换 (两种用法，是真的很方便)

  ```
  var_dump(strtr('{userna{me}', '{', 'A'));

  string(11) "AusernaAme}"
  ```

  ```
  var_dump(strtr('{username} age', ['username' => 'cy', 'age' => 18]));

  string(7) "{cy} 18"
  ```

+ compact (把两个变量组装成数组)

  ```
  $a = 5;
  $b = 6;
  var_dump(compact('a', 'b));

  array(2) {
    ["a"]=>
    int(5)
    ["b"]=>
    int(6)
  }
  ```

  ​


+ func_get_args ( 调用这个方法传入的参数)

  ```
  // 这个获取的是你实际传入的参数，和函数定义时候的参数没有什么关系
  test('www', 'ww');

  function test()
  {
    var_dump(func_get_args()); // www ww
  }
  ```

  ​


+ floor

  ```
  <?php
  echo floor(4.3);   // 4
  echo floor(9.999); // 9
  echo floor(-3.14); // -4
  ```


+ glob (返回一个包含有匹配文件／目录的数组。如果出错返回 **FALSE**。)

  ```
  <?php
  foreach (glob("*.txt") as $filename) {
      echo "$filename size " . filesize($filename) . "\n";
  }
  funclist.txt size 44686
  funcsummary.txt size 267625
  quickref.txt size 137820
  ```

+ DIRECTORY_SEPARATOR, 经常用在代码中分隔符的获取

+ rename （重命名文件）

+ basename （文件自身的名称）

+ dirname (上一层文件夹的名称)


  ​

### Array

+ array_fill (生成某种类型数组)

  ```
  var_dump(array_fill(0, 4, 'ss'));

  array(4) {
    [0]=>
    string(2) "ss"
    [1]=>
    string(2) "ss"
    [2]=>
    string(2) "ss"
    [3]=>
    string(2) "ss"
  }
  ```

  ​



1. 面向对象

is_callable()与method_exists()     //method_exists 只能判断方法是否存在，但如果比如父类的方法是protected这种，没权利执行也会返回true，这种情况就得callable了，他会返回false
```
<?php
class Foo {
    public function PublicMethod(){}
    private function PrivateMethod(){}
    public static function PublicStaticMethod(){}
    private static function PrivateStaticMethod(){}
}
$foo = new Foo();
$callbacks = array(
    array($foo, 'PublicMethod'),
    array($foo, 'PrivateMethod'),
    array($foo, 'PublicStaticMethod'),
    array($foo, 'PrivateStaticMethod'),
    array('Foo', 'PublicMethod'),
    array('Foo', 'PrivateMethod'),
    array('Foo', 'PublicStaticMethod'),
    array('Foo', 'PrivateStaticMethod'),
   );
foreach ($callbacks as $callback){
    var_dump($callback);
    var_dump(method_exists($callback[0], $callback[1]));
    var_dump(is_callable($callback));
    echo str_repeat('-', 10);
    echo '<br />';
}
```

