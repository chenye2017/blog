---
title: tp源码分析-validate验证器
date: 2019-12-16 00:12:26
tags: [PHP,ThinkPhP,validate]
categories: PHP
---

我们在写接口的时候经常要对传入的参数进行验证，平时供客户端调用的接口还好，一般他们不会乱传什么内容，但也保不准他们的参数是从某个地方获取的，然后误传了，所以我们还是需要做好我们自己的防线，防止后端服务崩溃。还有种情况就是我们开发的后端管理模块，是给运营他们使用的，他们有时候可能配置出错了，这很容易导致我们配置的app比如界面崩溃，所以这块我们的代码一定要严谨。

曾想着看laravel 的验证器咋写的，！！！但真的类封装的太多了，头疼，算了还是看tp的吧，简简单单两个类搞定。

<!--more-->

首先说下php 的验证库可以做哪些事吧，还是蛮丰富的。

1. 首先是验证器，在我看来验证器就是单独抽象了一个层出来，我们在controller里面只需要调用，让controller更加简洁，而且也能复用，验证器类都继承于think\validate。
2. 其次controller 里面的validate 方法可以直接调用，本质上和前面一样都是调用tp的验证器库。
3. 最后我们如果想单独使用验证器，就当做普通方法那样也是可以的，毕竟有时候我们除了对于传入参数的验证，业务代码中也有一些内容需要验证的。



### Validate

属性：

static type : 这个主要是我们第三种方法独立验证器用到的。rule 规则都存在这里面，当我们使用独立验证器的时候，想扩展验证规则，extend 方法修改的就是这个属性，因为是static， 所以在本次请求的下次使用中，也能调用上次extend 的验证规则

alias： 就是我们rule 中可以用  >: 5, 当做 gt: 5 来使用

rule: 验证规则，我们在写验证器的时候，重写的就是这个属性

message: 提示信息的重写。包含5个占位符，:attribute, :rule, :1, :2, :3

field: 字段的描述，主要作用用在message中message 中的占位符 :attribute 会被这个内容替代

typeMsg: 提示信息呗，这块是英文的，最终会通过 lang\zh-cn.php 转换成中文

currentScene: 当前验证的场景，这个感觉就是为了验证器准备的（场景），其实也蛮实用的，方便验证器中规则的自由组合。

filter: 主要是利用php filter_var函数的验证

regex: 主要是用到正则的验证, ctype 也是php 很好用的一个扩展，方便一些特殊字符的验证

scene: 场景

error: 错误信息，只有控制器中错误才能抛出异常

batch: 当前参数验证错误，还能轮到下一个参数

only: 场景中哪些参数被验证

remove: 场景中移除

append: 场景中添加 

```
看源码写的一点总结
// remove 不可能移除所有，除非append 中是空
// remove 连续移除是通过 |
// remove append 的最佳实践是二者不要存在顺序关系，因为程序不会鉴别
```



方法：

__construct:  

rule 验证规则， message 提醒， field 字段描述 :attribute 占位符实际内容



make:

new self 生成一个验证器



rule:

独立验证修改rule 和 field 属性，供后面check 使用。方便直接传入 rule 和 message 



extend:

独立验证器中的方法扩展



setTypeMsg:

独立验证器中使用



message:

提示信息



scene:

修改当前验证场景



hasScene:

scene 优先考虑 sceneEdit 方法，再考虑 scene -> ['edit' => []] 属性



batch:

批量验证



only:

场景的使用验证哪些字段



remove:

场景移除验证规则



append:

场景添加沿正规则: 上线 remove 和 append的调用的时候没有先后顺序 ！！！



check:

验证器核心，这时候传入的内容会覆盖掉初始化的内容。



getScene:

根据传入的scene 和 current scene 修改 only  remove add 内容



getDataValue: 

根据传入的data 数组和 可以， 获取对应的内容



checkItem:

实际的验证，通过 call_user_func 去调用对应的验证内容



checkRule:

官网上这么说，

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20191216005751.png)

感觉意义不大，毕竟静态方法make 创造的更好用点



getRuleMsg: 

获取错误信息，可以好好看看，挺好的，调用了lang 类。因为自定义的closure ，如果 不是 === true， 就当做error， 结果就当做errorMsg，和调用类方法还是有点区别的。



getValidateType:

像一些 is，比如 isEmail, isIp 都走的is 方法



后面就是一些验证方法了，以后写代码都可以借鉴下



### ValidateRule

这个类的主要用途就是通过rule 类的形式去调用对应的验证规则

属性：

title  :attribute

rule 规则

message 提示信息



方法：

addItem ：

为 is 方法统一添加rule，message 属性



getRule  getTile  getMesage 获取对应属性



title 设定title



——call  

——callStatic 







