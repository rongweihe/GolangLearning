
## 通道相关重点

- [什么是CSP概念](#什么是CSP概念)
- [channel底层的数据结构是什么](#channel底层的数据结构是什么)
- [向channel发生数据的过程是什么的](#向channel发生数据的过程是什么的)
- [向channel接收数据的过程是什么的](#向channel接收数据的过程是什么的)
- [关闭一个channel的过程是怎样的](#关闭一个channel的过程是怎样的)
- [操作channel的情况总结](#操作channel的情况总结)
- [如何优雅的关闭channel](#如何优雅的关闭channel)
- [channel的发送和接收元素本质是什么](#channel的发送和接收元素本质是什么)
  

### 什么是CSP概念

### channel底层的数据结构是什么

### 向channel发生数据的过程是什么的
### 向channel接收数据的过程是什么的

### 关闭一个channel的过程是怎样的
### 操作channel的情况总结

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


> 参考：[golang.design](https://golang.design/go-questions)
