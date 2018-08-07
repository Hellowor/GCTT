# Go 接口（第二部分）
> 类型声明与类型转换

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