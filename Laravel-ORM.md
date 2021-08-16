---
title: Laravel ORM
date: 2018-03-30 11:30:05
tags: [Laravel,orm]
categories: Laravel
---

orm并不是一个新兴的概念，他出现很久了，只是刚开始学习的自己并没有接触到，他能帮我们少写很多代码，让我们能以更面向对象的方式来处理关于数据库的操作。

<!--more-->

Eloquent 是laravel的orm，我们通过

```
php artisan make:model User
php artisan make:model Models/User  //放在文件夹models里面
php artisan make:model Models/User -m  //顺便把迁移文件也创建了
```

生成的类虽然是继承Model，但本质上都是继承这个Eloquent，他把我们每张表看做一个类，表里面的数据看做一个实例，我们对其进行操作的时候，更类似于面向对象的操作

```
$user = User::find(1);
$user->name = 'cy';
$user->city = 'shanghai';
$user->save();
```

更便于理解。

生成的model很简单，大致

```
<?php

class Article extends \Eloquent {

protected $table = 'my_article';  //自定义表名

protected $fillable = [];

protected $hidden = [];

}
```

关于表名：基本都是小写，然后多个单词用_连接（php的类是首字母大写的驼峰命名法）

这个fillable里面就是可以批量操作的字段，批量操作在我之前的文章里面有提到过，我觉得更通俗的理解是可更新的字段。主要是create方法时的那种批量更新，会用到。

hidden 就是对一些用户数据的隐藏，还记得我们经常取数据时候的select * 吗，容易把用户的身份证号，密码都取出来，这样不太好

只需要继承一下 Eloquent 类，就可以干 'first() find() where() orderBy()' 等非常非常多的事情，这就是面向对象的强大威力。

通过

```
php artisan tinker
```

可以开启一个交互环境进行orm的调试，其实我感觉不止是orm，别的代码也都是可以执行的，只是方便了orm的打印操作，能很方便的看到执行结果。有一点需要注意的就是tinker是基于当前代码的，如果代码重新修改来了，我们需要重启tinker来加载新的代码。

还有我们会利用composer安装laravel-debugbar，来进行orm的调试，当我发现在别的项目中使用不了这个插件的时候，我改变了对composer的看法，他安装的东西更类似于代码，而不是软件，我们如果要在不同的项目中也使用这个插件，我们同样需要在另一个项目目录线面进行composer安装

介绍几个常用的函数

* find()

  ```
  User::find(1)
  App\User {#764
       id: 1,
       name: "cy",
       email: "1967196626@qq.com",
       created_at: "2018-03-30 01:42:40",
       updated_at: "2018-03-30 01:59:51",
     }

  ```

  这个1是主键


* first()

  ```
  User::where('id', '>', 1)->first()
  App\User {#771
       id: 2,
       name: "Bella Lebsack",
       email: "cristobal58@example.net",
       created_at: "2018-03-30 01:59:51",
       updated_at: "2018-03-30 01:59:51",
     }
  ```


* save()  save 能新增用户，通过构建一个new 的User 对象(不需要设置model里面的fileable，create需要设置)，或者更新一个对象

  ```
   $user = new User();
   $user->name='cyc';
   $user->email = '199@qq.com';
   $user->notification_count = 0;
   $user->password = bcrypt('12345');
   $user->save();exit;
   
   $user = User::find(10);
   $user->name='cy';
   $user->save();
  ```

* update()更新

  ```
  $user = User::find(10);
  $user->update(['name'=>'Aufree'])
  ```

  ​

  ​

  上面两个获取的内容都是对象，可以直接使用

* get()  获取的是一个collection

  ```
   User::where('id', '>', 1)->where('id', '<', 3)->get()
  => Illuminate\Database\Eloquent\Collection {#773
       all: [
         App\User {#778
           id: 2,
           name: "Bella Lebsack",
           email: "cristobal58@example.net",
           created_at: "2018-03-30 01:59:51",
           updated_at: "2018-03-30 01:59:51",
         },
       ],
     }
  ```

* all() 获取的是一个collection

  ​all的使用好像不能加条件

  ​上面的collection可以通过toArray() 方法转换成数组，但其实我们平时用的时候没用toArray也是可以的

上面需要注意的点就是：

1. 所有的中间方法如 'where()' 'orderBy()' 等都能够同时支持 '静态' 和 '非静态链式' 两种方式调用，即 'Article::where()...' 和 'Article::....->where()'。
2. 所有的 '非固定用法' 的调用最后都需要一个操作来 '收尾'，本片教程中有两个 '收尾操作'：'->get()' 和 '->first()'。
3. 每一个继承了 Eloquent 的类都有两个 '固定用法' 'Article::find($number)' 'Article::all()'，前者会得到一个带有数据库中取出来值的对象，后者会得到一个包含整个数据库的对象合集。



Builder

```
Article::where('id', '>', 10)->where('id', '<', 20)->orderBy('updated_at', 'desc')->get();
```

这段代码的 `::where()->where()->orderBy()` 就是Builder。用面向对象的方法来理解，可以总结成一句话：创建一个对象，并不断修改它的属性，最后用一个操作来触发数据库操作。

```
User::orderBy('id', 'desc');
=> Illuminate\Database\Eloquent\Builder {#817}
```

这个get() 就是那个最后的方法。



值的思考的一个问题

> 如果直接用 :: 来访问某个 function，无论这个 function 是否为 static，构造函数 __construct() 都不会被调用，那么创建对象是如何实现的呢？请看：[https://github.com/illuminate/database/blob/master/Eloquent/Model.php#L3354](https://github.com/illuminate/database/blob/master/Eloquent/Model.php#L3354)

所谓 “终结者” 方法，指的是在 N 个中间操作流方法对某个 Eloquent 对象进行加工以后，触发最终的数据库查询操作，得到返回值。

`first()` `get()` `paginate()` `count()` `delete()` 是用的比较多的一些 “终结者” 方法，他们会在中间操作流的最后出现，把 SQL 打给数据库，得到返回数据，经过加工返回一个 Article 对象或者一群 Article 对象的集合。

all() 方法好像不是，因为all() 好像不能跟着条件

```
 User::orderBy('id', 'desc')->all();
BadMethodCallException with message 'Call to undefined method Illuminate\Database\Query\Builder::all()
```



关系

1 对 1 关系

做了个试验，发现如果通过传统的生成迁移文件，生成model，进行数据填充真的挺麻烦的，不如在数据库中快速建表来的容易

```
public function hasOne1()
    {
        return $this->hasOne(TestOneToOne::class, 'user_id', 'id');
    }

    public function belongsTo1()
    {
        return $this->belongsTo(TestOneToOne::class, 'id', 'user_id');
    }
```

这是User 中的两个方法

User::with('hasOne1')->find(1)

User::with('belongsTo1')->find(1)

获取的数据是一样的，说一下我的记忆方法。hasOne 的时候，对方表中的字段在前面，belongsTo的时候自己表中的数据在前面。（我一般喜欢用with表示模型的结合）

还有第一个参数注意一下，话说这个::class出现好久了，而我却很少用，真的很方便哦。

还有默认的orm模型名称都是表名的单数，这个是一定要注意的哦。除非自己重新定义。（many 都能识别成功manies,牛逼）



1对多

这个时候的belongsTo 只有一种了，和上面的一样，hasMany() 和上面也类似。值的注意的一点就是那个表::with就成主体了，还有一点的就是使用with的时候并不是和我们传统的那种使用left join，而是先计算出主表，再根据主表中的数据，就算从表中对应数据in() 这个集合中，我不能理解的一点就是这样通过in，怎么完成了数据的连接。



多对多

这个时候经常需要第三张表来存储数据，值的注意的一点就是第三张表往往不需要生成model。

第三张表值记录对应关系，感觉默认的就是主键id

```
public function belongsToMany1()
    {
        return $this->belongsToMany(\App\User::class, 'test_many_to_many', 'user_id', 'test_id');
    }
```

上面是一种我的粉丝，比如我是users 表中的一员，我的粉丝也是这张表中的数据，注意存储数据关系的表不要用::class，因为不存在这个class，还有如果要加where条件，里面的条件用调用主题的字段，比如users中的字段id。













