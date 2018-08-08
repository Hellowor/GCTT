# Go 接口（第二部分）
> 类型断言和类型转换

 有的时候值需要被转换成不同的类型。转换被检查在汇编时和整个转换机制在
 [之前部分](https://medium.com/golangspec/conversions-in-go-4301e8d84067)
 介绍过。总之，它看起来像这样的

```go
type T1 struct {
    name string
}

type T2 struct {
    name string
}

func main() {
    vs := []interface{}{T2(T1{"foo"}), string(322), []byte("abł")}
    for _, v := range vs {
        fmt.Printf("%v %T\n", v, v)
    }
}
```
输出：
```
{foo} main.T2
ł string
[97 98 197 130] []uint8
```
Golang具有可分配规则，在一些情况下允许赋值一个变量给不同的类型：
```go
type T struct {
    name string
}

func main() {
    v1 := struct{ name string }{"foo"}
    fmt.Printf("%T\n", v1) // struct { name string }
    var v2 T
    v2 = v1
    fmt.Printf("%T\n", v2) // main.T
}
```
本文将着重于接口所涉及的转换问题。此外，将引入新的构造---类型断言和类型转换。


假设我们有两个接口类型变量，我们要把一个变量赋值给另一个：
```go
type I1 interface {
    M1()
}

type I2 interface {
    M1()
}

type T struct{}

func (T) M1() {}

func main() {
    var v1 I1 = T{}
    var v2 I2 = v1
    _ = v2
}
```

即使I2有其他接口方法，而I1没有那么这些接口仍然彼此实现。方法的顺序也不重要。值得记住
的是，方法集不必是相等的（源代码）：:
```go
type I1 interface {
    M1()
    M2()
}

type I2 interface {
    M1()
}

type T struct{}

func (T) M1() {}
func (T) M2() {}

func main() {
    var v1 I1 = T{}
    var v2 I2 = v1
    _ = v2
}
```
这样的代码运行仅因为第3种赋值而有趣。I2类型的值实现I1，因为其方法集是来自I1的方法的子集。
如果不是这样，编译器会相应地作出反应（源代码）：
```go
type I1 interface {
    M1()
}

type I2 interface {
    M1()
    M2()
}

type T struct{}

func (T) M1() {}

func main() {
    var v1 I1 = T{}
    var v2 I2 = v1
    _ = v2
}
```
以上代码由于错误而无法编译：
```
main.go:18: cannot use v1 (type I1) as type I2 in assignment:
	I1 does not implement I2 (missing M2 method)
```
我们已经看到了两个接口类型的情况。前面列出的可赋值的第三种情况也适用于右侧值是具体类型（非接口类型）并且实现接口（源代码）：
```go
type I1 interface {
    M1()
}

type T struct{}

func (T) M1() {}

func main() {
    var v1 I1 = T{}
    _ = v1
}
```
当接口类型值需要赋值给具体类型的变量时，它是如何工作的? ：
```go
type I1 interface {
    M1()
}

type T struct{}

func (T) M1() {}

func main() {
    var v1 I1 = T{}
    var v2 T = v1
    _ = v2
}

```
这不起作用，抛出错误不能使用V1（类型I1）作为赋值中的类型T：需要类型断言。这就是类型断言步骤的所在。

只有当GO编译器能够检查其正确性时才能进行转换。编译时无法验证的场景如下：

### 1.接口类型 → 确定类型
```go
type I interface {
    M()
}

type T struct {}
func (T) M() {}

func main() {
    var v I = T{}
    fmt.Println(T(v))
}
```
它给出编译错误，不能将V（I型）转换为T型：需要类型断言。这是因为编译器不知道这种隐式转换是否有效，因为实现接口I的任何值都可以分配给变量V。
### 2.接口类型 → 接口类型，右边的方法集不是左边的类型设置的方法子集：
```go
type I1 interface {
    M()
}

type I2 interface {
    M()
    N()
}

func main() {
    var v I1
    fmt.Println(I2(v))
}
```
编译器输出:
```
main.go:16: cannot convert v (type I1) to type I2:
	I1 does not implement I2 (missing N method)
```
原因如前面所说。如果I2的方法集是I1的方法集的子集，那么编译器将在编译阶段知道它。这是不同于，而且这种转换只有在运行时才是可能的。
>
它不是严格的转换，而是类型断言和类型切换，允许检查/检索接口类型值的动态值，或者甚至将接口类型值转换为不同接口类型的值。
## 类型断言
类型断言表达式的语法如下：
```go
v.(T)
```
其中V为接口类型，T为抽象类型或具体类型。
### Concrete type
首先，让我们看看它是如何与非接口类型一起工作的
```go
type I interface {
    M()
}

type T struct{}

func (T) M() {}

func main() {
    var v1 I = T{}
    v2 := v1.(T)
    fmt.Printf("%T\n", v2) // main.T
}
```
类型断言中所指定的类型必须实现V1接口--I。它在编译阶段被验证（源代码）：
```go
type I interface {
    M()
}

type T1 struct{}

func (T1) M() {}

type T2 struct{}

func main() {
    var v1 I = T1{}
    v2 := v1.(T2)
    fmt.Printf("%T\n", v2)
}
```
成功编译这样的代码是不可能的，因为不可能的类型断言错误。变量V1不能保存为类型T2，因为T2类型不满足接口I，变量V1只能存储实现I的类型的值。
编译器不知道在运行程序的过程中，变量V1中存储了什么样的值。类型断言是从接口类型值中检索动态值的一种方法。但是如果动态类型的V1不匹配T会发生什么？（源代码）：
```go
type I interface {
    M()
}

type T1 struct{}

func (T1) M() {}

type T2 struct{}

func (T2) M() {}

func main() {
    var v1 I = T1{}
    v2 := v1.(T2)
    fmt.Printf("%T\n", v2)
}
```
程序将会panic:
```
panic: interface conversion: main.I is main.T1, not main.T2
```
### 多值返回（将不会panic）
类型断言可以用多值形式使用，其中附加的第二个值是布尔值，表示断言是否成立。如果不是，第一个值类型T为空（源代码）：
```go
type I interface {
    M()
}

type T1 struct{}

func (T1) M() {}

type T2 struct{}

func (T2) M() {}

func main() {
    var v1 I = T1{}
    v2, ok := v1.(T2)
    if !ok {
        fmt.Printf("ok: %v\n", ok) // ok: false
        fmt.Printf("%v,  %T\n", v2, v2) // {},  main.T2
    }
}
```
这次不会panic，布尔常数作为第二值返回可用于检查断言是否成功。
### 接口
在上述情况下，类型断言中使用的类型是具体的。Golang也允许传递接口类型。它检查动态值是否满足
期望的接口并返回这样的接口类型值的值。在接口间转换时，传递到类型断言的接口的方法集不必是V类
型方法集（源代码）的子集：
```go
type I1 interface {
    M()
}

type I2 interface {
    I1
    N()
}

type T struct{
    name string
}

func (T) M() {}
func (T) N() {}

func main() {
    var v1 I1 = T{"foo"}
    var v2 I2
    v2, ok := v1.(I2)
    fmt.Printf("%T %v %v\n", v2, v2, ok) // main.T {foo} true
}
```
如果接口不满足，则返回接口的零值-nil（源代码）：
```go
type I1 interface {
    M()
}

type I2 interface {
    N()
}

type T struct {}

func (T) M() {}

func main() {
    var v1 I1 = T{}
    var v2 I2
    v2, ok := v1.(I2)
    fmt.Printf("%T %v %v\n", v2, v2, ok) // <nil> <nil> false
}
```
>
在处理接口类型时，也支持单值类型的断言。

### nil
当v是nil，类型断言会失败。无论T是接口还是具体类型（源代码）:
```go
type I interface {
    M()
}

type T struct{}

func (T) M() {}

func main() {
    var v1 I
    v2 := v1.(T)
    fmt.Printf("%T\n", v2)
}
```
如上所述，这样的程序会panic:
```
panic: interface conversion: main.I is nil, not main.T
```
前面介绍了当v为nil时，多返回值断言可以防止panic。
## 类型转换
类型断言是一种方法，如果接口类型值的动态类型要么实现所需接口，要么与已通过的具体类型完全一致，则只需执行一次检查即可。
如果代码需要针对单个变量进行多个这样的测试，那么Golang具有比一系列类型断言更紧凑的结构，并且类似于传统的switch语句：
```go
type I1 interface {
    M1()
}

type T1 struct{}

func (T1) M1() {}

type I2 interface {
    I1
    M2()
}

type T2 struct{}

func (T2) M1() {}
func (T2) M2() {}

func main() {
    var v I1
    switch v.(type) {
    case T1:
            fmt.Println("T1")
    case T2:
            fmt.Println("T2")
    case nil:
            fmt.Println("nil")
    default:
            fmt.Println("default")
    }
}
```
语法类似于断言，但使用type关键字。当接口值为空时，输出为nil，但是我们将v的值替换为：
```go
var v I1 = T2{}
```
程序将打印T2。类型switch也适用于接口类型：
```go
var v I1 = T2{}
switch v.(type) {
case I2:
        fmt.Println("I2")
case T1:
        fmt.Println("T1")
case T2:
        fmt.Println("T2")
case nil:
        fmt.Println("nil")
default:
        fmt.Println("default")
}
```
它打印I2。如果将有更多匹配的接口类型的case，那么第一个将被使用（从上到下评估）。
 如果没有匹配到，那么什么也不会发生(源代码):
```go
type I interface {
    M()
}

func main() {
    var v I
    switch v.(type) {
    }
}
```
这个程序不会panic-它成功地执行。
### 每种情况多种类型
单switch case可以指定多个类型，用逗号分隔。如果要对多个类型的同一块进行评估，则可以避免这种代码复制:
```go
type I1 interface {
    M1()
}

type T1 struct{}

func (T1) M1() {}

type T2 struct{}

func (T2) M1() {}

func main() {
    var v I1 = T2{}
    switch v.(type) {
    case nil:
            fmt.Println("nil")
    case T1, T2:
            fmt.Println("T1 or T2")
    }
}
```
This one prints T1 or T2 and good since the dynamic type of v at the moment when guard i3s evaluated is T2.
这个打印 T1 or T2，
### default case
This case is similar to good old switch statement. It’s used when no matches have been found (source c0de):
```go
var v I
switch v.(type) {
default:
        fmt.Println("fallback")
}
```
### short variable declaration
So far we’ve seen type switches where guard has the syntax:
```go
v.(type)
```
where v is an expression like variable’s identifier. Additionally short variable declaration can be used there (source code):
```go
var p *T2
var v I1 = p
switch t := v.(type) {
case nil:
         fmt.Println("nil")
case *T1:
         fmt.Printf("%T is nil: %v\n", t, t == nil)
case *T2:
         fmt.Printf("%T is nil: %v\n", t, t == nil)
}
```
It prints *main.T2 is nil: true so the type of t is the type from case clause. If more than one type is specified in single clause then t’s type is the same as type of v (source code):
```go
var p *T2
var v I1 = p
switch t := v.(type) {
case nil:
         fmt.Println("nil")
case *T1, *T2:
         fmt.Printf("%T is nil: %v\n", t, t == nil)
}
```
This one outputs *main.T2 is nil: false. Variable t is of interface type because it’s not nil but points to nil pointer (part I explains when interface type value is nil).
### duplicates

Types specified in case clauses must be unique (source code):
```go
switch v.(type) {
case nil:
    fmt.Println("nil")
case T1, T2:
    fmt.Println("T1 or T2")
case T1:
    fmt.Println("T1")
}
```
Attempt to compile such code will end up with an error duplicate case T1 in type switch.
### optional simple statement
Guard can be preceded with simple statement like another short variable declaration (source code):
```go
var v I1 = T1{}
switch aux := 1; v.(type) {
case nil:
    fmt.Println("nil")
case T1:
    fmt.Println("T1", aux)
case T2:
    fmt.Println("T2", aux)
}
```
This program prints T1 1. Additional simple statement can be used not matter if the guard is in the form of short variable declaration or not.
Click ❤ below to help others discover this story. Please follow me if you want to get updates about new posts or boost work on future stories.
## Resources
[The Go Programming Language Specification ](https://golang.org/ref/spec)
[Interfaces in Go (part I)](https://medium.com/golangspec/interfaces-in-go-part-i-4ae53a97479c)
[Assignability in Go](https://medium.com/golangspec/assignability-in-go-27805bcd5874)
[Conversions in Go](https://medium.com/golangspec/conversions-in-go-4301e8d84067)
[“Simple statement” notion in Go](https://medium.com/golangspec/simple-statement-notion-in-go-b8afddfc7916)
---

via: https://medium.com/golangspec/interfaces-in-go-part-ii-d5057ffdb0a6

作者：[Michał Łowicki](https://medium.com/@mlowicki)
译者：[Hellowor](https://github.com/Hellowor)
校对：[校对者ID](https://github.com/校对者ID)

本文由 [GCTT](https://github.com/studygolang/GCTT) 原创编译，[Go 中文网](https://studygolang.com/) 荣誉推出