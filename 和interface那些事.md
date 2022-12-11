Go 的interface 有两种

```
type A interface {
  speak()
}
// 数据结构 iface


var c interface{} 
// 数据结构 eface
```



带方法的interface 让静态语言go 有了动态语言的便利性， 但凡一个结构体实现了 这个interface 的所有方法，我们就可以认为这个结构体是interface 类型。

```
type A interface {
 speak()
 run()
}

type People struct {

}

func (p People) speak() {}  // 可以自动生成 引用类型同名方法

func (p *People) run() {} // 不能自动生成 people 的同名 run 方法

var _ A = People{}   // 编译失败
var _ A = &People{}  // 编译成功

var people = People{}
people.run() // 这样调用是成功的，注意和上面的区别，这个是因为有语法糖进行了转换, 传进去的还是地址类型
```



iface 的内存结构

```
type iface struct {
	tab  *itab   // 地址类型，占用 8字节
	data unsafe.Pointer // 实体变量地址，8字节
}

type itab struct {
	inter  *interfacetype  // interface 的详细信息
	_type  *_type  // 实体类型信息
	link   *itab
	hash   uint32 // copy of _type.hash. Used for type switches.
	bad    bool   // type does not implement interface
	inhash bool   // has this itab been added to hash?
	unused [2]byte
	fun    [1]uintptr // variable sized ， interface的方法
}
```



eface 的内存结构

```
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```



下面这个例子比较经典

```
package main

import "fmt"

type Coder interface {
	code()
}

type Gopher struct {
	name string
}

func (g Gopher) code() {
	fmt.Printf("%s is coding\n", g.name)
}

func main() {
	var c Coder
	fmt.Println(c == nil)
	fmt.Printf("c: %T, %v\n", c, c)

	var g *Gopher
	fmt.Println(g == nil)

	c = g
	fmt.Println(c == nil)
	fmt.Printf("c: %T, %v\n", c, c)
}

一开始，c 的 动态类型和动态值都为 nil，g 也为 nil，当把 g 赋值给 c 后，c 的动态类型变成了 *main.Gopher，仅管 c 的动态值仍为 nil，但是当 c 和 nil 作比较的时候，结果就是 false 了。

我们在 做 
var g Coder = Gopher{} 这步操作的时候，实际上做了一次类型转换，Gopher 转换成 coder
在 传入方法需要 interface 类型，或者返回方法 需要 interface 类型，都会做一次类型转换。
```



打印出接口的动态类型和值

```
package main

import (
	"unsafe"
	"fmt"
)

// 和iface 一样的内存结构
type iface struct {
	itab, data uintptr
}

func main() {
	var a interface{} = nil

	var b interface{} = (*int)(nil)

	x := 5
	var c interface{} = (*int)(&x)
	
	ia := *(*iface)(unsafe.Pointer(&a))
	ib := *(*iface)(unsafe.Pointer(&b))
	ic := *(*iface)(unsafe.Pointer(&c)) // 说明 interface 就是 iface 类型，而不是 *iface 类型

	fmt.Println(ia, ib, ic)

	fmt.Println(*(*int)(unsafe.Pointer(ic.data))) // 因为 data 本身就是地址，所以可以直接取值
}


// 最近写代码又发现一个问题

c := int(12)
var cer interface{}

cer = c
fmt.Println(cer) // 12
c = 13
fmt.Println(cer) // 12


caddr := int(12)
var cadder interface{}

cadder = &caddr
fmt.Println(cadder) // 12
caddr = 13
fmt.Print(cadder) // 13


反射发现，如果 原类型就是引用类型，data 就用这个类型
如果原类型是 值类型，data 就用这个类型的地址

```



Go 语言中不允许隐式类型转换，也就是说 `=` 两边，不允许出现类型不相同的变量。

`类型转换`、`类型断言`本质都是把一个类型转换成另外一个类型。不同之处在于，类型断言是对接口变量进行的操作。



