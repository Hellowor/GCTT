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
So for we’ve seen how cases where two interface types are involved. 3rd case of assignability listed before applies also when right-side value is of concrete type (non-interface type) and implements an interface (source code):
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

How it works though when interface type value needs to be assigned to variable of concrete type? (source c0de)：

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

This doesn’t work and throws an error cannot use v1 (type I1) as type T in assignment: need type assertion. This is where type assertion steps in…

Conversion can be done only if Go compiler is able to check its correctness. Scenarios where it isn’t verifiable at compile-time are as follows:
1.interface type → concrete type (source code)
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
It gives a compilation error cannot convert v(type I) to type T: need type assertion. It’s because the compiler doesn’t know if such implicit conversion is valid since any value implementing interface I can be assigned to variable v.
2.interface type → interface type, where the method set of the right side isn’t a subset of the method set from the type on the left (source code)：
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
Compiler’s output:
```
main.go:16: cannot convert v (type I1) to type I2:
	I1 does not implement I2 (missing N method)
```
The reason is as before. If the method set of I2 would be a subset of method set of I1 then compiler would know it at compilation phase. It’s different though and such conversion is possible only while run-time.
>
It won’t be strictly conversion but type assertion and type switch allowing to check / retrieve dynamic value of interface type value or even convert interface type value to value as of different interface type.