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
本文将着重与接口所涉及的转换问题。此外，将引入新的构造---类型断言和类型转换。


假设我们有两个接口类型变量，我们要把一个变量分配给另一个：
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