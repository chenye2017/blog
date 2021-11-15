---
title: go 小知识点
date: 2020-07-14 17:57:32
tags: [go]
categories: go
---

读源码学到的新的语法，补充自己对go的更多了解

<!--more-->

# 断言

今天在看别人代码的时候发现了一个 interface.([]int) 新用法，以前不知道，这个其实就是断言 assert， 其实php 中也经常用到

1.go 中目前我接触到断言,取map值的时候 ，a, ok :=  map["name"],  虽然没有这个ok 也是能正确运行的，比如 a := map["name"]

2.interface 断言， interface.([]int), 转换成[]int类型

3.断言失败会取断言类型的默认值，如果断言失败还是不知道原因可以用reflect.TypeOf获取断言的真正类型。断言失败的时候经常就是胯类型断言，比如你知道一个类型[]map[string]string, 但你收到这个值的时候不能直接断言interface.([]map[string]string), 而是应该 interface.([]interface{}),  for range 每个值断言map[string]string.** 所以能用结构体就用结构体接受吧，要不然每层断言很辛苦  ，曾经断言60行的代码，用结构体 不到10行 就接受了，还不用处理一堆的断言错误**。

4.关于类型 interface{} 兼容 string， 但不代表 []interface{} 兼容 []string

```
Can I convert a []T to an []interface{}? ¶
Not directly. It is disallowed by the language specification because the two types do not have the same representation in memory. It is necessary to copy the elements individually to the destination slice. This example converts a slice of int to a slice of interface{}:

t := []int{1, 2, 3, 4}
s := make([]interface{}, len(t))
for i, v := range t {
    s[i] = v
}

还有另一种方式实现转换

[]interface{}{"11", "112"}

notice : []interface{}{v...} , 这样不行的，这样转换的interface slice， 数量总是 1
```



5.今个写代码遇到件事， 对于  int 1, 我用 string(int 1) -----> 想直接得到结果，这样是不行的，还是老老实的用 strconv.Itoa,  string 应该直接用在字符类型，比如 rune  []byte 这类 



# Slice

奇怪的现象

1.最近用slice 总容易写一个bug， 就是make 的时候给定大小，然后append slice， 这样会导致一直往后面插入slice ，而不是从0 开始修改slice 的值。原因就是 make 的时候 len 和 cap 都是 给定的值，每个位置都有自己的默认值，append 的元素已经没地方放了，只会动态扩容 slice 来容纳给更多的内容。

2.slice 和 map 虽然是地址类型，但是我们 for range 的时候改变值并不会修改自身，原因就是因为for range 的时候是copy 。copy 之后dst 和src 指向的底层 array 不一样，导致指向 dst 和  src 的slice 随意修改都不影响对方。

3.copy 也有需要注意的地方，就是 src 的len > dst 的len ，并不会copy 全，也就是copy的时候并不会动态扩容

4.切片的传递属于引用传递，我们日常使用切片也是引用。所以我们在用切片相互赋值的时候，修改某一个可能会影响两一个。原因就是slice 底层指向了同一个数组

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/Image.png)



5.我们在什么情况下使用切片不会影响到之前赋值的切片呢，就是在切片动态扩容，改变切片地址指向的时候，比如 append 的时候。因为我们一般切片创建用的 make，make slice  len == cap ，当我们append 的时候必然扩容。

6.切片是由3个属性决定， 指针 ，len， cap 一般情况下我们 len 和 cap 都一样， 所以会导致我们觉得 指针改变，切片就变化，指针不变，切片就不变。

```
func m(modify[]int) []int{
	fmt.Printf("\n %p", modify)
	modify = append(modify, 13)
	fmt.Printf("\n %p \n", modify)
	fmt.Println(len(modify), cap(modify), modify)
	return  modify
}

func main()  {
	a4 := make([]int, 2,10)

	fmt.Printf("%p \n", a4)
	m(a4)
	fmt.Printf("%p", a4)
	fmt.Println(a4, len(a4), cap(a4))
	}
// 这个例子a4 没有变化，虽然 没有扩容 ，指针没变，一般情况下如果m 方法中我们 m[0] = 12, 这样a4 必然改变、
// 这个例子中 ，虽然没有扩容，但是 cap 和 len 没有通过函数返回，导致即使 a4在 m 方法中没有扩容，指针没改变，m 方法中 append 13 也没有影响 a4 。 仅在m 方法中a4 被影响了一小会,因为受影响的 len 没有返回，所以退出m方法， a4又还原了。这也是 append 必须有接受值的原因才能改变slice
```

6.我们var 定义 []int 的时候，[]int 是nil， 所以我们不能给他赋值。但是我们make 的时候，虽然此时没有开辟内存空间，但是point 是有值的。！！！所以如果用var 定义的变量赋值会报address 不存在错误，如果用make 就不会。

7.for range 的时候只循环len 的内容，不循环cap 的内容





# channel定义

之前看过对channel 的定义最好不要在全局，之前不知道为啥原因，当时因为想做缓冲channel， 而以为var 没法做，所以一直没用全局channel

```
var channel = make([]chan task, 10)
// 10 个的缓冲， 然后是 task 组成的chan 的slice

slice 用make 和 自身 定义的区别
make([]int, 10)  [0 0 0 ..]
[]int{}  就是nil
```



# goroutine的意义

把一个任务分成很多部分，每个任务完成的周期很短。多个任务中我们可以通过channel 进行通信。如果我们通过单个channel ，在进行io的时候会阻塞的， 所以我们需要多个channel 来配合多个goroutine。多个goroutine消耗多个channel, 取数据的时候，可以把channel 传入goroutine当中，来消耗特定的channel。投递数据的时候咋办？当往特定的channel 中投递任务，因为go不像php 那样可以拼变量名，我们可以先把多个channel 放在一个数组中，然后通过数组index 去取特定的channel。



go  slice 结构， （指向array 的指针，len, cap）

（go 中slice 的改动会及时没有用 & 也会影响自身）

```
func handle(a []string)  {
	a[0] = "bb"
}

func main() {
	b := []string{"name", "str", "aa"}
	
	c := make([]string, len(b))
	copy(c, b) // copy 的话就不会影响,注意copy 的时候一定要len 一样，否则会copy不全
	
	// 这种直接赋值的话， c 的改动会直接影响b
	// c := b
	
	handle(c)
	fmt.Println(b, c)

	a := []string{"name", "str", "aa"}
	sort.Strings(a)

	fmt.Println(a)
}
```





(copy 方便数组的拷贝，不影响原始数组的变动)

（arr := [...]int{1,2,3,4}, arr[1:2:3], start 1 end 2  len 3, 索引的位置）



// go 中的引用类型

引用类型和原始的基本类型恰恰相反，它的修改可以影响到任何引用到它的变量。在Go语言中，引用类型有切片、map、接口、函数类型以及`chan`。

引用类型之所以可以引用，是因为我们创建引用类型的变量，其实是一个标头值，标头值里包含一个指针，指向底层的数据结构，当我们在函数中传递引用类型时，其实传递的是这个标头值的副本，它所指向的底层结构并没有被复制传递，这也是引用类型传递高效的原因。



//  go 中经常这样，类型别名

```
type Duration int64
```



// go 可变参数

可以变参数，可以是任意多个。我们自己也可以定义可以变参数，可变参数的定义，在类型前加上省略号…即可。



// 组合类型

```
type user struct {
	name string
	email string

}

type admin struct {
	user
	level string
}

func main() {
	ad:=admin{user{"张三","zhangsan@flysnow.org"},"管理员"}
	fmt.Println("可以直接调用,名字为：",ad.name) // 能运行
	fmt.Println("也可以通过内部类型调用,名字为：",ad.user.name) // 能运行
	fmt.Println("但是新增加的属性只能直接调用，级别为：",ad.level)
}

```



// 访问权限

```
type user struct {
	Name string
}

type Admin struct {
	user
}

user 无法被导出，因为 小写，类似严格访问类型
```



// race 检测对共享变量的修改

```
go build -race 10.go

10.exe
```



// sync 包真的是解决并发问题的一个优点

```
package main

import (
	"fmt"
	"runtime"
	"sync"
)

var (
	count int32
	wg    sync.WaitGroup
	mutex sync.Mutex
)

func main() {
	wg.Add(2) // 计数器
	go incCount()
	go incCount()
	wg.Wait() // 如果信号量不到0 main 进程就一直堵塞
	fmt.Println(count)
}

func incCount() {
	defer wg.Done()
	for i := 0; i < 2; i++ {
		mutex.Lock() // 只能有一个goroutine 进来
		value := count
		runtime.Gosched()
		value++
		count = value
		mutex.Unlock()
	}
}
```



// 很经典的一个关于获取三个url 最快速的方式

```
func mirroredQuery() string {
    responses := make(chan string, 3)
    go func() { responses <- request("asia.gopl.io") }()
    go func() { responses <- request("europe.gopl.io") }()
    go func() { responses <- request("americas.gopl.io") }()
    return <-responses // return the quickest response
}
func request(hostname string) (response string) { /* ... */ }
```



# json 序列化的小问题

json 协议没有int 类型，只有 number 类型。int 都会被解析成 float64， 注意！！

上面的描述有bug ，今天一个str  json 反序列化的时候很成功，啥时候会出现上面问题，通过 interface{} 断言的时候。



json  序列化的时候如果没有这个值，就不给客户端 （比如我们更倾向于返回空对象，而不是一个完整对象，然后值都是空的），可以使用json tag 中的 omitempty 



json 序列化的时候如果不想要这个，可以直接 - （比如密码这类我们不想暴露给客户端，我们只是我们后端struct 使用，并不需要给客户端）



json 我们也可以用 int 接受 string 类型 （需要注意的是 我们再次 json序列化的时候 还是 string ）



今天遇到一个很好用json 方法，[就当做json 序列化和 反序列化前的操作](https://medium.com/@xfstart07/go-json-%E7%9A%84%E7%BC%96%E7%A0%81%E5%92%8C%E8%A7%A3%E7%A0%81-e689522a1f1f)



# url处理

最近有一个很恶性的需求，就是解析别人填入的url， 再添加一些想要的参数，组成新的url 返回，

```

u, err := url.Parse(href)  // 解析成一个url 对象，这个url 对象有个 string() 方法，可以直接输出这个 href
// 获取 query
query := u.RawQuery  
// 解析	query
queryMap, err := url.ParseQuery(query) // 返回一个values 对象，是一个 map[string][]string
// go 中一个参数默认对应一个slice， 如果是单个参数，就是slice 的第一位啦
// 这个values 有很多方法，如果想改参数试试 set， 注意参数是[]string, 否则会整体覆盖哦
// 注意map 赋值是不会被修改的，所以还是调用他的方法吧
queryMap.Encode()
u.RawQuery = queryMap.Encode()
// 组成新的query 参数,再赋值下就能得到新的url
```



# http 请求

## get

golang 发送http 请求没有 php 那么直接，其实php 也没有那么直接，毕竟curl 那么一大串，只是php 的curl 面向过程，看起来是那么自然，从上而下

```
import "net/http"
...
resp, err := http.Get("http://wwww.baidu.com")
//
上面的方法编程平时应该用不到，因为我们的请求至少需要一个超时时间吧
```

```
import "net/http"
...
clt := http.Client{}
resp, err := clt.Get("http://wwww.baidu.com")

// 不是那么丝滑，需要用客户端发送请求，但又是那么的符合生活，像我们平时请求就应该有个客户端
```

```
// 本质
req, err := http.NewRequest("GET", "http://www.baidu.com", nil)

//然后http.client 结构体的 Do 方法
//http.DefaultClient可以换为另外一个http.client
resp, err := http.DefaultClient.Do(req)
```

评论 ：Go的get请求面上有好几种请求方式，实则只有一种：

1、使用`http.NewRequest`函数获得`request`实体

2、利用`http.client`结构体的`Do`方法，将`request`实体传入`Do`方法中。



## post

```
import (
"net/http"
"net/url"
)
...
data := url.Values{"start":{"0"}, "offset":{"xxxx"}}
body := strings.NewReader(data.Encode())
resp, err := http.Post("xxxxxxx", "application/x-www-form-urlencoded", body)

//
注意这个body 是个io.reader. 通过这个strings.NewReader 转变来，显然，这个方法的参数是 strings
// 当我们的content-type 是 application/ json, 我们这块就不能用 data.Encode 了而是应该用 json .Marc

```

```
import (
"net/http"
"net/url"
)
...
var r http.Request
r.ParseForm()
r.Form.Add("xxx", "xxx")
body := strings.NewReader(r.Form.Encode())
http.Post("xxxx", "application/x-www-form-urlencoded", body)

// 这种form 请求之前在js 里面用的很经常，简单哇
```

```
import (
"net/http"
"net/url"
)
...
data := url.Values{"start":{"0"}, "offset":{"xxxx"}}
http.PostForm("xxxx", data)

// golang 封装的postForm 
```

当然上面的方法本质上也是用client 发出来的

然后client 本质也是依靠 newRequest

```
import (
"net/http"
"net/url"
)
...

data := url.Values{"start":{"0"}, "offset":{"xxxx"}}
body := strings.NewReader(data.Encode())

req, err := http.NewRequest("POST", "xxxxx", body)
req.Header.Set("Content-Type", "application/x-www-form-urlencoded")

clt := http.Client{}
clt.Do(req)
```

!!!notice

```
添加request header
net/http包没有封装直接使用请求带header的get或者post方法，所以，要想请求中带header，只能使用NewRequest方法。

import (
"net/http"

)
...

req, err := http.NewRequest("POST", "xxxxx", body)
//此处还可以写req.Header.Set("User-Agent", "myClient")
req.Header.Add("User-Agent", "myClient")

clt := http.Client{}
clt.Do(req)
```

有一点需要注意：在添加header操作的时候，`req.Header.Add`和`req.Header.Set`都可以，但是在修改操作的时候，只能使用`req.Header.Set`。

有一点需要注意：在添加header操作的时候，`req.Header.Add`和`req.Header.Set`都可以，但是在修改操作的时候，只能使用`req.Header.Set`。
这俩方法是有区别的，Golang底层Header的实现是一个`map[string][]string`，`req.Header.Set`方法如果原来Header中没有值，那么是没问题的，如果又值，会将原来的值替换掉。而`req.Header.Add`的话，是在原来值的基础上，再`append`一个值，例如，原来header的值是“s”，我后`req.Header.Add`一个”a”的话，变成了`[s a]`。但是，获取header值的方法`req.Header.Get`确只取第一个，所以，如果原来有值，重新`req.Header.Add`一个新值的话，`req.Header.Get`得到的值不变。

其实不止是header 会这样，query 参数也会这样。



## Response

```
import (
	"net/http"
	"net/url"
	"io/ioutil"
)
...
content, err := ioutil.ReadAll(resp.Body)
respBody := string(content)
```

获取返回值



## Sort

go sort 已经有人封装好包了，相比较自己写快排，冒泡的好处就是，这个包会根据效率自动选择合适的排序方式，我们需要做的就是实现sort 中的接口 （https://books.studygolang.com/The-Golang-Standard-Library-by-Example/chapter03/03.1.html）

```
Len()  获取要排序的slice 的len
Less() return s[i] < s[j]  默认倒序
Swap(i, j int) 交换  // s[i], s[j] = s[j],s[i]

sort.Sort([]object)
```

对于倒序，很简单的实践方式

```
sort.Sort(sort.Reverse([]object)) // 帮我们少写了好多代码
```

```
上面的用法得写一堆东西，还有简单的方式

直接调用  sort.Slice() 方法

```



## runtime.Caller

在调用公司组件的时候，发现log 输出信息有问题，错误行号和文件是上一层函数调用，而不是我想输出的地方的调用。问题就出在，runtime.Caller(skip), 这个参数，

```
for i:=0; i<=4; i++ {
		_, file, line, _ := runtime.Caller(i)
		fmt.Println(file,line,"=====",i)
	}

// 如果 caller(i)  i 是 0 的时候，就是当前文件的输出，因为这个方法被封装了，所以，不管我在项目  // 中哪个地方调用，每次输出内容都是一样的。
//   试试不调用组件，在单独文件中 runtime.Caller(i) 输出试试，就会发现当i = 0 时候这个方法的真  // 正含义。

// 看看beego 中获取行号和文件名的方法

func FILE() string {

    _, file, _, _ := runtime.Caller(1)

    return file

}

// __LINE__ returns the line number at which the function was invoked

func LINE() int {

    _, _, line, _ := runtime.Caller(1)

    return line

}

skip 1 ,仅仅跳出当前方法所在的文件，因为他仅仅封装了一层
	
```

https://studygolang.com/articles/3116, 这篇文章对 call 和 calls 方法讲解的比较细致

```
    pc := make([]uintptr, 1024)
    for skip := 0; ; skip++ {
        n := runtime.Callers(skip, pc)
        if n <= 0 {
            break
        }
        fmt.Printf("skip = %v, pc = %v\n", skip, pc[:n])
    }
    // 当我们调用 callers 方法的时候，pc 这个array 的容量一定要足够大，否则 n 一直不会渠道数据
    // runtime.FuncForPC(pc[skip]).FileLine(pc[skip]) , 这个方法比较好玩，从上面calles 获取   //到的指针连获取对应方法的方法名和 函数
```

https://colobu.com/2018/11/03/get-function-name-in-go/, 这篇文章也对上面两个方法做了详细的解释



## http状态码

429  ，限流了

499 ， 服务端返回的时间超出客户端设置的超时时间，到这客户端提前关闭



## fmt问题

平时为了打印结构体我们就用 fmt.printf("%+v"), 但当我们用[]*struct 的时候，这个打印就不好使了，会直接打印内存地址（我们为啥要用*struct， 因为遍历的时候 struct 不能修改值）

```

type Stu struct {
	Name string `json:"name"`
	Age string `json:"age"`
}

/*func (t *Stu)String()  string{
	return "11"
}*/

func main()  {
	str := `
[{"name":"12"}]
`
	var students []Stu

	err := json.Unmarshal([]byte(str), &students)

	for _, v := range students {
		v.Name = "xiugia111"
		v.Age = "1111"
	}

	fmt.Printf("%+v ", students[0], err)
}

// 为了让修改生效，我们必须让 students []*Stu
// 但这样之后我们打印 []*Stud 就是内存地址，我们需要哥 *Stu 定义 string() 方法，这样就能 fmt // 直接打印了
```



# Kafka使用







# Es使用 （olivere/elastic.v6，我们的es 是6.x 版本，所用的v6）

今天在使用es 的时候，想起了之前使用redis 的时候有not found 的判定，找了下 果然这个es 包中也有。只是这个es 包判定not found 用的是方法，原理是 http 请求的code，相比较redis， 感觉这个更靠谱些。



# Time

time 很实用的一个方法， time.Since 可以获取时间差



https://www.jianshu.com/p/f809b06144f7, 时间的很好的一个文章



timer :  延迟触发，只触发一次。 可以重置

ticker : 多次执行。

```
// timer
        d := time.Duration(time.Second*2)

        t := time.NewTimer(d)
        defer t.Stop()

        for {
                <- t.C

                fmt.Println("timeout...")
		// need reset
		t.Reset(time.Second*2), // 如果没有这个reset 就deadlock了
        }
 // ticker , 这个随便获取
  t := time.NewTicker(3*time.Second)
        defer t.Stop()

        fmt.Println(time.Now())
        time.Sleep(4*time.Second)


        for {

                select {

                case <-t.C:

                        fmt.Println(time.Now())

                }


        }
        
```





# Error

go 中 error 的处理还是没有总结出什么好的办法，目前想到的就是公用的方法，统一内部自己处理异常，当然也会抛出去，外层就可以不用处理了。

几种error 可以不处理的场景:

1.断言失败，有默认值的

2.调用方法，失败了error有返回，对于int float 这些都有默认的返回值。但对于一些结构体，还是要给默认值的，否则可能会是nil

3.思考一下我在项目中error 的处理方式。一般有问题，都在最底层的服务统一处理，因为一层层的往上抛，感觉日志记录可能重复，详见我的third 包调用 第三方的处理方式。但是我的service中， error 一般都没处理，一般是抛给了 controller 处理，因为我的service 虽然按道理也是公共的，但是调用方其实很少，当时考虑的也不太周到，所以都是交给controller 处理。



error 几种常见需要解决的问题

1.wrap ，我经常要 对错误信息添加，比如对发生错误时候的参数进行记录。

2.判断两个error 是否相等。我们需要注意的是 error 是地址类型 （可以通过 reflect 获取 ），所以两个error 完全相等必须是同一个变量，而不只是 new 里面的内容相等

```
var UserSexNil = errors.New("未查到该用户信息！")

// 然后我们通过 err == UserSexNil 判断
```



go 1.13 之后又很完美的方式解决我上面的蛋疼问题

```
fmt.Error("%w xxx", errors.New("old error")) // 这个新生成的error 就是old error + 自定义的xx信息， 但此时我们要判断是否是old error 生成，需要调用特殊的方法

errors.Is(newError, oldError)

当我们要断言某种错误的时候， 这个我其实用的很少，可以通过 errors.As() 来断言获取到对应的内容，因为要赋值

```

[这篇文章讲的很好](https://www.flysnow.org/2019/09/06/go1.13-error-wrapping.html)



引申: 看了这个想到了怎么对interface 类型执行，可以





# 静态文件引入

php 中对于静态文件引入很容易，定义个相对路径就好了，但是我们go 微服务不可以（不是所有的都不可以）， 我们go 的执行方式是

```
exec /app/"$@" -conf /app/config.yaml

// 这个 @是个二进制文件（也就是build之后生成的文件），我们在go 中获取到的执行路径就是 当前路径，所以一旦修改了这个二进制文件地址，这个相对路径就不生效了， 所以这块有两种方式
1. 相对路径改成绝对路径，我们把我们的文件也copy到文件中某个位置，然后用绝对路径
2. go build 只会对go 结尾文件生效，我们利用第三方包把我们的静态文件也打包成go 文件，这样我们就能取到这个文件了

// 对于静态文件打包成go 文件可以试一下这个包
go-bindata， 我其实就用到他的压缩，然后获取内容用的 Asset方法， 可以看一下生成的
```



# go.mod

indirect , 可能是我平时直接 go get 获取的，项目中用不到



go.sum  https://studygolang.com/articles/25658, 这篇文章讲的很好。 go.mod 中只是我们直接import 的文件，go.sum 中存在依赖的依赖。 那串hash 主要用来校验，防止别人修改代码。go 中发布是可以删除tag 修改代码再发布的，这样就会导致你之前依赖的代码可能被别人修改过，而你不知道，这时候go.sum 就会发挥用处。



go.sum 也是个文件，可能存在conflict， 而且他还是类似 log 日志的那种存在。如果冲突，我们可以都保留，或者保留最新的，反正只是个验证作用。



如果实在通不过，难道真的是包作者删除 tag，再重新打包？ ~~





# 一些思考

1.一些公共变量在框架init 的时候初始化，但这个变量应该放在哪？我开始是放在main 文件中，但有个很坑爹的时候，service 对于main 中变量引入不了，一来是因为循环依赖（比如 a 依赖b， b反过来也引入a），我们可以开一个单独的包，定义这个公共变量，并且 包含 自身init 函数

2.今天听大佬分享 公共pool 中对象被耗尽的问题，如何解决？ （最近接的案例就是我们的redis pool 被耗尽了）。1.池子里面对象尽可能给多，2.池子里面对象取出来尽快还回去。