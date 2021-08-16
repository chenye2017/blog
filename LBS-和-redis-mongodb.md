---
title: LBS 和 redis mongodb
date: 2018-12-24 22:00:01
tags: [redis,mongodb]
categories: Redis
---

LBS,刚听起来挺高大上的，其实简单理解就是附近的人。其实这个需求蛮常见的，比如大众点评中搜索摸某一个店铺按照离我当前位置距离进行排序，又或者某些社交软件中的附近的人，实现的方式有很多，php + mysql ， php + redis ， php+ mongodb， 其中mysql 的话，就是把所有的点的坐标保存下来，然后通过编写sql语句，直接通过sql去查询每个点离我们的距离，然后排序返回，但数据量大的时候，可能不太好，所以就有了后面的nosql ， redis 和 mongodb 的相关应用，其中 mongodb 在这方面应该是我听得最多的，当初想试这个功能也是因为想教一下前端朋友mongodb的相关功能，方便他面试，但后来发现其实还是redis 实现比较方便，因为php的mongodb扩展 mongo(php5） 和 mongodb(php7) 接口一点都不一样，而且中文文档比较少啦。其实通过es 也是可以实现的，方法很多.
[详细见这篇文章](https://zhuanlan.zhihu.com/p/31380780)
<!--more-->

## Redis
([参考](https://yq.aliyun.com/articles/62844))redis 自从3.2就有了对于经纬度的处理，但注意并不是新增了一个一个数据结构，这个经纬度是在 有序集合zset 的基础上发展而来的，所以我们对于zset 的操作都能对经纬度使用，比如
```
zrange key 0 10 #  把key 中所有的member 都取出来 zrange cy1 0 10
zcard key   # key 中有多少个member, zcard cy1
zsore  key member #  key 中具体的member 的score，对于经纬度这个取出来看不懂
```
穿插一句：刚开始使用redis 的时候，虽然他的api很简单，便于理解，但是和sql编程还是不同的，为什么我们一个对象有了member 这个名称还需要key 呢，你想啊，我么需要从一个集合中获取离这个点比较近的点，这个集合的名称就是 key,这个是事先我们往集合中塞就定好的，后面我们可能变化的是member 的名称，需要我们从外边传入。
关于php使用redis, 大致分成两种方法，有通过编译c扩展，还有就是通过composer 安装predis,这是两种方法，我们不要混淆了，平时我们大部分用的是c扩展额，然后自己封装其中基本的方法和连接方式，需要注意的是关于重连啊，或者单例模式是的使用啊之类的，还有就是其中__call方法的使用，如果有那些我们没有封装的方法，但是自己还使用了，默认就会调用这个方法。一般很多时候调用方法正确的放回是0和1，所以我们不能根据0和1来判定方法是否执行成功，很多时候他代表着影响的行数，比如新增返回1，修改返回0。还有c扩展方式安装的，比如 geoRadiusByMember, 不知道这个方法的参数都是啥，也不知道在哪能找到，只能凭着redis的使用去猜测，比如最后一个option 应该传数组，但其实如果传错的话，也有提示，感谢扩展报错信息的详细
![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/3280215645624671911.png)

geoadd: 增加地理位置的坐标。
geodist: 获取两个地理位置的距离。
geohash: 获取地理位置的GeoHash值。
geopos: 获取地理位置的坐标。
georadius: 根据给定经纬度坐标获取指定范围内的地理位置集合。
georadiusbymember: 根据给定地理位置获取指定范围内的地理位置集合。

georadius 和 georadiusbymember 的参数

WITHDIST: 同时返回地理位置与给定位置的距离
WITHCOORD: 同时返回地理位置的经纬度坐标
WITHHASH: 同时返回Redis内部的GeoHash值（非标准算法值），一般用于debug
ASC|DESC：结果按距离升降序排序
STORE|STOREDIST: 结果存到新的有序集合中，前者以GeoHash值做score，后者以与指定位置的距离作score，该选项与WITH[DIST|COORD|HASH]选项冲突

```
public function addLocation()
    {
        $redis = new Redis();

        //var_dump($redis->geodist('cy1', 'a', 'b'));exit;

        $lnt = Request::instance()->post('lnt');
        $lat = Request::instance()->post('lat');
        $name = Request::instance()->post('name');
      // 1 成功 新增
        // 0 也可能是成功  修改

        $res = $redis->geoadd('cy1', $lnt ,$lat, $name); // 相同的名称就直接替换了
        return $res;
    }

    public function dist()
    {
        $member1 = Request::instance()->post('member1');
        $member2 = Request::instance()->post('member2');

        $redis = new Redis();
        var_dump($redis->geodist('cy1', $member1, $member2));
    }

    public function fujin()
    {
        $radius = Request::instance()->post('radius'); // 多少半径
        $member = Request::instance()->post('member'); // 点的名称
        $unit = Request::instance()->post('unit'); // 单位

        $option = ['withdist', 'withcoord'];

        $redis = new Redis();
        var_dump($redis->georadiusbymember('cy1', $member, $radius, $unit, $option

        ));
    }
```
验证方法，通过求得两个点的距离之后按照多少米之内半径调整圆的大小，来观察另一点是否在圆圈内来验证.

## Mongodb
关于mongodb的安装就看菜鸟教程吧，也是解压完就能用，类似redis,然后服务端启动是 mongod, 客户端shell是 mongo(然后据说是js shell，不知道需不需要node的支持，反正我的服务器上node 是必须的，因为vue 啊 或者 hexo 啊 都需要node), 启动的时候要设置 dbpath 和 log path,直接写到配置文件中就可以 mongod启动了。

小插曲：因为这个mongo启动的时候输了一堆的信息，然后又看不懂，所以就像通过检查端口号来看是否启动成功，这又到了用ps 和 netstat 的时候了。先通过ps -ef|grep 27017 没有，是不是这个命令本来就不输出端口号呢？ps -ef |grep nginx,果然没有端口号，于是netstat -antp|grep 27017,看到了，害怕这个应用层服务不是基于tcp (也许是多虑吧)，我把t 去掉了，查到了。完事之后自己思考了下，ps 主要是用来检查进程的，netstat 是用来检查网络连接的，拿nginx 来举例，他可能有一个manage 进程，然后有个master 子进程，然后每个master还有worker进程，然后他监听了 80 ，443 等端口，所以ps 查出来的只能是进程相关，netstat 查出来的才是各种提供的服务和端口号。

mongodb 和 redis 同属于nosql，拿现在比较流行的话说，他也是对标mysql的产品，很多教程拿他和mysql做对比，确实也更容易理解
![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/9864415645624675608.png)

我们刚开始进入shell :
show dbs   显示有哪些数据库，默认 local 和 admin
可是默认我们在test 下面，为什么test 没有显示，因为test 中没有内容呀，但凡我们插入数据，就可以有了

### 数据库

use test 切换数据库，如果没有就新建

db 显示当前数据库

db.dropdatabase  删除当前数据库

### 表
db.runoob.drop 删除当前表/collection

show tables   显示当前数据库中有哪些表 

show collections  同上

db.createCollection('test')   创建表

db.createCollection("mycol", { capped : true, autoIndexId : true, size : 
   6142800, max : 10000 } ) 创建表

###  插入
db.runoob.insert({"name":"菜鸟教程"})  插入数据 ，这个runoob 是表的名字，在mongodb 中是文档的意思，然后mongo 中table 中 field 不需要统一

db.col.insert({title: 'MongoDB 教程', 
    description: 'MongoDB 是一个 Nosql 数据库',
    by: '菜鸟教程',
    url: 'http://www.runoob.com',
    tags: ['mongodb', 'database', 'NoSQL'],
    likes: 100
})   // 往col 表里面插入数据

###  查找
db.col.find({title:'cy', sex:'boy'})   //查找col 表里面数据,and 条件
db.col.find().pretty() 更好的显示出来

![image.png](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/8883215645624677668.png)


###  更新
db.col.update({'title':'MongoDB 教程'},{$set:{'title':'MongoDB'}})

>db.col.save({
    "_id" : ObjectId("56064f89ade2f21f36b03136"),
    "title" : "MongoDB",
    "description" : "MongoDB 是一个 Nosql 数据库",
    "by" : "Runoob",
    "url" : "http://www.runoob.com",
    "tags" : [
            "mongodb",
            "NoSQL"
    ],
    "likes" : 110
})

save 的话，保存id 就是更新，没有id 的话就是新增

默认的话只能更新一条，如下可以更新多条，因为第二个布尔值代表多条的意思

db.col.update( { "count" : { $gt : 3 } } , { $set : { "test2" : "OK"} },false,true );

###  删除
db.col.remove({'title':'MongoDB 教程'}) 
db.col.remove({'title':'MongoDB 教程'}， 1)  // 只想删除一个，just one 参数设置为1

###  条件
db.col.find({likes : {$lt :200, $gt : 100}})

### type
db.col.find({"title" : {$type : 2}})

### 分页
db.COLLECTION_NAME.find().limit(NUMBER).skip(NUMBER)

### 排序
db.COLLECTION_NAME.find().sort({KEY:1})

### 索引
db.col.createIndex({"title":1})
db.values.createIndex({open: 1, close: 1}, {background: true})，第二个是参数，前面的1是升序，-1是降序

未完待续····（坑爹的php 的 mongodb扩展，感觉不知道咋用呀）



