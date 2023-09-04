## Interface 内部结构

```
iface {
   _itab 类型
   data 数据指针
}

eface {
   type 类型
   data 数据指针
  } 
  
  (empty的意思)

必须type data 都是nil, 才能代表是 nil。
```



## interface 在一些包中的传参转换

```
 这个考的概率不大，平时容易遇到的问题。 （es 的查询，或者 mysql query参数的传入）
 
 cannot use names (type []string) as type []interface {} in argument to printAll） 
  es 的 terms 查询条件。
  interface 参数可以被 string 等很多类型传入，但是 []interface 不能直接用 []string, 需要主动构造[]interface。 时间复杂度是 o(n)， 得一个一个转换
 https://www.jianshu.com/p/03c81f4e3956
```



## 哪些数据类型可以比较大小

```
interface 可以比较大小. (主要是比较是否相等， 经常用来和nil 比较是否相等)

常见error 比较大小, 其实是比较是否相等


slice， map 和 func 都不能比较是否相等，

需要通过 deep equal.

(数组能比较是否相等)

  注意这些比较大小的时候，都和元素顺序有关，（slice 有关，map 本身就是无序的，应该没啥关系，interface 这种涉及到 type  和 元素自身的比较大小）


  https://juejin.cn/post/6881912621616857102 
  关于是否可以比较


  struct 可以比较的前提是必须是同一种数据类型，或者是强制转换成统一种类型。还有如果 == ，必须所有字段都是可比较类型 


  type T2 struct {
      Name  string
      Age   int
      Arr   [2]bool
      ptr   *int
      map1  map[string]string
  }

  type T3 struct {
      Name  string
      Age   int
      Arr   [2]bool
      ptr   *int
      map1  map[string]string
  }

  func main() {
      var ss1 T2
      var ss2 T3
      ss3 := T2(ss2)
  fmt.Println(ss3==ss1)   // 含有不可比较成员变量， 还有这似乎得在同一个包中
  }
```



## 怎么判断 interface 是nil

```
下面本质上还是interface 比较的问题
  
  我们在返回类型的时候，如果返回类型是interface 类型，比如 error

 notice:
  1.我们直接返回nil （不给具体的动态type类型）
  2.如果我们返回 具体的类型，那不管这个类型表示的意义如何， 他都不等于 nil, 因为interface 的动态类型有了具体的值。

  


 因为返回interface 的话，我们很难判断是否是nil， （因为 interface 的 type 肯定不是nil 了）
 我们需要把他的值拉出来判断是否是 nil。可以通过反射。

 b站的 error ，相判断二者是否相等， 先断言成标准error  struct, 再利用该 error 的cause 方法解析。
 
 
 判断interface 是否value 是nil

  func main() {
      var data *byte
      var in interface{}
       in = data
  fmt.Println(IsNil(in))
  }


   // go的 反射包封装了 isNil 方法。可以了解下 reflect的 key ， value 之间的转换。
  func IsNil(i interface{}) bool {
      vi := reflect.ValueOf(i)
      if vi.Kind() == reflect.Ptr {  // 这块其实有缺陷，除了指针，还有 map slice func 的特殊处理
          return vi.IsNil()
      }
      return false
  }
```

```
 // error 的比较本质上还是 interface 的比较
 
 // 这个 error 错误比较很重要，因为类似redis 查询的时候都会用到， redis.ErrNil, 所以比较的时候都是对比 package 里面定义的变量
  https://www.veaxen.com/golang%e6%8e%a5%e5%8f%a3%e5%80%bc%ef%bc%88interface%ef%bc%89%e7%9a%84%e6%af%94%e8%be%83%e6%93%8d%e4%bd%9c%e5%88%86%e6%9e%90.html
  
   看了上面的文章，发现要真的是 errors.New("11") == errors.New("11") 如果为true 的话，那就完了。因为error 生成的时候又不会全局检查，万一一样了。
   
   
  没有方法的interface , 包含元素类型 和 元素的实际存储位置，只有二者都相等，我们才能认为2个interface 相等。

  同理 interface 和 nil 比较大小，也就是只有 type 和 value 都等于 nil 的时候我们才能认为他是 nil
  
```



![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/img/20211003170536.png)



```
上面这个图还是很经典的

MyErr 是个结构体

地址类型，给了 nil

所以

&MyErr == nil 是true

但我们要注意的是我们把MyErr 转换成了 error 

通过 error 去 和 nil 比较大小， error 是 interface， 是否被当做nil ，要看两方面 。1.data 2.type。图中 data 是 nil， type 不是 nil 是 MyErr
```



## interface内部结构

```
 go 中关于interface 解释   
 
 // interface 是值类型， 只是两个字段 _type 和 data 都是 unsafe.pointer, 都是引用
 
// interface ===> iface (包含方法), eface (不包含方法)

 // 并不是说 iface 或者 eface 是引用类型，而是因为
  iface {
   itab uintptr
   data uintptr
  }

  eface {
   _type uintptr
   data uintptr
  }
  
    
 // 包含的第一个属性是 引用类型，类似 slice 。

 // 本质上和 map -》 hmap 不一样， map 取得是 hmap 的地址类型，上面的验证可以通过 unsafe 去验证
 
 
 interface 和 nil 比较，需要 动态类型和动态值 都是 nil， 才能判断  == nil
 
  例子：

  func main() {
      var v interface{}
      v = (*int)(nil)
      fmt.Println(v == nil)
  }

  // 输出 false， 因为 type 是 *int

  func main() {
      var data *byte
      var in interface{}
      fmt.Println(data, data == nil)
      fmt.Println(in, in == nil)
       in = data
     fmt.Println(in, in == nil)
   }

  // 输出
  true
  true
  false
  
  
  
   https://mp.weixin.qq.com/s/F9II4xc4yimOCSTeKBDWqw

  煎鱼的这篇文章很好的诠释了工作中可能不小心遇到的问题。

  对于结构体指针类型，我们可以直接用nil 比较。

  对于 interface （error） 类型，我们要结合 type 和 value 一起比较


  以上interface 的比较，就遇到之前在做 redis ErrNil 错误的对比，注意ErrNil 是我们申明的一个变量，方法里面我们使用这个变量的时候，直接return 这个包级别的变量，不能根据 message 再重新生成，否则不等于 （临时生成的变量地址，还外抛， 野指针）（不要拿两个不同包中变量比较相等，即使名称一样）
```



![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/img/20211012182635.png)

```
 断言，是针对 接口的存在
 
 t, _ := m1.(int) // 安全断言，通过 comma 判断断言是否正确，如果断言错了， t 是该类型零值
  // 如果不用ok， 断言错了，会panic
  // 这个断言可以是 具体类型，也可以是 interface
  // 只能对接口进行断言 
```



```
go 中interface 容易遇到的问题：
  https://sanyuesha.com/2017/07/22/how-to-understand-go-interface/

  package main

  import "fmt"

  type Person struct {
      age int
  }

  func (p Person) Elegance() int {
      return p.age
  }

  func (p *Person) GetAge() {
      p.age += 1
  }

  func main() {
      // p1 是值类型
      p := Person{age: 18}
      // 值类型 调用接收者也是值类型的方法
  fmt.Println(p.Elegance())
  
  // 值类型 调用接收者是指针类型的方法
 p.GetAge()
 }
 
  // ----------------------
  
 // p2 是指针类型
 p2 := &Person{age: 100}
  
 // 指针类型 调用接收者是值类型的方法

 // 指针类型 调用接收者也是指针类型的方法
 p2.GetAge()
 
 
 
  // 上面那块代码虽然是能直接执行的  !!!! （因为语法糖， 而不是 生成了对应的发放）

  // 但是如果我们定义了 interface 类型， 我们的 p 变量是没有拥有 *p 的方法的。 虽然  *p 包含了 所有 p 的方法

  // p 能自动生成*p 的方法， 但是 *p 的方法不能自动生成 p。

  // interface 之所以对应的 struct 不能执行 *struct 方法，有种解释是， interface 转换后的变量不能寻址，所以 struct{} 不能像语法糖那样 直接调用 &struct{} 的方法
```



```
发现引用类型 ,如果是 nil 也能调用对应 结构的方法，只是不能调用对应结构 中的字段

type Name struct {
    N1 string
  }

// grpc 中经常封装这种方法
  func (n  *Name) Get() string{
     if n == nil {
      return ""
    }
     return n.N1

}

 var t *Name 
t.Get() (方法) 是正常执行的，我们再调用字段的时候，用 t.Get() 可以省去判断 t 是否是nil.

```



```
  var c error
   c = errors.New("11")
  
   c1, cok := c.(interface{})
  
   fmt.Println(cok)
   fmt.Println(reflect.TypeOf(c1).Name())
   
   今天遇到断言的一个小问题。 interface 再断言成 interface 。其实是可以的，不会报错， ok 那边代表的是但凡有实体type 类型，返回的就是true， 否则就是 false。
   
```



```
 // 下面这种编译不通过
 // 原因就是 地址类型没法生成 结构体类型
 // 但如果 student 本身是能调用的，主要转换成interface 后不能再调用回来
 package main
  
  import (
  	"fmt"
  )
  
  type People interface {
  	Speak(string) string
  }
  
  type Student struct{}
  
  func (stu *Student) Speak(think string) (talk string) {
  	if think == "bitch" {
  		talk = "You are a good boy"
  	} else {
  		talk = "hi"
  	}
  	return
  }
  
  func main() {
  	var peo People = Student{}
  	think := "bitch"
  	fmt.Println(peo.Speak(think))
  }
```



```
*简单介绍下 interface 的应用场景

1.interface 没有方法的。eface， 这种一般是 比如 atomic.Value 我们明确知道存进去的是什么类型，准确断言出来。因为go 是强类型语言，如果不用interface， 需要对atomic.Int ， atomic.bool 都各定义个方法， 比较麻烦。

2.interface 没有方法的，iface， 这种比如error， 定义一个接口类型，类似协议，后面比如某些外部内容想接到我们平台上来，必须满足一定的规范。
 
```

