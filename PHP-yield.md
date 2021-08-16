---
title: PHP yield
date: 2018-05-25 13:55:52
tags: [PHP]
categories: PHP
---

yield，生成器(generator)，蛮重要的，各种语言中都有，比如js, 实际工作中用的比较少~~

<!--more-->

关于yield这种数据类型，还有迭代器，还有collection，都是能用foreach进行循环遍历的。

```
function productNum($length)
{
 $data = [];
  for ($i=0; $i< $length; $i++){
     $data[] = $i;
	}
 return $data;	
}

$data = productNum(100000000);
foreach ($data as $key=>$value) {
  var_dump($value);
  
}
```

用php执行上端代码会报出超出分配给php的内存错误

```
function productNum($length)
{
 //$data = [];
  for ($i=0; $i< $length; $i++){
     //$data[] = $i;
     yield $i;
	}
 //return $data;	
}

$data = productNum($100000000);
foreach ($data as $key=>$value) {
  var_dump($value);
  
} 
```

能执行成功，其实这个时候打印$data变量，可以看出来他是一个类。



每次foreach 去$data中取数据，上面的那个for循环就会执行一次，再次输出一个变量\$i。

so，如果使用yield的内容，每次也得循环使用吗 （对的，核心就是生产的时候要循环生产，消费的时候也要循环）



yield定义的内容不能有返回。



下面写一个读取文件，再往另一个文件中写入内容的例子，其实这个例子不具有实际价值，其实就是一个copy文件，用shell_exec执行linux命令就可以，或者file_get_contents,file_put_contents这样就好了，使用yield分多次，唯一可以想到的好处就是控制对内存占用的大小。

```
function getContent($file)
{
  $resource = fopen($file);
  while (feof($resource) === false ) {
         yield fgets($resource);
    }
}

$content = getContent($file);
$res = fopen('/home/itbasic/3.pdf', 'a');
foreach ($content as $c_key=>$c_value) {
    fwrite($res, $c_value);
}
fclose($res);
echo "使用: ".memory_get_usage()."B\n";  //使用: 362640B
```



用传统方式

```
$res = file_get_contents($file);
file_put_contents('/home/itbasic/1.pdf', $res);
echo "使用: ".memory_get_usage()."B\n"; //使用: 39875608B
```



显然超过一个数量级，yield就是把一个结果集分批的给你，你什么时候问他要下一个结果集，他什么时候给你。



js 版的yield 使用

```
// 产生一个连续的 id

x= 1;
function* nextId()
{
   while(true) {
      yield x;
      x++;
   }		
}

var f = nextId()
while (true){
  console.log(f.next().value)
}
```

需要注意的点蛮多的

1.function的命名

2.函数本身执行一次

3.next() 多次循环执行



PHP 版本类似代码

```
function pr()
{
    $x = 1;
    while(true) {
        yield $x;
        $x++;
    }
}

$a = pr();

for ($i=1; $i < 3; $i++) {
    var_dump($a->current());
    $a->next(); // php 的生成器的这个方法返回值是空，所以不能链式调用
}

exit;
```

PHP 的 yield 本身产生的就是一个生成器，而这个生成器自身也是可以循环的,所以上面的代码还可以改造成

```
function pr()
{
    $x = 1;
    while(true) {
        yield $x;
        $x++;
    }
}

$a = pr();

foreach ($a as $value) {
  var_dump($value); // 注意这个直接就是值，不是generator， 本身也不需要通过 next() 接口主动循环
}

exit;
```



