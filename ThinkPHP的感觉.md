---
title: ThinkPHP的感觉
date: 2018-11-12 18:12:27
tags: [PHP,ThinkPHP,框架]
categories: PHP
---

学习，为了更好地适应未来的工作。简单的解除了下tp和laravel，总体来说tp比laravel简单多了，封装的也没有那么深，所以用起来还是很方便的，用的是5.0，5.1和5.0区别就挺大了，比如有了门脸，又是一个使用类不知道到底用哪个命名空间下类比较好的时候了，还有获取参数的方式，5.1已经没有getInstance()这种东西了，记录一下这几天的感觉,方便交流

<!--more-->

### 目录结构
![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/4694815645624705617.png)

首先是模块的创建，主要是为了方便后台模块和 api 模块的分开

后台模块， 这个项目中主要是通过接口能返回html 模板

但是 api 模块，单单只用返回json 数据

### Controller

关于api 模块的json 返回，我们只要调用json() 方法就可以了，不需要做进一步的配置，  我们还可以通过json 方法的进一步封装，让调用起来更加方便

[图片上传失败...(image-14c2c0-1542018168779)]

![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/810621564562470764.png)

这种方法其实就是在返回头上加上个别的信息

这种公共的方法我们可以写在 common.php 文件中

控制器中获取传过来的参数

有个坑就是5.0 和 5.1 通过request 获取参数的方式竟然就不一样了，

5.0 中 其中一种 Request::instance()->get()

5.1 Request::get() (5.1中已经开始出现faceade这种看不懂的东西了)

其实用助手函数就没有上述问题了，···但无奈我不喜欢用，但感觉助手函数的话对不同版本的tp代码都是通用的

对于put 参数，我们在用postman 提交代码的时候需要通过第二种方式提交，第一种方式是获取不到的，可能是因为表单默认屏蔽了put delete 提交方式



![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/6614015645624709869.png)

![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/472831564562471279.png)

这个header 头信息是真的坑，解决这个办法就是先把$header 信息保留成数组，然后再通过数组去寻找这个变量

讲道理，还是有点不明白_initialize 和 __construct(） 的区别

关于文件的定义，基本上除了配置文件，剩下的都是class 文件，按照psr4 ,类名要和文件名一样，我之前有过类名和文件名不一样，导致文件加载失败

异常处理那块，当调用父的 异常处理函数，需要return ,否则会报错，感觉和调用模板也需要return 是一个道理

### Router

关于 tp 的路由，好久之前就知道被吐槽pathinfo 模式，虽然我不知道是因为什么被吐槽，除了pathinfo 还有别的模式吗？

除了默认的pathinfo, 我们对于api 接口（admin 那种返回html 的可以不用管），可以使用路由注册的形式，就是在route.php 中进行路由注册，他会首先匹配,没找到的会使用pathinfo 模式，这块需要我们在配置文件中进行配置

（可以完全关闭pathinfo 或者 路由注册的形式）

（关于路由注册这块，tp 的路由注册和laravel 的还不一样，他不是使用命名空间的形式，用的是感觉自己定义的形式， 但感觉因为这个方面，tp的路由方面应该不能单独拆出来给别的项目使用），但其实也挺好理解， 注意

完全路由匹配，还有路由参数的限制


![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/8861315645624715465.png)

参照： [https://www.jianshu.com/p/73ed6e42d389](https://www.jianshu.com/p/73ed6e42d389)

### Config

今天因为不太知道config() 助手函数都是从哪里获取的数据，查了一下，这篇文章写得还是蛮好的

[https://www.jianshu.com/p/cd8b68143fde](https://www.jianshu.com/p/cd8b68143fde)

比如我们那个api模块的接口需要很多额外的参数，我们只需要在application 最外层定义一个extra 的目录，然后会自动引入里面的文件

我们配置文件的数据返回大致如下


![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/684601564562471785.png)

通过 config() 可以看到全局的配置参数

###  DB

model 最近用的都是orm,不管是后台admin 还是api接口，都可以公用，所以新建了一个common模块，在里面写入了model,通过model() 或者命名空间的方式都能使用（我比较喜欢用命名空间的方式去使用，tp的orm可以分离出来去别的项目中使用，当然model()方法引入model类就不可以了，只能通过命名空间的方式）

题外话 ：关于为什么用model() 就能引入common 模块下的model，我们可以理解成约定俗成，甚至可以理解成1 + 1  = 2， 之前我纠结这些问题很久，但其实他就像我们使用别的约定俗成的概念是一样的，只是这个可能是由于框架的封装，在我们的印象中很难站住脚 ，或者没有看到官方给的白纸黑字的文档，能做的就是看源码，model 是怎么执行加载的，表哥说过，你可以画上 1年半载去弄懂框架的源码，但如果你弄不懂的话，你也可以通过专注业务区提升自己的能力，有些东西能用就好了。

基本的方法:


![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/2795015645624720051.png)

通过在数据库配置文件中设置表前缀，默认类名称是  '表前缀_类名' ， laravel 是表名单数

**查询**

select   获取集合  ，这个直接json 返回外面就是array ,在项目里面打印是 是object ,这个项目里面转换成 array 需要 collection($arr)->toArray()， 这个Db::  用select , 查出来竟让也是 object ,想变成 array,可以通过toArray()

find   只能获取一条 ,这个直接 就是数组

column, 查询单列

value  查询单个值（那种集合的查询调用 value 也是只返回一个值）

field  获取哪些对应字段,还可以给字段取别名（感觉很强大，还能用函数 sum(score)）

order   同上

limit  5 size

page  1  页码  (感觉这个page 还是和limit 分开来使用比较方便)

**查询方法**

where  =>   ['id'=> 1, 'status'=> 1],  多条件等于情况， ['id', 'status']  代表两个column

where => ['id' => ['>', 3]]   id 大于3的情况

whereOr

**插入**

insert

insertAll  批量插入

insertGetId  插入成功之后返回id

getLastInsID  返回最新插入的id

**更新**

update

setField 更新某个字段

setInc   自增

setDec 自减

**删除**

delete 

关于链式操作，注意select() ，table 这些不属于链式操作，所以需要放在最后面，剩下的那些链式操作没有顺序要求

alias  给表取别名

groupby  分组

having  （最好在group by 中使用count这种的聚合函数）

子查询和 union 基本不用

distinct 唯一不同的值，不同的值，比如这个返回admin和chunice


![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/1597715645624722466.png)

count  这个好像需要放在 where 条件后面，要不然也会报错

orm 之所以能调用这些db的方法是因为orm 的基础是db

自动写入时间戳都是针对orm 这种操作可以的，db 操作是不可以的， 上面的操作都是db 的操作

注意用Db 进行操作的时候，命名前缀已经失效了，记住要用表的全名

关于join 表，老师和表哥说都不要join ,因为数据多的时候join会出问题，对于用户和用户名的对应的时候，利用先找出所有数据，再把userid 取出来进行in 的查询，再通过遍历之前需要处理的数据，把username 链接进去，这样是可以的，但是对于zan这种，可能不存在这条数据

这几种join是当初面试的时候经常问的


![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/887215645624725236.png)

comment  给sql语句增加注释（··感觉好没用啊）

fetchSql 返回sql 语句 （ 感觉挺好，可以知道执行了什么，但用with 那种关联关系，只能返回第一条语句···，太废物了）

### ORM

其实，感觉：查询出来的是object 对象，还是纯种的array ,就是看调用者，如果是 Db::table 调用，就是 数组

如果是 User::  这种方式调用，就是 object， object 又有 user  和 collection 之分

**新增**

save   插入 这个不能用静态方法，只能实例化调用

saveAll 批量

返回值是受影响的行数，我们想获取到自增id ,可以通过 

$user = new User()

$user->save([])

echo $user->id ,这种方式获取到，因为save 不能非静态化的调用， save() 返回值是受影响的行数

批量新增 saveAll()

create  静态调用

User::create() , 返回值是自增id

**修改**

save ,需要传入主键

saveAll  批量

update 静态调用 ,这个方法也可以直接调用，Db和模型都有这个方法

据说通过 ->where->update() 这种就不能使用tp模型的事件功能（什么是事件功能？就是那种更新表前会有什么操作，类似钩子）

感觉这个方法除了静态调用，就不知道有什么比较方便的地方了，感觉那种直接通过模型只使用update() 方法不太好，不太直观，感觉很多时候应该是不止是就是用一个update 方法这么简单

**删除**

delete 删除，也是模型和Db 都能使用的

destroy 删除

**查询**

get （条件） 

，find

感觉查询的功能真的是各种强大，慢慢学习吧

获取器的重要作用：

其实我们在平时的开发中总会遇到类似的问题，就是数据库中对于status 存储着各种数字，然后给前端展示的时候，循环结果，把结果中的每条数据的status 对应的中文取出来，然后再加上，读取器就是为了这个产生的，我们在读取一个

attr 的时候，类似定义一个钩子，他会自动帮我们把取到的数据进行修改（其实有两个参数的，第一个是这个值，第二个是整条数据的值）


![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/720931564562472744.png)

all (条件) select ()

**软删除**

这种事很常用的，我们平时使用的时候通过加入一个标志位，然后tp中也是使用标志位，delete_time ,但筛选数据的时候不用我们处理，仿佛真的就删除一样，可是在管理工具中还是能看见

关于orm 的执行日志，或者是说sql日志，统一的开启数据库调试模式，然后直接在log中查看就好了

notice : 就是很灵活，因为orm 也是基于数据库的，所以数据库中的很多方法在orm 中也可以用，甚至同名，比如update,但返回的内容不一样，orm 的update 返回的是更新的实例，数据库的update 返回的是兽影响的行数

感觉就是模型的话要是不用with 这些关联属性，意义会小很多，可是后来发现，那些处理orm 查出来的的数据，确实会方便很多

关联：

1对 1 ： 

虽然感觉有时候反着写也是对的，但最好还是遵从 （虽然七月老师说是二者是不对等的，比如 user 和 userinfo  ,userinfo 有 userid ,但user 找不到 userinfo ,但如果是 1对 1 ，是真的都能通过定义 hasOne 和 belongsTo 去获取对方）

（按照编辑器的推荐，确实 hasOne 更准确点）

（user  hasOne  userinfo   这个也更符合我们的习惯）

感觉上反正很怪，如果编辑器能提醒就好了，但只能提醒是主键还是外键，并不是提醒是哪种表的主键，那张表的外键


![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/348871564562472971.png)

user  -> info

user 中定义 hasOne

$this->hasOne('info', 'user_id','id'），这个提示比较符合大众的想法

info 中定义 belongsTo


![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/3707915645624732446.png)

$this->belongsTo('user', 'user_id,'id')

单个模型还能通过

$banner->hidden(['update_time', 'delete_time']); 来隐藏对应的字段

*感觉最好还是在model 的定义的时候隐藏*

*注意select () 这种查找出来的都是数据集合 collection*


![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/3186415645624734785.png)

看一下这个例子，通过嵌套关系，可以实现images 内部数据的排序，所以关联关系也是可以嵌套的



### Cache

自己新建的缓存文件不要和 thinkphp里面的字段名称一样，否则回报错误

###  **Exception**

配置文件中开启debug 模式，要不然总是 ‘tp 专为api而生，十年磨一剑’的错误信息

开启之后还需要处理，因为我们前台只需要json数据，我们需要考虑程序上线之后如果发生错误了我们该怎么处理，其实没法处理，因为我们不知道什么时候会发生，想象一下如果我们知道他会发生，那我们为什么不去阻止而让他发生呢，此时我们需要一个监听全局异常事件的东西，当php抛出异常的时候我们只要去捕获他，然后处理就好了，类似前端的钩子 。 

知道吗 ？在我们原生的php当中，其实有个专门对全局异常进行处理的函数，可以自己定义，可是呢··我没有自己去试过，首先是异常自己理解的比较浅，再者对于框架的使用比较少，

tp的全局异常处理应该也是这个原理，本质上应该还是修改这个函数，当我们在tp中想改变全局异常处理函数只需要修改配置文件中的异常处理函数名称，改成自己需要配置的就好了

接下来我们重写下全局异常处理函数，让他更符合我们的要求


![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/4308715645624736836.png)

首先需要继承tp自带的异常处理，然后重写他的render 方法，，为什么？因为我们在bug模式的时候需要tp给我们提示错误信息，总比我们自己看 $e->getMessage() 方便详细的多



![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/1986215645624739049.png)

接下来， Basci Exception 是自己默认定义的exception， 既然是自己定义的错误，自然不用tp的异常处理，直接返回前台json 数据就好了，因为自己定义的错误，一般是手动抛出，自己了解错误的原因

然后是非自己定义，首先要是判断是否开启了debug 模式，如果开启了，说明在调试，直接使用tp自带的就好了，千万注意要加上return ,要不然会报错，毕竟是html 页面的返回（类似可以想象一下tp中模板的返回）

再者是如果是上线环境，只需要告知前台服务器错误就好了，定义个固定的数组，返回给前端，然后记日志

（记录日志也是很简单的，log::error(), 写在runtime的log下面）

异常处理定义完毕，就要开始定义异常类了。使用异常类的原因就是告别传统的遇到错误的返回方式，还记得之前我们在嵌套函数中执行错误后怎么返回的吗，只能层层往上抛出，到了最外层的controller 返回，可是异常就不一样了，当异常处理检测到这个地方抛出的异常，就能直接进入异常处理函数中，多方便。

（还有一点我觉得定义异常的好处，就是之前我们对错误信息的输出总是 code => message => ,有时候为了规范我们通过写一个方法，通过code 去获取message ，但你想象一下，当你写完接口之后，满屏的 code ，你也不知道这个code 代表的啥意思，当然我们可以约定code的规范，如果我们通过throw new ParamErrorException([code=> , message=>]), 我们就会知道这个错误大致属于前端参数的错误）

notice :
1.不管是什么框架，我们定义的异常都一定继承于最原始的php \Exception

2.常用的方法 $e->getMessage(),获取错误信息

3.注意异常类因为继承自exception，我们一定不能定义exception中有的属性，比如我之前在异常类中定义过一个message属性，这个属性是exception 中的protected，然后在参数验证那块会出现莫名其妙的异常错误

### Validate

说实话，我到现在还没有明白验证器的具体作用，平时那种

new Vdalidate()

->bitch()->check()

感觉就能符合需要，可是要是真的每次在valiate里面写上同样的验证规则， 确实可能会有重复定义的嫌疑

但我感觉要真的发挥验证器的作用，必须介乎scene() 场景来使用，一个validate 对应一个类，或者model，这个validate有各个属性的校验，通过定义scene ，让我们不会定义过多的validate 类

在 $rule 里面

有各个报错信息 $message =  ['id.require' =>] 这种细致的分层，让错误信息准确的定位，或者直接 id => ,这样可能id对应多个验证规则的时候，需要把错误信息都加上，如果出错了，不知道具体是哪条规则出错了

scene ,对应场景，可能某个实体过来了，我需要验证其中valdiate 类中的某几个属性，我就用scene 去细分，如果不用scene 的话，想象一下，这个地方比如user 模型需要name 和 sex ，另一个地方需要 name 和 img ，难道我们需要为此定义两个validate ，感觉一点都不灵活（exception如果遇到这种通过向里面传入不同的参数来解决）

重写validate 和 重写 exception 不一样，重写valdiate ，其实是对传统 validate check 方法的封装，和 定义方法的更多加入，因为check 之后，所有的错误信息都会储存在 error  这个属性中（或者 $this->geterror()来获取），然后通过异常把这个错误信息抛出去给前台页面

![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/9710215645624742651.png)

### 其他

![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/2148015645624745182.png)

laravel 中也有相似功能，感觉就和路由中间件一样，在执行到这一步之前都会先执行这个验证操作

感觉这个比那个__initze() 什么方法方便多了



### **总结**

还有太多太多我不会用的，所以后面的路还很长啊，只是 为了更好的入门

### 


