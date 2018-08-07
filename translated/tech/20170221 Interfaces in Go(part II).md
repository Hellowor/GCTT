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
where v is of interface type and T is either abstract or concrete type.
### Concrete type
Let’s see how it works with non-interface types first
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
The type specified in type assertion must implement v1’s interface — I. It’s verified at compilation stage (source code):
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
Successful compilation of such code isn’t possible because of impossible type assertion error. Variable v1 cannot hold anything of type T2 since this type doesn’t satisfy interface I and variable v1 can only store values of types implementing I.
The compiler doesn’t know what kind of value is stored inside variable v1 over the course of running the program. Type assertion is a way to retrieve dynamic value from interface type value. But what will happen if the dynamic type of v1 doesn’t match T? (source code)
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
The program will panic:
```
panic: interface conversion: main.I is main.T1, not main.T2
```
### Multi-valued variant (do not panic please)
Type assertion can be used in multi-valued form where the additional, second value is a boolean indicating if assertion holds or not. If not the first value is zero-value of type T (source code):
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
This form does’t panic and boolean constant returned as a 2nd value can be used to check if assertion holds or not.
### Interface type
In all above cases type used in type assertions was concrete. Golang allows to also pass interface type. It checks if the dynamic value satisfies desired interface and returns value of such interface type value. In contract to conversion, method set of interface passed to type assertion doesn’t have to be a subset of v’s type method set (source code):
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
If interface is not satisfied then zero-value for interface is returned so nil (source code):
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
Single-valued variant of type assertion is also supported when dealing with interface types.

### nil
When v is nil then type assertion always fails. No matter if T is an interface or a concrete type (source code):
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
As stated such program will panic:
```
panic: interface conversion: main.I is nil, not main.T
```
Introduced earlier multi-value form protects against panic when v is nil —proof.
## 类型转换
Type assertion is a method to do a single check if dynamic type of an interface type value either implements desired interface or is identical to passed concrete type. If the code needs to do multiple such tests against a single variable then Golang has a construct which is more compact than a series of type assertions and resembles traditional switch statement:
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

The syntax is similar to type assertion but type keyword is used. Output is nil (source code) since the value of interface type value is nil but if we’ll set value of v instead:
```go
var v I1 = T2{}
```
then the program prints T2 (source code). Type switch works also with interface types (source code):
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

and it prints I2. If there would be more matching cases with interface types then the first one will be used (evaluated top-to-bottom). If there’re no matches then nothing happens (source code):
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
This program doesn’t panic — it successfully ends its execution.
### multiple types per case
Single switch case can specify more than one type, separated by comma. This way code duplication can be avoided if for multiple types the same block should be evaluated (source code):
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
This one prints T1 or T2 and good since the dynamic type of v at the moment when guard is evaluated is T2.
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