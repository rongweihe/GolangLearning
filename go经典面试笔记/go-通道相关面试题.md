
## 通道相关重点

- [什么是csp概念](#什么是csp概念)
- [channel底层的数据结构是什么](#channel底层的数据结构是什么)
- [向channel发生数据的过程是什么的](#向channel发生数据的过程是什么的)
- [向channel接收数据的过程是什么的](#向channel接收数据的过程是什么的)
- [关闭一个channel的过程是怎样的](#关闭一个channel的过程是怎样的)
- [操作channel的情况总结](#操作channel的情况总结)
- [如何优雅的关闭channel](#如何优雅的关闭channel)
- [channel的发送和接收元素本质是什么](#channel的发送和接收元素本质是什么)
- [不带缓冲的通道和带缓冲的通道区别是什么](#不带缓冲的通道和带缓冲的通道区别是什么)
- [channel的应用都有哪些](#channel的应用都有哪些)


### 什么是CSP概念

> Do not communicate by sharing memory; instead, share memory by communicating.

不要通过共享内存来通信，而要通过通信来实现内存共享。

这就是 Go 的并发哲学，它依赖 CSP 模型，基于 channel 实现。

CSP 经常被认为是 Go 在并发编程上成功的关键因素。CSP 全称是 “Communicating Sequential Processes”，这也是 Tony Hoare 在 1978 年发表在 ACM 的一篇论文。论文里指出一门编程语言应该重视 input 和 output 的原语，尤其是并发编程的代码。

Go 是第一个将 CSP 的这些思想引入，并且发扬光大的语言。仅管内存同步访问控制（原文是 memory access synchronization）在某些情况下大有用处，Go 里也有相应的 sync 包支持，但是这在大型程序很容易出错。

Go 一开始就把 CSP 的思想融入到语言的核心里，所以并发编程成为 Go 的一个独特的优势，而且很容易理解。

大多数的编程语言的并发编程模型是基于线程和内存同步访问控制，Go 的并发编程的模型则用 goroutine 和 channel 来替代。Goroutine 和线程类似，channel 和 mutex (用于内存同步访问控制)类似。

Goroutine 解放了程序员，让我们更能贴近业务去思考问题。而不用考虑各种像线程库、线程开销、线程调度等等这些繁琐的底层问题，goroutine 天生替你解决好了。

Channel 则天生就可以和其他 channel 组合。我们可以把收集各种子系统结果的 channel 输入到同一个 channel。Channel 还可以和 select, cancel, timeout 结合起来。而 mutex 就没有这些功能。

Go 的并发原则非常优秀，目标就是简单：尽量使用 channel；把 goroutine 当作免费的资源，随便用。

### channel底层的数据结构是什么

底层数据结构需要看源码，版本为 go 1.9.2：
```go
type hchan struct {
	// chan 里元素数量
	qcount   uint
	// chan 底层循环数组的长度
	dataqsiz uint
	// 指向底层循环数组的指针
	// 只针对有缓冲的 channel
	buf      unsafe.Pointer
	// chan 中元素大小
	elemsize uint16
	// chan 是否被关闭的标志
	closed   uint32
	// chan 中元素类型
	elemtype *_type // element type
	// 已发送元素在循环数组中的索引
	sendx    uint   // send index
	// 已接收元素在循环数组中的索引
	recvx    uint   // receive index
	// 等待接收的 goroutine 队列
	recvq    waitq  // list of recv waiters
	// 等待发送的 goroutine 队列
	sendq    waitq  // list of send waiters

	// 保护 hchan 中所有字段
	lock mutex
}
```

关于字段的含义都写在注释里了，再来重点说几个字段：

- buf 指向底层循环数组，只有缓冲型的 channel 才有。
- sendx，recvx 均指向底层循环数组，表示当前可以发送和接收的元素位置索引值（相对于底层数组）。
- sendq，recvq 分别表示被阻塞的 goroutine，这些 goroutine 由于尝试读取 channel 或向 channel 发送数据而被阻塞。
- waitq 是 sudog 的一个双向链表，而 sudog 实际上是对 goroutine 的一个封装

```go
type waitq struct {
	first *sudog
	last  *sudog
}
```
lock 用来保证每个读 channel 或写 channel 的操作都是原子的。例如，创建一个容量为 6 的，元素为 int 型的 channel 数据结构如下 ：

![](https://golang.design/go-questions/channel/assets/0.png)

### 向channel发生数据的过程是什么的？
todo
### 向channel接收数据的过程是什么的？
tood

### 关闭一个channel的过程是怎样的？
```go
func closechan(c *hchan) {
	// 关闭一个 nil channel，panic
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	// 上锁
	lock(&c.lock)
	// 如果 channel 已经关闭
	if c.closed != 0 {
		unlock(&c.lock)
		// panic
		panic(plainError("close of closed channel"))
	}

	// …………

	// 修改关闭状态
	c.closed = 1

	var glist *g

	// 将 channel 所有等待接收队列的里 sudog 释放
	for {
		// 从接收队列里出队一个 sudog
		sg := c.recvq.dequeue()
		// 出队完毕，跳出循环
		if sg == nil {
			break
		}

		// 如果 elem 不为空，说明此 receiver 未忽略接收数据
		// 给它赋一个相应类型的零值
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		// 取出 goroutine
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, unsafe.Pointer(c))
		}
		// 相连，形成链表
		gp.schedlink.set(glist)
		glist = gp
	}

	// 将 channel 等待发送队列里的 sudog 释放
	// 如果存在，这些 goroutine 将会 panic
	for {
		// 从发送队列里出队一个 sudog
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}

		// 发送者会 panic
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, unsafe.Pointer(c))
		}
		// 形成链表
		gp.schedlink.set(glist)
		glist = gp
	}
	// 解锁
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
	// 遍历链表
	for glist != nil {
		// 取最后一个
		gp := glist
		// 向前走一步，下一个唤醒的 g
		glist = glist.schedlink.ptr()
		gp.schedlink = 0
		// 唤醒相应 goroutine
		goready(gp, 3)
	}
}
```
close 逻辑比较简单，对于一个 channel，recvq 和 sendq 中分别保存了阻塞的发送者和接收者。关闭 channel 后，对于等待接收者而言，会收到一个相应类型的零值。对于等待发送者，会直接 panic。所以，在不了解 channel 还有没有接收者的情况下，不能贸然关闭 channel。

close 函数先上一把大锁，接着把所有挂在这个 channel 上的 sender 和 receiver 全都连成一个 sudog 链表，再解锁。最后，再将所有的 sudog 全都唤醒。

唤醒之后，该干嘛干嘛。sender 会继续执行 chansend 函数里 goparkunlock 函数之后的代码，很不幸，检测到 channel 已经关闭了，panic。receiver 则比较幸运，进行一些扫尾工作后，返回。这里，selected 返回 true，而返回值 received 则要根据 channel 是否关闭，返回不同的值。如果 channel 关闭，received 为 false，否则为 true。这我们分析的这种情况下，received 返回 false。

### 操作channel的情况总结

总结一下操作 channel 的结果：

- 操作	｜ nil channel	 ｜ closed channel	｜ not nil, not closed channel
- close	 ｜ panic	 ｜ panic	 ｜ 正常关闭
- 读 <- ch	 ｜ 阻塞	｜ 读到对应类型的零值	｜ 阻塞或正常读取数据。缓冲型 channel 为空或非缓冲型 channel 没有等待发送者时会阻塞
- 写 ch <-	｜ 阻塞	 ｜ panic	｜ 阻塞或正常写入数据。非缓冲型 channel 没有等待接收者或缓冲型 channel buf 满时会被阻塞

总结一下，发生 panic 的情况有三种：
- 向一个关闭的 channel 进行写操作；
- 关闭一个 nil 的 channel；重复关闭一个 channel。
- 读、写一个 nil channel 都会被阻塞。

### 如何优雅的关闭channel

关于channel的使用，有几点不方便的地方
- 在不改变channel自身状态的情况下，无法获知一个channel是否关闭
- 关闭一个closed channel会导致 panic，所以，如果关闭channel的一方在不知道channel是否处于关闭状态下就贸然是很危险的事情
- 向一个已经closed channel的channel发生数据也会导致 panic，所以，如果向channel发送数据的一方不知道channel是否已经关闭就贸然去向channel发送数据是很危险的事情

有两个不那么优雅关闭channel的方法
- 使用 defer-recover 机制
- 使用 sync.Once 来保证只关闭一次

根据实际情况，有下面四种情况

- 一个 sender，一个 receiver
- 一个 sender，多个 receiver
- 多个 sender，一个 receiver
- 多个 sender，多个 receiver

对于 1 和 2，直接从 sender 端关闭就好了。
对于 3 情况，优雅方案是增加一个传递关闭信号的 channel，receiver 通过信号 channel 下达关闭数据的指令，senders 监听到关闭信号之后，停止发送数据。代码如下：
```go
func main() {
	rand.Seed(time.Now().UnixNano())

	const Max = 100000
	const NumSenders = 1000

	dataCh := make(chan int, 100)
	stopCh := make(chan struct{})

	// senders
	for i := 0; i < NumSenders; i++ {
		go func() {
			for {
				select {
				case <- stopCh:
					return
				case dataCh <- rand.Intn(Max):
				}
			}
		}()
	}

	// the receiver
	go func() {
		for value := range dataCh {
			if value == Max-1 {
				fmt.Println("send stop signal to senders.")
				close(stopCh)
				return
			}

			fmt.Println(value)
		}
	}()

	select {
	case <- time.After(time.Hour):
	}
}
```
对于 4 情况：增加一个中间人（注意需要声明成一个带缓冲型的通道），多个 receiver 都向它发送关闭 data 的请求，中间人收到第一个请求之后，就会直接下达关闭 data 指令。
```go
func main() {
	rand.Seed(time.Now().UnixNano())

	const Max = 100000
	const NumReceivers = 10
	const NumSenders = 1000

	dataCh := make(chan int, 100)
	stopCh := make(chan struct{})

	// It must be a buffered channel.
	toStop := make(chan string, 1)

	var stoppedBy string

	// moderator
	go func() {
		stoppedBy = <-toStop
		close(stopCh)
	}()

	// senders
	for i := 0; i < NumSenders; i++ {
		go func(id string) {
			for {
				value := rand.Intn(Max)
				if value == 0 {
					select {
					case toStop <- "sender#" + id:
					default:
					}
					return
				}

				select {
				case <- stopCh:
					return
				case dataCh <- value:
				}
			}
		}(strconv.Itoa(i))
	}

	// receivers
	for i := 0; i < NumReceivers; i++ {
		go func(id string) {
			for {
				select {
				case <- stopCh:
					return
				case value := <-dataCh:
					if value == Max-1 {
						select {
						case toStop <- "receiver#" + id:
						default:
						}
						return
					}

					fmt.Println(value)
				}
			}
		}(strconv.Itoa(i))
	}

	select {
	case <- time.After(time.Hour):
	}

}
```

### channel的发送和接收元素本质是什么
- 就是说，channel的发送和接收操作本质上都是“值的拷贝”。


### 不带缓冲的通道和带缓冲的通道区别是什么

#### 无缓冲通道（Unbuffered Channel）
特点

- 同步阻塞：发送和接收操作必须同时进行，否则会阻塞。
- 容量为0：通道不存储任何数据，直接传递值。
- 原子性：操作是原子性的，确保数据传递的安全性。

阻塞条件

- 发送阻塞：发送方尝试发送数据时，如果接收方未准备好接收，则发送方会阻塞。
- 接收阻塞：接收方尝试接收数据时，如果发送方未准备好发送，则接收方会阻塞。

应用场景

- 严格的同步场景：适用于需要确保数据即时传递的场景，例如生产者-消费者模式中，生产者生成的数据需要立即被消费者处理。
- 信号传递：用于协程之间的同步信号传递。
- 简单的同步机制：可以用于一个协程等待另一个协程完成任务。


#### 有缓冲通道（Buffered Channel）
特点

- 异步通信：发送操作可以在接收操作之前进行，只要缓冲区未满。
- 缓冲区管理：通道可以存储一定数量的数据，缓冲区大小在创建时指定。
- 减少阻塞：通过缓冲区暂存数据，减少发送和接收操作的阻塞。

阻塞条件
- 发送阻塞：当缓冲区满时，发送操作会阻塞，直到缓冲区有空间。
- 接收阻塞：当缓冲区为空时，接收操作会阻塞，直到缓冲区有数据。

应用场景
- 生产者-消费者模式：适用于生产者和消费者速率不一致的场景，缓冲区可以暂存数据，避免生产者因等待消费者而被阻塞。
- 批量数据处理：可以暂存数据，直到收集到足够数量的数据再进行处理。
- 高并发场景：减少协程之间的阻塞，提高并发性能。

性能对比
- 无缓冲通道：同步通信，阻塞时间较长，适用于需要严格同步的场景。
- 有缓冲通道：异步通信，减少阻塞时间，适用于需要提高并发性能的场景。

总结
- 无缓冲通道：适用于需要即时处理数据和严格同步的场景。
- 有缓冲通道：适用于需要暂存数据、减少阻塞和提高并发性能的场景。


### channel的应用都有哪些

Channel 和 goroutine 的结合是 Go 并发编程的大杀器。
而 Channel 的实际应用也经常让人眼前一亮，通过与 select，cancel，timer 等结合，它能实现各种各样的功能。接下来，我们就要梳理一下 channel 的应用。


Channel 和 goroutine 的结合是 Go 并发编程的大杀器。
而 Channel 的实际应用也经常让人眼前一亮，通过与 select，cancel，timer 等结合，它能实现各种各样的功能。接下来，我们就要梳理一下 channel 的应用。

#### 停止信号

“如何优雅地关闭 channel”那一节已经讲得很多了，这块就略过了。

channel 用于停止信号的场景还是挺多的，经常是关闭某个 channel 或者向 channel 发送一个元素，使得接收 channel 的那一方获知道此信息，进而做一些其他的操作。

#### 任务定时
与 timer 结合，一般有两种玩法：实现超时控制，实现定期执行某个任务。

有时候，需要执行某项操作，但又不想它耗费太长时间，上一个定时器就可以搞定：

```go
select {
	case <-time.After(100 * time.Millisecond):
	case <-s.stopc:
		return false
}
```

等待 100 ms 后，如果 s.stopc 还没有读出数据或者被关闭，就直接结束。这是来自 etcd 源码里的一个例子，这样的写法随处可见。
定时执行某个任务，也比较简单,每隔 1 秒种，执行一次定时任务.
```go
func worker() {
	ticker := time.Tick(1 * time.Second)
	for {
		select {
		case <- ticker:
			// 执行定时任务
			fmt.Println("执行 1s 定时任务")
		}
	}
}
```
#### 解耦生产方和消费方
服务启动时，启动 n 个 worker，作为工作协程池，这些协程工作在一个 for {} 无限循环里，从某个 channel 消费工作任务并执行：
```go
func main() {
	taskCh := make(chan int, 100)
	go worker(taskCh)

    // 塞任务
	for i := 0; i < 10; i++ {
		taskCh <- i
	}

    // 等待 1 小时 
	select {
	case <-time.After(time.Hour):
	}
}

func worker(taskCh <-chan int) {
	const N = 5
	// 启动 5 个工作协程
	for i := 0; i < N; i++ {
		go func(id int) {
			for {
				task := <- taskCh
				fmt.Printf("finish task: %d by worker %d\n", task, id)
				time.Sleep(time.Second)
			}
		}(i)
	}
}
```
5 个工作协程在不断地从工作队列里取任务，生产方只管往 channel 发送任务即可，解耦生产方和消费方。

程序输出：
```c
finish task: 1 by worker 4
finish task: 2 by worker 2
finish task: 4 by worker 3
finish task: 3 by worker 1
finish task: 0 by worker 0
finish task: 6 by worker 0
finish task: 8 by worker 3
finish task: 9 by worker 1
finish task: 7 by worker 4
finish task: 5 by worker 2
```


#### 控制并发数
有时需要定时执行几百个任务，例如每天定时按城市来执行一些离线计算的任务。但是并发数又不能太高，因为任务执行过程依赖第三方的一些资源，对请求的速率有限制。这时就可以通过 channel 来控制并发数。

下面的例子来自《Go 语言高级编程》：

```go
var limit = make(chan int, 3)

func main() {
    // …………
    for _, w := range work {
        go func() {
            limit <- 1
            w()
            <-limit
        }()
    }
    // …………
}
```
构建一个缓冲型的 channel，容量为 3。接着遍历任务列表，每个任务启动一个 goroutine 去完成。真正执行任务，访问第三方的动作在 w() 中完成，在执行 w() 之前，先要从 limit 中拿“许可证”，拿到许可证之后，才能执行 w()，并且在执行完任务，要将“许可证”归还。这样就可以控制同时运行的 goroutine 数。

这里，limit <- 1 放在 func 内部而不是外部，原因是：

- 如果在外层，就是控制系统 goroutine 的数量，可能会阻塞 for 循环，影响业务逻辑。

- limit 其实和逻辑无关，只是性能调优，放在内层和外层的语义不太一样。

- 还有一点要注意的是，如果 w() 发生 panic，那“许可证”可能就还不回去了，因此需要使用 defer 来保证。
> 参考：[golang.design](https://golang.design/go-questions)
