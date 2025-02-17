## Go sync.Map 源码详解

Go 中的 map 类型是非并发安全的，所以 Go 就在 sync 包中提供了 map 的并发原语 sync.Map，允许并发操作，本文就带大家详细解读下 sync.Map 的原理。

使用示例
sync.Map 提供了基础类型 map 的常用功能，使用示例如下：

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
var s sync.Map

// 存储键值对
s.Store("name", "hrw")
s.Store("age", 30)
s.Store("location", "Shenzhen")

// 读取值
if value, ok := s.Load("name"); ok {
    fmt.Println("name:", value)
}

// 删除一个键
s.Delete("age")

// 遍历 sync.Map
s.Range(func(key, value interface{}) bool {
    fmt.Printf("%s: %s\n", key, value)
    return true// 继续遍历
 })
}
```

sync.Map 是一个结构体，和其他常见的并发原语一样零值可用。Store 方法用于存储一个键值对，Load 方法根据给定的键读取对应的值，Delete 方法则可以删除一个键值对，Range 方法则用于遍历 sync.Map 中存储的所有键值对。

### 源码解读
以在 sync.Map 文档中看到其定义和实现的 exported 方法：

https://pkg.go.dev/sync@go1.23.0#Map

```go
type Map
    func (m *Map) Clear()
    func (m *Map) CompareAndDelete(key, old any) (deleted bool)
    func (m *Map) CompareAndSwap(key, old, new any) (swapped bool)
    func (m *Map) Delete(key any)
    func (m *Map) Load(key any) (value any, ok bool)
    func (m *Map) LoadAndDelete(key any) (value any, loaded bool)
    func (m *Map) LoadOrStore(key, value any) (actual any, loaded bool)
    func (m *Map) Range(f func(key, value any) bool)
    func (m *Map) Store(key, value any)
    func (m *Map) Swap(key, value any) (previous any, loaded bool)
```
sync.Map 结构体不仅提供了 map 的基本功能，还提供了如 CompareAndSwap、LoadOrStore 等复合操作，随着我们对源码的深入探究，你将学会这里所有方法的原理和用途。

### 核心结构体
sync.Map 主要由以下几个核心结构体组成：

https://github.com/golang/go/blob/go1.23.0/src/sync/map.go

```go
type Map struct {
    mu   sync.Mutex               // 保护 dirty map 的互斥锁
    read atomic.Pointer[readOnly] // 只读 map，原子操作无需加锁
    dirty map[any]*entry          // 可变 map，包含所有数据，操作时需要加锁
    misses int                    // 记录 read map 的 miss 计数
}

type readOnly struct {
    m       map[any]*entry // 存储数据的只读 map
    amended bool           // 是否有新的 key 只存在于 dirty map
}

type entry struct {
    p atomic.Pointer[any] // 存储值的指针，支持原子操作
}

var expunged = new(any) // 用于标记从 dirty map 中被删除的元素
```
sync.Map 结构体字段不多，不过 sync.Map 源码整体上来说逻辑还是比较复杂的，并且细节很多，先大致梳理下 sync.Map 的设计思路，方便后续理解源码。

我们可以粗略的将 sync.Map 的设计类比成缓存系统，当我们要读取 map 中某个 key 对应的值时，sync.Map 会优先从缓存中读取，如果缓存未命中，则继续从 DB 中读取。sync.Map 还会在特定条件满足的情况下同步缓存和 DB 中的数据。

有了缓存系统的类比，我再为你讲解 sync.Map 的源码，就更容易理解了。

- 在 sync.Map 结构体中，mu 字段是一个互斥锁，用于保护对 dirty 属性的操作;
- dirty 字段是一个 map 类型，可以类比为缓存系统中的 DB，所以理论上 dirty 中的数据应该是全量的;
- read 字段是一个原子类型的指针，指向 readOnly 结构体，readOnly 内部其实也是使用 map 来存储数据，你可以将 read 类比为缓存；
- misses 字段则用于计数，记录缓存未命中的次数，当我们要读取 map 中某个 key 对应的值时，优先从 read 读取，如果 read 中不存在，则 misses 值加 1，然后继续从 dirty 中读取，当 misses 值达到某个阈值时，sync.Map 就会将 dirty 提升为 read。
- readOnly 结构体中 m 字段用于存储 map 数据；amended 用于标记是否有新的 key 只存在于 dirty 中，而不在 read 中。
- 当我们要读取 sync.Map 中某个 key 对应的值时，会优先从 read.m 中读取，如果 read.m 中不存在，read.amended 则决定了是否需要继续从 dirty 中读取
- read.amended 为 false 时，表示 read.m 和 dirty 中的数据一致，所以无需继续从 dirty 中读取；
- read.amended 为 true 时，表示 read.m 中的数据要落后于 dirty，read.m 存在数据缺失，所以可以尝试继续从 dirty 中读取
- 如果 dirty 中还是没有，才最终确定 sync.Map 中确实没有此 key。


未完待续，，，

> 参考：https://mp.weixin.qq.com/s/Bh3j9zGccXjm6jfQL6JxmA  