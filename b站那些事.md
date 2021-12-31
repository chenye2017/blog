### 异味:

```
1.关于 map 中key 的取值，一直比较喜欢这样， tmp, _ := m1["xxx"], 这样取值。但其实不用，只需要 tmp := m1["xxx"], 如果取不到值，不会panic，会给对应属性的零值。 只有当我们需要的判断这个给值是默认的零值还是本身就是零值得时候需要这个ok 参数。

2.断言panic。断言的时候一定要 tmp, _ := (interface).(int), 否则如果断言错误，会直接panic

```



### 关于error 处理的正确方式

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/errors.png)

```
b站的ecode 有一点，就是Cause() 方法是基于 pkg/errors 开发的，对于堆栈信息的保存这些并没有在自己包中有，所以如果需要堆栈信息，还是得用pkg/errors 去生成error。

b站的ecode 核心就是code 错误码，比对的时候也是用code 码去做大小比对，msg 可以自己传进去，并没有过多的约束。所以经常 code码是一致的，msg 不一样。我们即可以用ecode 服务自带的msg， 也可以用自己的 Error 生成时候传进去的msg。

b站的web server 在 c.json 输出的时候，会把 error 做cause ,然后断言成ecode 的 codes， 如果是 ecode 的codes， 那就返回这个error 的Error() 输出，如果不是，就直接返回 500 错误。 
```



### 关于error 什么时候用 wrap

```
因为 errors.Wrap 会返回堆栈，我们底层的io 包含了这个堆栈，上一层调用io 的，只需要 withMsg 包含一层信息吐出去就好了，这样就不用重复打印堆栈了。

rpc 调用的时候我们没有返回堆栈，是因为b站这边的 ecode error 是不包含堆栈的，他的核心就是 code。

```



### 关于方法中的error 是否要记录日志

```
感觉可以直接wrap 后吐出，因为外层肯定要判断 + log + scene， 如果内层也记录 就重复记录了。但如果是内层记录的可以少很多log 日志的地方。

但内层记录的可以把scene 传到方法里面，方便取出日志。

info 日志的话可以在内层记录，因为info 日志有时候要每个io 都记录方便排查，如果记录在外层，可能不是最原始的io，而是处理过的io。
```



### 关于https 的抓包

```
在虎扑，我们的手机可以一键切换hosts， 所以一般我们切换到http， 就不需要装证书抓包。但b站这边应该没有这个功能，所以当我们想连接别人电脑charles 抓包，一般都不会成功。

charles 好像针对每台电脑生成的证书都不一样。

web 端我们也需要信任证书。 mac 上证书的信任都在钥匙串里。
```

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20211223105204.png)