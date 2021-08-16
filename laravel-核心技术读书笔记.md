---
title: laravel 核心技术读书笔记
date: 2019-01-03 23:31:06
tags: [PHP,Laravel]
categories: PHP
---

关于laravel 框架核心技术一书的阅读感受
<!--more-->
###1.关于命名空间
我理解的命名空间是文件中的namespace 可以定义成任意的值，但是composer 的自动加载因为符合psr4 ，所以要求命名空间和文件路径一致，文件名和类名保持一样。当我们不适用composer的自动加载的时候，就可以随便指定文件中的namespace是多少了，比如
```
<?php
namespace test;
class test
{
}
```
可以保存在任意位置任意名称的文件中，但是不能写在composer autoload 驱动的框架中，除非改成别的方式加载，比如map
还记得我们在控制器中引用别的类吗，通过use,然后就能直接使用new classname()了，但是我们在脚本中使用的时候，我们一定要用到include 先去包含这个文件
```
<?php

namespace test;

include 'test2.php';

use test2\test5;

class test 
{
  public function test2()
  {
      test1();
  }
}


function test1()
{
    echo 1;
}


$a = new test();
$a->test2();

$b = new test5();
$b->test3();
```
可能你在想，既然我们都include 了，那为啥还要使用use，use在这个地方的作用是什么，当你使用 use test2\test5的时候，你在使用 new test5(), 它实际new 的是 test2\test5(), 如果没有 new test5(), 当前命名空间下的test5类，所以你明白了use 并不是导入包含文件，只是引入一个类的前缀，想不适用前缀，那就 new \test5() ，绝对路径吧，如果想使用当前命名空间下的 test5(),那就把外来引入的换个名称吧，比如 as。现在想想，其实我们平时同一个文件夹下面命名空间一样，本质上是写在一个文件中的，只是我们为了方便好看，才把分到多个文件中，但是我们互相引用的时候，同一个命名空间下面，不用use，因为默认就在当前文件夹下面找。

当我接触use 的时候，书中还有一句话，use只能引用类，并不能引用常量，函数··！！难道还可以这样
```
<?php

namespace test;

//include 'test2.php';

use test2\test5;

class test 
{
  public function test2()
  {
      test1();
  }
}



/* class test5
{
    public function test3()
    {
        echo 1;
    }
} */

function test1()
{
    echo 1;
}

function autoload($class) {
    echo $class;
    include './test2.php';
}

spl_autoload_register('test\autoload');

$a = new test();
$a->test2();

$b = new test5();
$b->test3();
```
对，你没看错，class 的外面还能定义function, 然后我们当前命名空间中可以直接使用，如果是不同的命名空间，需要加前缀

###2.spl_autoload_register 和 __autoload
\_\_autoload已经被废弃了，在7.2 中。\_\_autoload 如果在一个库中出现多次，会出错的，虽然是魔术方法，但可以不在类中定义，spl_autolad_register ，可以绑定多个自动加载，还可以删除，composer的自动加载机制中就用到了删除。他是类似一种方法的执行，可以写在任意地方，可以直接是方法名
```
spl_autoload_register('test\autoload');
```
可以是面向对象的方式
```
spl_autoload_register([$this, 'autoload'], true, true);
```
第二个参数true 代表异常可以捕获，__autoload 就不行，第三个参数说实话不知道咋用，队列头部和队列尾部，就是先加载哪个自动加载函数的意思呗？





