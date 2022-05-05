2021年11月18日入职了b站直播中心营收增长，这段时间以来也遇到一些问题，记录下，康康是不是能帮助后面的朋友少走一些弯路。

### 分表规则

```
```



### HTTP接口超时控制

```
```



###  err is shadowed during return

```
https://blog.csdn.net/wo198711203217/article/details/60574268
```



### go的继承问题

```
背景:
同事今天问了一个go的继承问题，go 有没有继承。
我平时对继承的认知只是需要 子类拥有父类的方法就行了，比如这样

type Father struct{}
func (f *Father)Name() {}

type Son strcut{Father}
这样son 就可以调用father 的name 方法

同事想出来的场景：
type Father struct{}
func (f *Father)Name() {
		fmt.Println(f.GetName())
}
func (f *Father)GetName() string{
   return "father"
}

type Son strcut{Father}
func (f *Son)GetName() string{
   return "son"
}

他希望的是son GetName 输出的是son， 因为 son自己实现了getname。 这在php 和 python 中都是这样的返回结果， go 中不行，go中是完全分开的，如果出现重名，你想要调用father 的GetName ，需要 son.Father.GetName()。

遇到的问题：
上面的过程容易让人产生一种想法，如果我们通过结构体嵌套， 不能改变原先结构体的任何属性。事实证明也是如此，我在看公司代码的时候，公司有个一ctx 取消超时的控制实现：

下面的ctx 其实并没有取消嵌套的ctx 的超时，可以直接这样尝试下就会发现
ctx, _ := context.WithTimeout(context.Background(), 2*time.Second)

c1 := C{
		ctx,
}
fmt.Println(c1.Context.Deadline())
fmt.Println(c1.Deadline())

总结:
go 的继承更像是不同结构体的拼凑实现。如果想实现背景里面的公共的方法2 调用重写的方法1，可以通过interface 去实现

type People interface {
   GetName() string
}
type Father struct{}

func (f *Father)GetName() string{
   return "father"
}
type Son strcut{}
func (f *Son)GetName() string{
   return "son"
}

公共方法
func Name(p People) {
		fmt.Println(f.GetName())
}

```



![取消超时ctx](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20220505111026.png)