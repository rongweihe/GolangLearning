## 数组和切片

slice 的底层数据是数组，是对数组结构的封装，它描述了一个数组的片段，两者都可以通过下标来访问单个元素。

数组是定长的，长度定义好之后不能更改，在 Go 中，数组不太常见，因为长度定长限制了它的表达能力。而切片更加灵活可以动态扩容，切片的类型和长度无关的。

数组就是一片连续的内存， slice 实际上是一个结构体，包含三个字段：长度、容量、底层数组。

```go
type slice struct {
    array unsafe.Pointer // 元素指针
    len int //长度
    cap int //容量
}
```
slice 底层结构如下
![](https://golang.design/go-questions/slice/assets/0.png)

注意，底层数组是可以被多个 slice 同时指向的，因此对一个 slice 的元素进行操作是有可能影响到其他 slice 的。

> 【引申1】 [3]int 和 [4]int 是同一个类型吗？
不是。因为数组的长度是类型的一部分，这是与 slice 不同的一点。
