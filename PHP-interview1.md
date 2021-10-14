---
title: PHP面试题
date: 2019-02-21 10:50:35
tags: [PHP,interview]
categories: PHP
---

收集一些总是考的面试题

<!--more-->

# Web

```
怎么会导致跨域，为什么会出现跨域
schema，或者ip地址，或者端口号 不同都会出现跨域。为了防止前端随意调用别人的后端接口，比如获取别人的某个邮箱文件内容（如果该邮箱网站用的cookie 存储登陆信息，而前端发送请求的时候会自动携带cookie 信息，如果不加以限制，cookie的使用会遭到恶意的攻击，csrf 攻击就是基于这一点，为了防止，让前端主动添加token 作为登陆凭证，调用后端接口）

怎么解决跨域
jsonp(用的比较少,好像是通过拼凑返回内容,让前端主动调用某个申明好的js文件), cors(通过在返回头中添加header信息,比如允许的域名,允许的调用方法，允许的自定义header参数，是否允许携带cookie (按理说cookie 应该在header 参数之中，但因为特殊，所以单独拿了出来))。

什么是简单请求，什么是非简单请求
好像是除了get post 之外的跨域请求都属于非简单请求，除此之外携带额外的参数也属于非简单请求。在发送非简单请求之前都会发送一个options请求，options请求没有请求体内容，你只需要把预定的header信息返回就好了，如果返回的header请求信息做了跨域处理，那么后面浏览器就不会屏蔽掉返回内容
```
```
get post 区别
get 可以浏览器收藏，参数会被缓存，post 不可以 ，
get 参数只能被url 编码， post 支持多种方式编码 （比如 application/x-www-form-urlencoded， 比如json 格式）
get 长度一般由浏览器决定，比较短 ， post 比较长
get 传递的参数一般直接显示在url 地址栏， 相比较的话，post会比较安全，get不要放敏感信息， get 参数放在url 中，post参数放在request body 中
```
```
正则
\d 数字
\w 字符，字母数字下划线
. * ? 组合匹配所有
* 匹配0个或者很多歌
? 匹配0个或者1个
+ 匹配1个或者多个

匹配电话号码
preg_match_all('/1[23456]\d{9}/'， 15618014759, $res1);
匹配邮箱
preg_match_all('/[a-zA-Z0-9]+[-_.]?[a-zA-Z]@[A-Za-z0-9]+\.[a-zA-Z]+/', '1967196678@qq.com', $res2)
```

```
http请求过程
浏览器开始在host文件中查找对应网址的ip地址，找不到的话再在本地浏览器缓存中查找，再找不到去dns解析中查找，找到服务器ip地址之后，一般会先去监听的web服务器上，比如nginx，nginx对于静态文件直接返回，对于php类似文件有转发作用，比如转发给9000端口的php-fpm, 或者swoole， 然后他们再交给单个的工作进程中的php处理

为什么http请求需要三次握手4次挥手
第一次握手，验证cli 可以发消息到 server， server确定cli 可以推送消息，
第二次握手，server验证自己可以推送消息，cli知道自己能推送消息并能收到返回消息，此时已经可以发送消息
第三次握手，server知道自己可以同送消息，也能接收到消息，于是握手结束

第一次挥手，通知sever 我要关闭了
第二次挥手，server 通知cli， 好的，你关闭吧
第三次挥手，server通知cli
第四次挥手，cli告知server， 收到
（通过netstat -antp 可以查看tcp 连接（第一个ip端口代表host机器，第二个代表客户端），最好是在同一台服务器上客户端curl ，就能看到两条连接）
（tcpdump 可以看到抓包过程）
```
![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20190901201717.png)

```
常用的http状态码
200 成功
300 重定向， 301 永久重定向，302 临时重定向， 304 未修改
400 客户端错误，401 未授权（可能是未登录）， 403 禁止访问（没有权限），404 not found 
500 服务端错误，502 网关挂掉了, 504 超时
```

```
https 和 http的区别
https基于http， 有个ssl 包裹住用户信息
https加密原理：因为非对称加解密很消耗cpu资源，所以利用对称加解密。怎么传输这个共同的密钥呢，肯定不能直接传输，一般是服务端生成一个公钥和私钥，公钥传输给客户端，客户端拿着公钥加密一个字符，服务端私钥解密这个字符，得到这个传输过程中加密数据的对称密钥，为了防止传输给客户端的公钥被修改。通过拉入第三方机构，第三方机构对（服务端信息 + 服务端产生的公钥） hash 成一个字符串 （相当于签名），然后利用第三方机构的私钥进行加密生成签名，因为不知道ca 的私钥，所以这个hash 不可能被改变，得到hash 值之后，对于证书中包含的 （服务端信息 + 服务端产生的密钥） + hash 算法 是否等于 hash 值进行比对，如果不一致说明公钥被修改了。
简单点：
1. 对称加密传输数据，需要获得一个别人获取不到的对称密钥？
2. 通过非对称加密，由客户端的公钥加密字符，服务端私钥解密字符，这个字符就是对称密钥。客户端怎么获取一个非对称的公钥？
3。 ca 证书包含公钥 + ca 私钥 生成的签名，无法修改， ca公钥（每个浏览器或者os 都已经包含了）解密获取签名前的hash， 验证证书内容是否改变，也就是保证公钥的正确性。
```

```
什么是socket编程
Socket 又称网络套接字，是一种操作系统提供的进程间通信机制。

工作流程：
1. 服务端先用 socket 函数来建立一个套接字，bind 端口号，再调用 listen 函数，使服务端的这个端口和 IP 处于监听状态，等待客户端的连接
2. 客户端用 socket 函数建立一个套接字，设定远程 IP 和端口，并调用 connect 函数
3. 服务端用 accept 函数来接受远程计算机的连接，建立起与客户端之间的通信
4. accept 只做请求的接受完了，直接投递给master 进程让他处理
4. 完成通信以后，最后使用 close 函数关闭 socket 连接。

这面即使 epoll 多路复用技术

感觉类似swoole, swoole 先开启监听，new 一个server
然后写监听事件，用于cli的请求的接受
请求处理完，关闭
```

```
OAuth(Open Authorization) 协议为用户资源的授权提供了一个安全的、开放而又简易的标准，第三方无需使用用户的用户名与密码，就可以申请获得该用户资源的授权。

运行流程：

1. 用户打开客户端以后，客户端要求用户给予授权。
2. 用户同意给予客户端授权
3. 客户端使用上一步获得的授权，向认证服务器申请令牌。
4. 认证服务器对客户端进行认证以后，确认无误，同意发放令牌。
5. 客户端使用令牌，向资源服务器申请获取资源。
6. 资源服务器确认令牌无误，同意向客户端开放资源

OAuth 2.0 定义了四种授权方式，授权码模式、简化模式、密码模式、客户端模式，具体的授权流程，请看阮一峰老师的文章理解OAuth 2.0。
```

```
csrf 攻击
防止因为浏览器请求本域名接口自动携带着cookie， 在第三方者在用户没有感知的情况下，调用api。解决办法是携带参数token
xss 攻击
用户输入信息没有做合理过滤，导致让前端页面调用非法的js等内容。用户信息过滤 
sql 注入
用户输入信息不合法, 导致sql执行用户命令。 用pdo，sql语句预处理功能，或者用户输入过滤。
```



# PHP

```
PHP 位运算符 
| 半个或
& 同或
^ 异或
~ 取反 （其实最常见的就是error_reporting(E_ALL ~E_NOTICE)）
3 | 5 7
3 & 5 1
```

```
字符串比较
'111aa' == 111  // true
```

```
setcookie(xxx);
var_dump($_COOKIE);  // 空值，本次还没有携带cookie
```

```
常用string ,arr 函数
array:
array_merge
array_slice
array_values
array_keys
array_columns
is_array
array_key_exists
array_change_key_case
array_diff
array_intersect
array_fill
array_sum
array_map
array_reduce
array_reverse
key
current
next


string:
explode
substr
strpos
preg_split
preg_match
strstr
sort
rsort
asort
arsort
ksort
krsort
usrot
```
```
array_merge 和 +  的区别
+ 同key, 前面覆盖后面 （index 或者 关联）
merge 关联后面覆盖前面，数字索引直接扩展开了 // $a = [1 =>2]; $v = [1 => 3]; array_merge($a,$v); 输出 [2, 3]; 重新排序

在写代码的时候，如果我们确定key 不一样，可以考虑下 +， 因为merge 会把数字索引重新排序，这个可能是我们不想看到的
```

```
php7新特性

1.throwable 分成 error 和 exception，都可以被捕获
2.太空舱运算符， 一般usort 里面排序
3.三目运算符的缩写， ??
4.define可以定义常量数组 (以前好像只有const 可以，const) //  const 和 define 的区别，但感觉实际还是define 用的比较多 https://www.jianshu.com/p/a38b81433183
5.返回值也能严格要求返回类型
6.传入参数类型声明的扩展，比如int
7.命名空间合并
8.closure call 替代bindTo 更加的方便
```

```
php7为什么相较于php5性能有很大提升？
减少内存分配次数，优化底层zval 数据结构(64 位减少到24 位)，字符串解析的优化，缓存hash
```

```
php垃圾回收机制
1、以php的引用计数机制为基础（php5.3以前只有该机制）（refcount 是1 的时候就删除， is_ref 是否是引用类型）
2、同时使用根缓冲区机制，当php发现有存在循环引用的zval时，就会把其投入到根缓冲区，当根缓冲区达到配置文件中的指定数量后，就会进行垃圾回收，以此解决循环引用导致的内存泄漏问题（php5.3开始引入该机制）（为了防止子变量中引用父变量，父变量unset 之后refcount 到不了0 一直得不到回收）
其实最简单的防止那种慢性内存溢出的方式就是php-fpm 的连接重启(当然你要是一次性消耗了太多内存，还是会内存溢出的)
```

```
如何解决php内存溢出问题（phpexcel中经常出现，第二个很管用）
1. 增大 PHP 脚本的内存分配（这个不能一劳永逸）
2. 变量引用之后及时销毁
3. 将数据分批处理
```

```
$z = 0.58;
var_dump(intval($z * 100));
```

答案是 57，所有语言对于小数的存取都是不精确的， 0.58其实是0.57999999，而且intval() 总是从遇到第一个不是数字开始截取，导致了0.57，比较通用的方法是转成别用intval, 或者用数学函数，bcmath扩展计算

```
cookie 和 session 的 区别
cookie 更像一个钥匙，因为没有加密，所以客户端可以查看和随意更改cookie的数据，cookie 对应的 session 才是服务端真正存储的序列化数据，
php 中 cookie 默认名称是 phpsessid , 通过这个值，服务端php能识别客户端对应的用户

什么是jwt， jwt 和 cookie, session 的区别
jwt 是一种token 的实现方式，由base64 处理过，因为base64可以放解码，所以不要存储重要信息，存放一些基本的辨识用户的信息就好了。
他是三段组成的，第一段是签名的加密算法，第二段是用户数据（载体）, 第三段是前面数据的签名，防止用户数据被修改。
jwt相较于cookie 和session ,不需要服务端存储用户的基本信息，token下发到client ,完全由客户端保存, 可以防止csrf 攻击。但是jwt 下发之后除非做特殊处理比如服务端，否则不能失效，蛮尴尬的。
```
```
spl_autoload_register和 _autoload 的区别
现在的自动加载已经不用_autoload 了，这个只支持一个方法，现在一般用spl_autoload_register, 或者全部用composer， 但composer本质上还是spl_autload_register,他可以支持多个函数
```


```
判断一个日期是否合法
// 思路
把一个日期转用strtotime 转换成时间戳再通过 date(), 转换成标准时间，和原先的日期比对是否相同，相同就合法，不同不合法
// 注意：错误的时间date 也能转换，只是可能转换成1970 这样的时间
// 字符串的比对比较耗时
```

```
单引号，双引号区别
单引号解析变量
双引号不解析变量
```
```
include require 区别
include 包含文件不存在 warning
require 错误
required_once
```
```
中文截取字符串
mb_substr
```


```
平时开发用的设计模式

单例  保证在整个应用程序的生命周期中，任何一个时刻，单例类的实例都只存在一个，同时这个类还必须提供一个访问该类的全局访问点。（getInstance）

工厂 定义一个创建对象的接口，但是让子类去实例化具体类。工厂方法模式让类的实例化延迟到子类中。tp中对于config文件的不同处理，有三个驱动比如 ini,yaml, php, 当我们想读取ini文件的时候，factory 方法中传入的是ini,然后拼凑命名空间，交给 Ini 类处理，方便扩展。还有日志的驱动，用文件还是 socket，cookie 的驱动用文件还是redis

注册树  用一个属性保存各个类的实例，需要的时候就从这个属性中取，容器用的就是这种思想

门面  方便静态调用. 实际调用的是别的类（感觉主要是这些方法中不需要实例，所以可以静态调用）（如果是多进程，对于那些连接的操作，感觉通过门面不行，因为单个连接实例可能不能用了，得还连接池中别的实例）

依赖注入 通过对类中需要以来的对象，直接以参数的形式传入（需要携带参数类型，也就是类名），让容器通过反射自己去实例化，解耦了各个类的依赖，下次以来的类修改的时候，不用我们修改生成依赖类的代码

生产者消费者 异步任务或者花费时间多的任务放入队列中,让消费者消费

订阅发布 主体对象状态发生改变,与之关联的观察者对象会收到通知，并进行相应操作

装饰模式（laravel 的路由，核心 array_reduce）
```

```
常用的魔术方法
__construct  类初始化
__destruct  对象销毁的时候
__call  调用的方法不能访问 （redis 类封装的时候很好用）
__callStatic  调用的静态方法不能访问
__get  获取的属性不能访问
__set  设置的属性不能访问
__isset  isset empty 不能访问的属性
__unset unset 不能访问的属性
__sleep  序列化一个对象的时候
__wakeup  反序列一个对象的时候
__clone 对象（作为两个实体的存在)
__invoke 调用类实例像调用方法那样自动触发，像closure类自带这个方法，php中所有的匿名函数默认都是closure的实例，所以我们执行保存匿名函数的变量自动执行了这个方法

魔术变量
__CLASS__  类名
__LINE__  行号
__FILE__  文件名
__METHOD__ 类中方法名
__FUNCTION__ 普通方法名
```

```
php 错误信息控制
error_reporting(E_ALL)
```



```
trait Singleton {}
多继承的实现

class Db
{
  use Singleton;
}
```

```
抽象类只能继承， extend ，接口可以实现多个，比如我想我这个类拥有foreach, 之类的功能， implement

abstract 抽象类 （基本很少用）
抽象类不能实例化
抽象类中方法可以有方法体，没有方法体的只能定义abstract

abstract 介于 class 和 interface 之间

interface 接口
interface 中不能用变量,但是可以有常量
interface 中方法的访问权限只能是public, 不能是protected或者prviate, 方法不能有方法体，


抽象类和接口中但凡没有方法体，子类必须去实现
```

```
static ,延迟静态绑定，
self, parent 调用的方法的时候都是以当前类为标准,
static 根据实际调用对象，每当到了static 的 时候调用的上一个确定了调用类的对象方法 (parent self 都不能改变调用对象)（优先调用自己的，如果自己的不能调用，调用父类的）（注意延迟调用的一定要是静态的static）


class A {

    public static function who()
    {
        echo __CLASS__;
    }
    
    public static function test()
    {
        static::who();
    }

}

class B extends A{
    protected static function who()
    {
        echo __CLASS__;
    }
}

B::test(); // 调用不了， 在 a 中调用b的protected


class A {
    public static function foo()
    {
        static::who();
    }
    
    public static function who()
    {
        echo __CLASS__;
    }
}

class B extends A {
    public static function test()
    {
        A::foo();
        parent::foo();
        self::foo();
    }
    
    public static function who()
    {
        echo __CLASS__;
    }
}

class C extends B {
    public static function who() {
        echo __CLASS__;
    }
}

C::test();

A C C
```

```
防止sql 注入
sql 注入本质上就是害怕用户拼接sql, 在sql后面携带着用户传入的内容

select name from user limit 10 offset . $offset;

$offset = ' 0; select * from user';

上述代码在pgsql 扩展中可以直接执行， 索然mysqli 中执行不了，

handle:
最好的方法就是利用预处理语句（那种 绑定??， 不能单纯的理解成嵌套变量进去，如果真的那样那还是防止不了sql注入）

参数类型限定
我们平时可以通过 转义 str_replace 之类的，但感觉这样很容易误伤，所以干脆还是用pdo的预处理语句吧
```

```
红包，我能想到比较简单的方法
存在一定可能性死循环，就是后面数字的和已经不够一人一个了，已解决，每次计算剩余的最小值至少是每个1分，如果少于，则重新计算
// mt_rand 只能产生随机整数，对于现实生活中的分，可以通过扩大100倍来解决
// mt_getrandmx() 那个例子产生随机浮点数也是同样的道理
// arr 是目前产生的数组
// num 是剩余的值
function productRand($arr, $num, $leave)
{
    $tmp = mt_rand(0, $num);
    $leveTotal = $num - $tmp; // 剩余的钱不能少于每个红包1分钱
    if ($tmp == 0 || $tmp == $num || $leaveTotal < $leave * 1) {
        return productRand($arr, $num);
    } else {
        $arr[] = $tmp;
        $num -= $tmp;
        return [$arr, $num];
    }
}

// $total 代表总数量
// $count 代表红包个数
function hongbao($total, $count)
{
    $num = $num * 100;
    $arr = [];
    $top = $num;
    for ($i = 0; $i < $cishu - 1; $i++) {
    	$leave = $count - $i - 1; // 剩余红包个数
        $res = productRand($arr, $top, $leave);
        $arr = $res[0];
        $top = $res[1]; 
    }
    $arr[] = $num - array_sum($arr);
    return $arr;
}

var_dump(hongbao(11, 4));

exit;
```

```
同步，异步区别 阻塞非阻塞区别
同步，异步主要是获取返回值内容的时候，同步如果一次性不能获取返回内容，需要不断的轮训获取返回内容，异步返回内容会主动通知（swoole task异步任务， php-fpm模式下同步代码）
阻塞和非阻塞主要是进程方面，阻塞api所在进程会被系统直接挂起，直到完成（sleep， io比如redis mysqli pdo 等），非阻塞的时候所在进程不受影响，可以做别的事情（array, string 函数）
```

```
ab 测试
-n  发送的请求总数
-c  并发量
ab -n 5000 -c 200 http://www.baidu.com/ (注意末尾的斜线)
Requests per second  每秒钟完成的请求数量
Time per request  客户端等待时间
Time per request 服务端等待时间
```

```
yield  感觉就类似分页，有效减少内存的消耗（大数据量一次性读入内存吃不消），注意yield 的使用和数组还是有区别的.
就相当于我们值读入部分数据到yield 中，我们还得foreach yield 抛出的数据（iterator 实现了迭代器，所以可以循环），循环去插入 取出yield 中的数据

function test() {
  for($i = 1; $i<5; $i++) {
      yield $i;
  }
}
$data = test();
foreach ($data as $value) {
    var_dump($data->current(),$value);
} 
exit;
```

```
自带的接口
iterator // 可以循环，下面的那些方法都是为了foreach能正确数据准备的
class test implements Iterator
{
    public $arr = [];
    public $keyArr = [];
    public $keyP = 0;
    
    public function __construct($arr)
    {
        $this->arr = $arr;
        $this->keyArr = array_keys($arr);
    }
    
    public function rewind()
    {
        // TODO: Implement rewind() method.
        reset($this->arr);
        $this->keyP = 0;
    }
    
    public function current()
    {
        // TODO: Implement current() method.
        return pos($this->arr);
    }
    
    public function next()
    {
        // TODO: Implement next() method.
        $this->keyP++;
        return next($this->arr);
    }
    
    public function key()
    {
        // TODO: Implement key() method.
        return $this->keyArr[$this->keyP];
    }
    
    public function valid()
    {
        // TODO: Implement valid() method.
        return isset($this->keyArr[$this->keyP]);
    }
}
```

```
php 进程间通信的方式

第三方，比如redis
消息队列， 也可以属于第三方
共享内存， 不如swoole_table
socket 文件
管道 （应该类似 grep ）
信号  字进程结束给父进程结束信号，请求回收
tcp
```

```
几个预定义接口 spl标准类库
ArrayAccess , 让类可以类似数组方式访问
Countab  , 让类可以被 count()
iterator  让对象能循环，几个必须实现的方法（current, 当前key, key, 就是 key 的名称， next 往下走， rewind, 重置，valid 是否有效，本身php 的数组就包含上述的实现，$arr[$key], key,next, reset, isset）
iteratorAggregate 感觉就是方便的创造一个迭代器比较方便（官网上直接调用spl 的一个标准库, ArrayIterator(这是类，不是预定义接口), 数组对象转成 Iterator 方便）
```

```
php 多进程写入同一个文件
file_put_contents() // 添加第三个参数ex_lock

对文件加锁，并对锁设置超时时间（类似给redis 那种锁 sex 锁， setnx 不能同时设置锁的时间，不具有原子性，这里面的锁，其实就是一个key-value）

$resource = fopen('./tmp.log');
$try = 0;
while(!flock($resource, LOCK_EX) && $try != 100)
 {
	$try++;
} 
if ($try == 100) {
  return false; // 测试太多失败
}


fwrite($res, 'ceshi');
fclose($res); // 一般就能删除锁， 或者 flock($resource, UN_LOCK);
return true;

关于php这种锁，进程结束了就自动消失了，文档上说是脚本结束了，其实是一个意思（想一下我们用redis实现的锁，也是这样的，redis要是挂了，锁是不是没了）
（当我用用了排它锁，我们再打开文件就是空的）
(感觉写文件也不是直接往磁盘文件中写，而是写入一个缓存中，之后合并，因为当我用排它锁锁定一个文件的时候，fwrite 能立刻写入成功，虽然此时文件还没有修改)
```





# Laravel

```
laravel 调优
1. 开启Opcache
   2.关闭debug
   3.缓存配置 php artisan config:cache
   4.缓存路由 php artisan router:cache
   5.类映射加载优化 php artisan optimize
   6.根据需要只加载必要的中间件
```

```
laravel 生命周期
```

```
loc ,facade, contract, 服务提供者是什么
```

```
laravel 和 其他框架的区别
```






# Mysql

```
mysql 性能优化
1. 建立索引，优化索引 (最左前缀，覆盖索引，联合索引)
2. 读写分离 （怎么保证一致性）（先删缓存，再插入数据，容易读到旧的内容，因为插入数据比较慢，中间容易被介入，可以插入数据之后，暂停几ms 删除缓存，这个ms 是根绝自身写入缓存的逻辑时间）（或者先插入数据，再删除缓存，因为插入数据比较慢，所以插入数据和删除缓存之间很难有空隙介入）
3. 保证单表数据量，分库分表
```
```
mysql 乐观锁，悲观锁，共享锁，排它锁，行锁，表锁
乐观锁自己实现：每条数据后面都有个version, 读取version的内容，更新的时候观察这个version 是否改变，可以通过where 中加version 条件，如果version 改变了，则修改失败，如果version 没改变，则修改成功
悲观锁 ，数据库自带。悲观锁又分为共享锁 lock in share mode 和排它锁 for update。共享锁锁住，不能更新，只能读取。排他锁锁住，也是不能更新，但是能读取
行锁：innodb 引擎
表锁： myisam 引擎
```
```
mysql 的隔离级别
读未提交
读已提交
可重复读
可串行化
默认的可重复读。
并发事务中容易出现的问题，脏写(当前事务中写的内容被另一个事务回滚了)，脏读（读取的内容是一个未提交事务中内容），不可重复读（多次读取获取的内容不一致），幻读（后读取的内容多读了之前不存在的）。
脏写大家都能防止，读未提交不能防止脏读，读已提交可以防止脏读，但不能防止不可重复读，幻读，因为他在一个事务中获取的视图可以是不一样的，但凡事务提交了，他就能获取该事物产生的新的试图，可重复读在事务过程中获取的视图始终唯一
```

```
int(1) 和 int(11)的区别
都是占用 4个字节，1 和 11 代表显示长度，除非添加 zerofill 要不然没区别
```
```
varchar 和 char 区别 (括号中是字符数)
varchar 会根据内容长度确定大小，最大长度是定义的长度 * 3,varchar 中会存储字符的长度
char 是固定大小， 
```

```
hash索引和b+索引的区别
hash 索引只能用来进行等值匹配, 也不能排序，b+索引能用来排序，可以范围取值

什么是聚簇索引
索引本质上是树，聚簇索引的叶子结点是用户记录
普通索引
普通索引的叶子结点是主键(id), 索引字段， 主键主要是为了回表，通过id 去 聚簇索引中查找用户记录
覆盖索引
因为回表属于不连续的io， 比较耗时，如果不用回表，索引字段包含select 的内容，那就会很快，这种索引就叫覆盖索引
联合索引
给多个字段建立索引，联合索引是最左原则，得在前面的字段是等值的情况下，后面的字段才能符合要求（比如前面的字段是范围索引，那么取出来的数据是无序的，在无序的情况下，联合索引第二个字段是起不到排序的作用的）
```

```
innodb 和 myisam 存储引擎的区别
innodb 是行锁，myisam表锁
innodb 支持事务，myisam不支持
innodb 数据和索引存储文件一样，myisam是分开的 
innodb 中的聚簇索引的叶子结点就是数据记录或者主键，myisam中叶子结点只是数据的存储位置，所以肯定需要回表（感觉可以和第三点合并）
```

```
什么样的字段适合建立索引 （索引建立的标准）
区分度大的字段，查找起来更容易确定具体的记录，减少回表的内容
对于字符串，需要先确定首部字符串，才能匹配类似字符串
```

```
常见的索引
主键， 唯一索引，复合索引，普通索引
```



```
mysql_fetch_row, mysql_fetch_assoc, mysql_fetch_array 区别
row 好像是数组，assoc 关联数组， array 是前面二者的结合体
```

```
limit, left join ,order by ,group by, where , select , from ,having 

select 
left join
where 
group by 
having
order by
limit

执行顺序： 先连接，再筛选，再分组，再having, 再select, 最后order by
FROM
ON
JOIN
WHERE
GROUP BY
WITH CUBE 或 WITH ROLLUP
HAVING
SELECT
DISTINCT
ORDER BY
TOP 

```





# Nginx

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20190718195208.png)

```
404   最长匹配原则，但是只能匹配到 a
500   ~* 不区分大小写， 权重比 / 高
```

```
nginx 负载均衡（属于反向代理）upstream prox_pass关键字
轮询（默认）
权重
ip_hash
url_hash

upstream  test{
   server 127.0.0.1:8081 weight=1;
   server  127.0.0.1:8082 weight=2;
}
prox_pass http://test # 还可以配置back 备胎机器
```



# Redis

```
redis 和 memecached的区别
1. 更丰富的数据结构，除了缓存，更多的使用场景
2. 单进程，只能利用单核
3. redis 可以持久化 ,memcached 重启数据会丢失
```

```
redis 怎么持久化
AOF 记录每条命令，后面重新执行一遍
RDB 类似快照，把缓存中数据都存储下来（利用back，防止始终阻塞）
```

```
redis 的常用数据类型
string  key value 形式
hash  数组，关联数组，索引数组
list  队列，
set  集合
sort set  有序集合
```

```
redis 的使用场景
点赞，利用 incr
排行榜，有序集合，分数sorce 作为排序标准
api 限流（sort set 滑动窗口）
```

```


```



# Linux

```
linux 常用命令
cd  进入
ls -la  展示
tar 解压缩
mkdir -p 新建文件夹
touch 新建文件
rm -rf  删除文件夹
vim 编辑文件
tail -f  动态查看文件变化
netstat -antp 查看tcp 连接
ps -ef  查看进程
free 查看系统资源
top  查看系统进程消耗资源
df -h 查看系统盘
alias -p 查看取别名
find / -name 查找
locate  查找
chmod  修改权限
chown  修改用户组用户
wc -l 统计行数
```

```
git rebase 变基，比如把当前分支的分叉点移动到主分支最新节点，为的就是log日志中不产生分支
git push origin master:master  推送本地master分支到远程master分支
```


```
awk 的使用
awk -F ' ' {'print $1'} ./access.log |sort|uniq|wc -l   uniq只能合并相邻的不同的，所以要先排序

```





# 算法

```
4种排序
从小到大
1. 冒泡 (核心：每次一个数据到达最终位置，第一次确定最大的数字)
   $arr = [20, 19, 88, 76, 43, 1];

function maopao($arr) {
    $count = count($arr); 
    for($i=0; $i < $count -1; $i++) { // 最后一个数不用变动，位置自动确定
        for ($j= 0; $j < $count - 1 - $i; $j++) { // 每次上浮的数字都要和没有确定的比较
            if ($arr[$j] > $arr[$j+1]) {
                list($arr[$j+1], $arr[$j]) = [$arr[$j], $arr[$j+1]];
            }
            
        }
       
    }
    return $arr;
}

2.选择排序 (和冒泡的区别，这个每次选出一个最小的数，和冒泡一样，但是不移动数据，只是最终和确定的数组比较，如果不一致，才会移动)
function xuanze1($arr)
{
    $count = count($arr);
    for($i= 0; $i < $count - 1; $i++) {
        $p = $i;
        for ($j = $i; $j < $count; $j++) {
            if ($arr[$j] < $arr[$p]) {
                $p = $j;
            }
        }
        if ($i != $p) {
            list($arr[$i], $arr[$p]) = [$arr[$p], $arr[$i]];
        }
    }
    return $arr;
}
3.插入排序 (核心，前面排好序的字段不断扩展，用来容纳新插入的数据)
function charu($arr)
{
    $count = count($arr);
    for ($i=1; $i < $count; $i++) {
       
        if ($arr[$i-1] > $arr[$i]) {
            $tmp = $arr[$i];
            for($j=$i-1; $j>=0 && $arr[$j] > $tmp;$j--) {
                $arr[$j+1] = $arr[$j];
            }
            $arr[$j+1] = $tmp;
        }
       // var_dump($arr);
    }
    return $arr;
}
1. 快排：以一个数为基准，分原数组成两列，递归调用
   function quick($arr)
   {
    if (!$arr) {
        return [];
    }

    $tmp = $arr[0];
    $left = $right = [];
    $count = count($arr);

    if ($count > 1) {
        for ($i = 1; $i < $count; $i++) {
            if ($arr[$i] <= $tmp) {
                $left[] = $arr[$i];
            } else {
                $right[] = $arr[$i];
            }
        }
        return array_merge(quick($left), [$tmp], quick($right));
    } else {
        return $arr;
    }
   }
```

```
查找 （注意查找的算法都是在排好序的基础上）
1.二分查找
function search1($arr, $num, $min, $max) {

    if ($min == $max && $num != $arr[$min]) {
        return false;
    }
    
    $tmp = ceil(($min + $max) / 2);
    if ($num == $arr[$tmp]) {
        return $tmp;
    } else {
        if ($num < $arr[$tmp]) {
            $max = $tmp-1;
        } else {
            $min = $tmp + 1;
        }
       return search1($arr, $num, $min, $max);
    } 

}
```

```
1000000个数字的整数， 大小是0 到 999999， 找出其中重复的数字.

1. 最简单的。先排序，排序玩比较相邻数字是否相同，相同就添加进结果集。（二分 nlogn）
2. 利用hash, 时间复杂度n, 空间复杂度n. 我们知道php 的数组就是一个hash, 我们可以遍历这个大的原始数据，然后把其中的数字都往一个预先的hash 里面塞，如果之前有值就代表重复.
3. 因为大小是0 到 99999， 所以我们可以foreach ,然后把对应数据换算到我们要的位置上来，如果最后没有调回来，代表这个数字重复

$arr = [2,3,1,0,2,5,3];
// 转变过程
// [1,3,2,0,2,5,3]
// [3,1,2,0,2,5,3]
// [0,1,2,3,2,5,3]
// 2, 3 入结果集
// foreach 的时候，如果 &value,可以改变这次遍历的内容，单单$arr[$key] 只能改变下次遍历时候的$arr
 function chongfu($arr)
{
    $chongfu = [];
    foreach ($arr as $key => &$value) {
        if ($value == $key) {
            continue;
        } else {
            jiaohuan($arr, $chongfu, $key);
        }
        
    }
    return $chongfu;
}
function jiaohuan(&$arr, &$chongfu, $key)
{
    if ($key != $arr[$key]) {
       // var_dump($key, $arr[$key]);
        $value = $arr[$key];
        if ($value == $arr[$value]) {
            $chongfu[] = $value;//var_dump($chongfu, $key, $value);
        } else {
            list($arr[$key], $arr[$value]) = [$arr[$value], $arr[$key]];
            //var_dump($arr, $key);
            jiaohuan($arr, $chongfu, $key);
        }
    } else {
        return ;
    }
}

var_dump(chongfu($arr));


exit;
```
```
广度优先算法
```

```
567123, 怎么找出1
// 看到有序，想到查找算法，比如二分
// 随机找到一个数，比如6，5比6 小， 3比6大，说明1 在3到6 中间，再在 3和 6 中间取一个数字
```





# 项目
```
订单系统
```
```
100件商品高并发，先到先得
```
```
30w 的ip地址,类似如下
10.23.25.0  10.23.25.255    湖南省 长沙市
请设计出一个实现方式,可以给某个IP找到对应的省和市。要求效率尽可能的高。

利用redis的zset, 把ip地址变成数字作为score 存储，然后把各个省市的最大ip和最小ip存储，利用 zset 的范围，取到第一个最大的城市就是该城市
```

```
.请设计一个投票系统,满足如下要求
a.一个用户10分钟内对一个投票只能成功完成一次。
b.一个用户每个自然日最多只能成功完成50个不同的投票。

每次投票成功， 就往set 里面塞入一个投票类型，每次投片前都检测下，有没有超过50
利用redis, userid_投的票  => 时间，如果 时间 + 10 分钟 >= 现在时间就给投票
```

```
.用php写一个函数，获取一个文本文件最后$n行内容，要求尽可能效率更高，并可以跨平台使用

利用exec 执行 tail -f , 主要就是tail -f 没有停止，会持续获取，但也没关系，手动停了就好了，注意的是这块没停止的话，进程一直阻塞着，不能去往下执行，打印 exec 执行后的内容


$filename = '/opt/httpd/logs/access_log';
$xie = './xie.log';

function kan($line, $file, $xie)
{
    $line = $line -1;
    $x = fopen($xie, 'a+');
    $res = fopen($file, 'a+');
    $postion = -2;
   
    $str = fgets($res);
   
    while ($line) {
        fseek($res, $postion, SEEK_END);
        $tmp = fgetc($res);
        $postion -= 1;
       
        if ($tmp == PHP_EOL) {
            
            $str .= fgets($res);
            $line -= 1;
        }
    }
    var_dump($str);exit;
    fwrite($x, $str);
}
kan( 5, $filename, $xie);

exit;

```

```
有两个文本文件 A.txt B.txt
A.txt 3000万行，userid唯一，userid和username以空格分隔，如下所示：
userid  username
1       yi
2       er
3       san
...     ...
B.txt 3000万行，userid唯一，userid和realname以空格分隔，如下所示：
userid  realname
1       一
2       二
3       三

感觉可以把 a读入一个数组中，然后遍历 b ，往b中写入相应数据 。 a可以看做一个hash，所以查找一个数据就是1， 然后遍历 b文件，n， 所以总的复杂度是 n.

```

