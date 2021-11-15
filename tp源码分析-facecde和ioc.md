---
title: tp源码分析-facecde和ioc
date: 2019-08-09 09:47:55
tags: [PHP, ThinkPHP, 设计模式, facecde, ioc, 依赖注入]
categories: PHP
---

首先我们梳理一下关系，ioc 作为一个容器，是个单例的存在，它存在于自身container的instance 属性中，他还有个instances 属性，作为一个关联数组，key的内容是类名，value 内容是某个类的实例(这个类并不一定要是单例)， 当我们下次再去容器中取这个类实例的时候，还是之前的实例 (!!!千万注意一定是容器中取，才能保证是上次的类实例), 我们从容器中获取类实例，并不一定要提前bindto, bind 本质只是绑定一个别名，通过别名最终还是找到实际的类名，所以没有绑定直接传类名也是可以的。 那什么是facade 呢， 通过返回容器中的类别名或者类实际的名称，去容器中找类实例，然后调用该实际类的方法

<!--more-->

### Container

Container 实现了arrayAccess (方便数组调用)， iteratorAggregate (方便foreach循环) , countable(方便 count()) 调用

instance    container 的单例

instances  []    装的类的实例

bind  类标识，通过标识寻找实际类

name  别名，感觉和前面bind差不太多



getInstance 

获取container 的单例



setInstance

设置container 的单例



get

从container 的单例instance 的属性 instances 中获取类的实例，对于container 的单例调用 make 方法



set


container 实例 绑定一个类、闭包、实例、接口实现到容器，注意可能并没有生成类的实例，也就是instances属性中可能并没有




remove 

container 实例中instances 属性删除某个类实例



clear



bindTo

修改bind属性，包括关联数组（merge）, 闭包直接赋值，string 修改bind, 对象，修改bind ,并把 instances中绑定这个实例



instance 

绑定一个实例到容器中，感觉作用相当于bindTo的一部分



bound

判断容器中是否存在类标识或者实例



exists

判断容器中是否有该类实例

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20190809123749.png)

从这个方法很容易判断，instances 中key 是实际的类名



has

调用的上面bound 方法



make

创建类的实例， 第三个参数newInstance 类似我之前写的那个token 文件的第三个参数 force，强制刷新实例。

make 会先去查看是否有别名，然后换算成实际类的名称，去instances属性中查找。如果没有只能生成啦。如果是匿名函数，通过反射获取参数，然后绑定参数，如果有依赖注入，首先查看是否有传入这个实例，没有的话生成。如果是类的话，会先判断是否有__make 方法，没有的话调用constructor 构造函数，执行函数的过程类似上面执行匿名函数



delete

删除instances中的实例



all

输出instances内容，也就是生成的所有实例



flush

置空 instances ， bind， name 属性



invokeFunction

从一个方法中获取反射，然后执行这个方法



invokeMethod

从一个类的方法中获取反射，然后执行这个方法



invokeReflectMethod

这个方式要求传入实例还有反射，还有参数，（所有的参数都给备齐了）就相当于上面方法的子方法，上面方法也是调用这个方法去实际执行反射类中的方法



invoke

实现方法



invokeClass

实现类



bindParams 

生成类或者方法需要的参数，支持依赖注入



getObjectParam

bindParams 对于依赖注入的处理需要用到这个方法



后面的方法都是为了上面那几个接口中方法的实现，操作的都是instance 实例中的instances属性



### Facade

bind 别名，类似container 中的bind

alwaysNewInstance ，因为facade 本质上也是从container 中取实例，所以也需要这个强制刷新属性



bind

绑定别名



createFacade

先根据继承了facade 的类中getFacadeClass 返回的内容中确定类，如果没有，去bind属性中查找（注意这里是self）,都没有的话就用传入的类名。确定了类名去container 中找实例，没有的话自动生成



getFacadeClass 

返回容器中类的别名或者直接返回类的名称



instance



make

和createFacade 差不多



__callStatic

facade 的核心，通过createFacade 实例化实际类，然后用这个实例去调用method方法 （一定要注意我们在调用类中的方法的时候除了static都是在一个类实例上调用的方法，我们的方法中可能会用到\$this ,这个\$this就是对象）



### 总结

以Config 举例，\Config 通过class_alias 实际找到的是 \think\facade\config,然后这个类中没有我们想要的方法，调用的是父类 facade中的__callStatic, 这个方法先通过createFacade 生成实例，实例是先从container 中查找别名换成真的类名，然后去instance中查找实例，有就返回，没有再生成。实例获取到后，再去执行方法







