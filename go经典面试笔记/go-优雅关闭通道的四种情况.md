# golang 中应该如何优雅地关闭 channel？

根据 sender 和 receiver 的个数，分下面几种情况：

- 一个 sender，一个 receiver
- 一个 sender， M 个 receiver
- N 个 sender，一个 reciver
- N 个 sender， M 个 receiver



## 一个 sender，一个 receiver 
在这种情况下，由 sender 负责关闭 channel 是最简单和自然的方式。因为 sender 知道什么时候不再发送数据，所以可以在发送完所有数据后关闭 channel。 

```go
package main

import (
	"fmt"
	"sync"
)

// 在这种情况下，由 sender 负责关闭 channel 是最简单和自然的方式。因为 sender 知道什么时候不再发送数据，所以可以在发送完所有数据后关闭 channel。
func oneSendOneRecv() {
	ch := make(chan int)
	//sender
	go func() {
		for i := 0; i < 8; i++ {
			ch <- i
		}
		close(ch) // 发送完所有数据之后关闭 channel
	}()

	//receiver
	for i := range ch {
		fmt.Printf("[1]Receiver%d\n", i)
	}
}
```

## 一个 sender， M 个 receiver
同样，由唯一的 sender 负责关闭 channel。当 sender 关闭 channel 后，所有的 receiver 都会收到关闭信号，从而正常退出。
```go
func oneSendMoreRecv() {
	ch := make(chan int)
	var wg sync.WaitGroup
	//sender
	go func() {
		for i := 0; i < 8; i++ {
			ch <- i
		}
		close(ch) // 发送完所有数据之后关闭 channel
	}()
	// M 个 receiver
	const M = 3
	wg.Add(M)
	for i := 0; i < M; i++ {
		go func(id int) {
			defer wg.Done()
			for num := range ch {
				fmt.Printf("[2]Receiver %d received: %d\n", id, num)
			}
		}(i)
	}

	wg.Wait()
}

```
## N 个 sender，一个 reciver
这种情况下，需要使用一个额外的 done channel 来通知所有的 sender 已经完成了数据的发送，同时使用一个 sync.WaitGroup 来等待所有 sender 退出，最后由一个协调者关闭主 channel。
```go
// 由于有多个 sender 和一个 receiver，所以需要使用一个额外的 channel 来协调多个 sender 和 receiver。
// 具体来说，我们可以使用一个额外的 done channel 来通知所有的 sender 已经完成了数据的发送
// 然后由 receiver 来关闭主 channel。
func MoreSendOneRecv() {
	const M = 3
	ch := make(chan int)
	done := make(chan struct{})
	var wg sync.WaitGroup

	// M 个 sender
	wg.Add(M)
	for i := 0; i < M; i++ {
		go func(id int) {
			defer wg.Done()
			for {
				select {
				case <-done:
					return
				case ch <- id:
					fmt.Printf("[3]Sender %d sent: %d\n", id, id)
				}
			}
		}(i)
	}

	// receiver
	go func() {
		for i := 0; i < 8; i++ {
			fmt.Printf("[3]Receiver received: %d\n", <-ch)
		}
		close(done) // 通知所有的 sender 已经完成了数据的发送
		wg.Wait()   // 等待所有 sender 退出
		close(ch)   // 关闭主 channel
	}()

	for range ch {
		fmt.Printf("[3]Receiver received: %d\n", <-ch)
	}
}
```

## N 个 sender， M 个 receiver
这种情况下，需要使用一个额外的 done channel 来通知所有的 sender 已经完成了数据的发送，

同时使用一个 sync.WaitGroup 来等待所有 sender 退出，最后由一个协调者关闭主 channel。

```go
func MoreSendMoreRecv() {
	const M = 3
	const N = 2
	ch := make(chan int)
	done := make(chan struct{})
	var wgSend sync.WaitGroup
	var wgRecv sync.WaitGroup

	wgSend.Add(M)
	for i := 0; i < M; i++ {
		go func(id int) {
			defer wgSend.Done()
			for {
				select {
				case <-done:
					return
				case ch <- id:
					fmt.Printf("[4]Sender %d sent: %d\n", id, id)
				}
			}
		}(i)
	}

	wgRecv.Add(N)
	for i := 0; i < N; i++ {
		go func(id int) {
			defer wgRecv.Done()
			for num := range ch {
				fmt.Printf("[4]Receiver %d received: %d\n", id, num)
			}
		}(i)
	}

	//协调者
	go func() {
		for i := 0; i < 8; i++ {
			// 这里可以根据实际情况决定何时停止
			fmt.Printf("[4]Coordinator received: %d\n", <-ch)
		}
		close(done)   // 通知所有的 sender 已经完成了数据的发送
		wgSend.Wait() // 等待所有 sender 退出
		close(ch)     // 关闭主 channel
	}()
	wgRecv.Wait()
}
func main() {
	oneSendOneRecv()
	oneSendMoreRecv()
	MoreSendOneRecv()
	MoreSendMoreRecv()
}
```

## 运行结果
