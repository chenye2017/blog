---
title: PHP 反射
date: 2019-03-06 10:43:51
tags: [PHP,反射,依赖注入]
categories: PHP
---

故事背景：在做文件上传的时候，对图片的上传和视频的上传走的是相同的接口，可是视频和图片有些参数是不同的，于是通过一个基类base,扩展出多个子类，image,video ···上传类，根绝参数的不同调用不同类的upload方法。

>简单点说反射的就是让你拥有剖析类、函数的能力。
>
>有的同学可能会问我剖析类有什么用，我为什么要学反射，我只能说不学反射并不会对你实现业务有任何影响，但是如果你想写出结构优雅的程序，想写出维护性和扩展性都很高的程序，学习反射是必不可少的。

Laravel 中依赖注入用的就是反射。[参考文章](http://www.dahouduan.com/2017/08/21/php-refleciton-1/)

<!--more-->

base类

```
<?php
namespace App;

class Base
{
  public $name = '';
  public function upload()
  {
  	var_dump($this->name);
  }
}
 
```

video 类

```
<?php
namespace App

class Video extends Base
{
  public $name = 'video';
 
}
```

image 类

```
<?php
namespace App

class Image extends Base
{
  public $name = 'image';
}
```

反射类

```
<?php
namespace App

class Upload
{
  public static alias()
  {
  	return [
  		'video' => \App\Video,
  		'image' => \App\Image
	]
  
	}
  public fileUpload($type)
  {
  	$classArr = self::alias();
  	$className = $classArr[$type];
  	$reflection = new ReflectionClass($className); // 获取反射对象
  	return $ref->newInstance() // 获取类实例化对象
  }
 
}
```

上面代码只是一段很简单的使用反射实现的代码，其中对实例化参数的判断，类是否能实例化等诸多反射的方法都未使用到。

总结一下我在使用easyswoole 写代码过程中的感觉：

- 首先想熟练的使用easyswoole,必须得熟悉swoole,比如对swoole http server 的配置，举个例子，当上传文件大于easyswoole 默认上传文件总的大小的时候，easyswoole(包括swoole)的tcp 链接会主动断掉，但是swoole的服务器端会打印一个warning ,显示传输文件过大，但是easyswoole 不会，服务器端不报异常，只是客户端建立不了链接，这就很尴尬了，之前还以为是端口号或者自己代码哪里出了问题。（ps:php文件上传和两个参数有关，一个是php.ini 里面的upload size, 一个是post size, 如果是swoole 上传的话，还和swoole 的配置项有关，在easyswoole 中对http server 的配置和swoole 中对http server 配置一样，只是参数修改的位置不一样罢了）

- 再者是对easy swoole 的使用，之前习惯于查看文档或者手册去学习（比如laravel 和 php 手册），对其中的所有的方法都有详细的解释，我们应该学会通过查看源码去学习其中方法的使用，比如easy swoole 中对获取参数这些都没有doc 文档，我们可以在request 类下面查看有哪些可以调用的方法，或者是通过编辑器补全

- 说到编辑器补全，之前发现的一个问题就是比如上传类，当可能返回上传类或者null 的时候，通过链式调用，上传类的方法得不到补全，可是我们通过php7的返回结果限制，就能让编辑器自动补全了

  ```
  <?php
  class Request
  {
     public function uploadFile($name) :?UploadFile
      {
    		if ($file[$name]) {
            return $file[$name]
          } else {
           return null
            }
    
      }
    
  }
  // ?代表可能是Null
  ```

- 当我们想知道这个结果有哪些可以调用的方法的时候，我们可以直接打印这个结果，查看属于哪个类，然后我们到这个类下面查找，比如我之前想知道反射的 getParamtes  方法返回结果的foreach 中的value 有哪些可以调用的方法的时候，查看这个value 的类，然后查看这个类有哪些方法，比如getClass()->name, 就是获取参数提示符（这个参数提示符还是蛮奇怪的，Bag 这种自定义的类可以提示， 但是 string int 这种类却不提示）

- 今天在查看easyswoole源码的时候,发现自己对public protected private 都忘记了,首先public 是最简单的，protected 本类和继承类中可以使用，private只有本类中可以使用

  ```
  <?php

   class A {

      public function __construct()
      {
          
      }

      public function test1()
      {
          echo 1;
      }

      protected function test2()
      {
          echo 2;
      }

      private function test3()
      {
          echo 3;
      }

  }

  class B extends A {
      public function test4()
      {
          $this->test2();
      }
  }

  $b = new B();

  $b->test1();

  $b->test4();

  $b->test2();

  //$b->test3();

  ```

  ​


下面[文章来源](http://www.dahouduan.com)

PHP 内置了一组反射类来实现类的反射，常用的有：

- ReflectionClass 解析类
- ReflectionProperty 类的属性的相关信息
- ReflectionMethod 类方法的有关信息
- ReflectionParameter 取回了函数或方法参数的相关信息

想看全的就翻手册去。

今天先通过一段演示代码简单看下php的反射到底是个什么东西。

例子1：

```
<?php

class Hero {

    protected $name;

    protected $skills = [];

    public function __construct($name, $skills = []) {
        $this->name = $name;
        $this->skills = $skills;
    }

    public function attack($hero) {
        echo "Attack {$hero->name}" . PHP_EOL;
    }

    public function execute($index) {
        echo "Axecute {$index} skill" . PHP_EOL;
    }
}

$ref = new ReflectionClass('Hero'); // 参数类名

if ($ref->isInstantiable()) {
    echo '可以实例化' . PHP_EOL; // 判断是否可以实例化，比如单例或者静态类的存在
}

// 获取类的构造函数
$constructor = $ref->getConstructor();
print_r($constructor); //ReflectionMethod  E对象

//获取属性
if ($ref->hasProperty('name')) {
    $attr = $ref->getProperty('name');
    print_r($attr); //ReflectionProperty  对象
}

// 获取属性列表
$attributes = $ref->getProperties();
foreach ($attributes as $row) {
    //row 为 ReflectionProperty 的实例
    echo $row->getName() . "\n";
}

// 获取方法
if ($ref->hasMethod('attack')) {
    $method = $ref->getMethod('attack');
    //$method 为 ReflectionMethod 的实例
    print_r($method);
}

// 获取方法列表
$methods = $ref->getMethods();
foreach ($methods as $row) {
    //这的row 是 ReflectionMethod 的实例
    echo $row->getName() . PHP_EOL;
}
```

想知道上面代码的用途，最好的办法就是执行一下，自己打印一遍(如果用xdebug 对这种脚本式的执行是再方便不过了)

上面例子想表达的效果就是：仅仅通过类名，就能了解类的内部结构，进而去实例化类

例子2：

student

```
<?php

class Bag{

    public function name(){
        return  "学生包".PHP_EOL;
    }
}

class Student
{
    public $id;

    public $name;
    public function __construct($id,$name,Bag $bag)
    {
        $this->id = $id;
        $this->name = $name;
    }
    public function study()
    {
        echo $this->name.' is learning.....'.PHP_EOL;
    }

    public function showBag(){
        echo "My bag have ".$this->bag->all();
    }
}

```

run

```
<?php 

require 'student.php';
function make($class, $vars = []) {
    $ref = new ReflectionClass($class);

    if(!$ref->isInstantiable()) {
        throw new Exception("类{$class} 不存在"); // 感觉这块应该是累不可以实例化
    }

    $constructor = $ref->getConstructor();
    if(is_null($constructor)) {
        return new $class;
    }

    $params = $constructor->getParameters();
    $resolveParams = [];
    foreach ($params as $key=>$value) {
        $name = $value->getName();
        if(isset($vars[$name])) {
            $resolveParams[] = $vars[$name];
        } else {
            $default = $value->isDefaultValueAvailable() ? $value->getDefaultValue() : null;
            if(is_null($default)) {
           
                if($value->getClass()) {
                // 依赖注入的实现，以这个为例子
                // 需要Bag $bag ，却没有传入 bag，自动实例化
                // 如果需要参数，直接在vars 里面加，但感觉参数名称不能和原本的冲突
                    $resolveParams[] = make($value->getClass()->getName(), $vars);
                } else {
                    throw new Exception("{$name} 没有传值且没有默认值。");
                }
            } else {
                $resolveParams[] = $default;
            }
        }
    }

    return $ref->newInstanceArgs($resolveParams);
}

// 没有提供name 值
try {
    $stu = make('Student', ['id' => 1]);
    print_r($stu);
    $stu->study();
} catch (Exception $e) {
    echo $e->getMessage();
}

// 提供name 值
try {
    $stu = make('Student', ['id' => 1, 'name' => 'li']);
    print_r($stu);
    $stu->study();
} catch (Exception $e) {
    echo $e->getMessage();
}

// 需要 Bag ，却没有传入
try {
    $stu = make('Student', ['id' => 1, 'name' => 'li']);
    print_r($stu);
    $stu->study();
    $stu->showBag();
} catch (Exception $e) {
    echo $e->getMessage();
}
```

> 可以看到构造函数的第三个参数 `$bag` ,被自动实例化了，然后传递给了 `Student` 类的构造函数，这个部分很关键，这个地方可以用来实现依赖注入，我们不必在手动实例化对象了，我们可以根据参数的对应的类来自动实例化对象，从而实现类之间的解耦。
>
> // 感觉这个解耦就是不用手动去传参数，避免类修改的时候自己大片代码需要修改

例子3

```
<?php

// 这个不能算是容器，容器是反射的一种高级应用，下面的只是反射的一种基础应用
if (PHP_SAPI != 'cli') {
    exit('Please run it in terminal!');
}
if ($argc < 3) {
    exit('At least 2 arguments needed!');
}
$controller = ucfirst($argv[1]) . 'Controller';
$action = 'action' . ucfirst($argv[2]);
// 检查类是否存在
if (!class_exists($controller)) {
    exit("Class $controller does not existed!");
}
// 获取类的反射
$reflector = new ReflectionClass($controller);
// 检查方法是否存在
if (!$reflector->hasMethod($action)) {
    exit("Method $action does not existed!");
}
// 取类的构造函数
$constructor = $reflector->getConstructor();
// 取构造函数的参数
$parameters = $constructor->getParameters();
// 遍历参数
foreach ($parameters as $key => $parameter) {
    // 获取参数声明的类
    $injector = new ReflectionClass($parameter->getClass()->name);
    // 实例化参数声明类并填入参数列表
    $parameters[$key] = $injector->newInstance();
}
// 使用参数列表实例 controller 类
$instance = $reflector->newInstanceArgs($parameters);
// 执行
$instance->$action();
class HelloController
{
    private $model;
    public function __construct(TestModel $model)
    {
        $this->model = $model;
    }
    public function actionWorld()
    {
        echo $this->model->property, PHP_EOL;
    }
}
class TestModel
{
    public $property = 'property';
}
```

>（以上代码非原创）将以上代码保存为 `run.php`
>
>（以上代码非原创）将以上代码保存为 `run.php`
>运行方式，在终端下执行`php run.php Hello World`
>
>可以看到，我们要执行 `HelloController` 下的 `WorldAction`,
>
>可以看到，我们要执行 `HelloController` 下的 `WorldAction`,
>`HelloController` 的构造函数需要一个 `TestModel`类型的对象,
>
>通过php 反射，我们实现了, `TestModel` 对象的自动注入，
>
>上面的例子类似于一个请求分发的过程,是路由请求的分发的一部分，假如我们要接收一个请求 地址例如： `/Hello/World`
>
>意思是要执行 `HelloController` 下的 `WorldAction` 方法。

上面的例子有个最大的缺陷就是所有类写在一个文件中，导致没有使用bind 也不会出错 （注意container 中没有自动加载）



发现：感觉依赖注入的类的参数都是空的，都是直接就能实例化的那种

感觉上面那个发现并不多

控制反转：就是通过一个container 容器去解决依赖关系，把类的实例化从类的内部改成从外部传入

```

class Person
{
    public function buy($obj)
    {
       // $obj = new Car(); 改成依赖注入的模式
        $obj->pay();
    }
}

```

但是这个obj 我们不希望手动传入，希望可以帮我们自动生成，于是就出现了依赖注入（通过类的反射实例化类，但这个类我们要提前绑定到容器中，要不然可能找不到，我们的容器可没有自动加载机制，容器一般有两个方法bind 和 make, bind 一般是 一个类的标识 和一个匿名函数，make 实例化这个类的标识，执行之前绑定的匿名函数，这就是最简单的container）



能解决的问题：

当我们把依赖类通过注入的方式传入的时候，我们可以只是传入这个类的类型（接口），当我们使用的时候可以在这个容器中为这个类型不同的实体类，达到switch 切换的效果













