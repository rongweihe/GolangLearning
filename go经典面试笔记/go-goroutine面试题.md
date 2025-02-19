# 目录

- [goroutine和thread有什么区别](#goroutine和thread有什么区别)
- [什么是scheduler](#什么是scheduler)
- [为什么要scheduler](#为什么要scheduler)
- [深入理解gmp模型](#深入理解gmp模型)
- [gmp模型引入了p来解决gm模型的缺陷](#gmp-模型引入了-p-来解决-gm-模型的缺陷)

## goroutine和thread有什么区别

参考资料【How Goroutines Work】告诉我们可以从三个角度区别：内存消耗、创建与销毀、切换。

### 内存消耗

创建一个 goroutine 的栈内存消耗为 2 KB，实际运行过程中，如果栈空间不够用，会自动进行扩容。
创建一个 thread 则需要消耗 1 MB 栈内存，而且还需要一个被称为 “a guard page” 的区域用于和其他 thread 的栈空间进行隔离。

对于一个用 Go 构建的 HTTP Server 而言，对到来的每个请求，创建一个 goroutine 用来处理是非常轻松的一件事。
而如果用一个使用线程作为并发原语的语言构建的服务，例如 Java 来说，每个请求对应一个线程则太浪费资源了，很快就会出 OOM 错误（OutOfMemoryError）。

### 创建和销毀

Thread 创建和销毀都会有巨大的消耗，因为要和操作系统打交道，是内核级的，通常解决的办法就是线程池。
而 goroutine 因为是由 Go runtime 负责管理的，创建和销毁的消耗非常小，是用户级。

### 切换

当 threads 切换时，需要保存各种寄存器，以便将来恢复：

16 general purpose registers, PC (Program Counter), SP (Stack Pointer), segment registers, 16 XMM registers, FP coprocessor state, 16 AVX registers, all MSRs etc.

而 goroutines 切换只需保存三个寄存器：Program Counter, Stack Pointer and BP。

一般而言，线程切换会消耗 1000-1500 纳秒，一个纳秒平均可以执行 12-18 条指令。所以由于线程切换，执行指令的条数会减少 12000-18000。
 
Goroutine 的切换约为 200 ns，相当于 2400-3600 条指令。

因此，goroutines 切换成本比 threads 要小得多。

### 什么是scheduler
Go 程序的执行由两层组成：Go Program，Runtime，即用户程序和运行时。

它们之间通过函数调用来实现内存管理、channel 通信、goroutines 创建等功能。用户程序进行的系统调用都会被 Runtime 拦截，以此来帮助它进行调度以及垃圾回收相关的工作。

一个展现了全景式的关系如下图：

![](https://golang.design/go-questions/sched/assets/5.png)

### 为什么要scheduler
 Go scheduler 可以说是 Go 运行时的一个最重要的部分了。
 
Runtime 维护所有的 goroutines，并通过 scheduler 来进行调度。Goroutines 和 threads 是独立的，但是 goroutines 要依赖 threads 才能执行。

Go 程序执行的高效和 scheduler 的调度是分不开的。

### 深入理解GMP模型

#### 首先我们来思考一个问题，为什么需要协程？

其实答案也很明显：在多进程和多线程时代，CPU 内核负责调度，为系统提供并发处理能力，但也存在一些缺点：
- 资源消耗高：进程和线程的创建、切换和销毁都会消耗大量 CPU 资源。
- 内存占用大：每个线程约需 4MB 内存，大量线程会导致内存消耗过高。
- 应用层无法直接控制内核调度，只能通过减少线程创建和切换来优化性能。

这促生了协程的概念：用户级别的轻量线程。在 Go 中，协程被称为 goroutine，它主要解决了内核线程的两个“太重”问题：

- 创建和切换：goroutine 在用户态创建和切换，无需进入内核，开销比线程小得多。
- 内存占用：goroutine 的初始栈只有 2KB，栈空间可动态扩展或收缩，避免了内存浪费与栈溢出风险。
  
由于 goroutine 的轻量特性，Go 程序可以轻松创建成千上万个并发任务，而不用担心性能和内存问题。

#### 第二个问题：golang 可以在用户级别空间创建协程底层是靠什么实现的?

其实是虽然协程在用户态空间运行，但其底层实现依赖于Go运行时对操作系统线程（OS Threads）的管理和调度。

具体来说

Go协程运行在用户态空间，但它们并不是直接由操作系统调度的线程。相反，Go运行时通过一个用户态的调度器（Scheduler）来管理协程的执行，并将协程映射到操作系统线程上。
- 用户态协程（Goroutine）：轻量级的并发单元，由Go运行时管理。
- 操作系统线程（OS Thread）：由操作系统调度的线程，用于执行Go运行时的代码。
- 运行时调度器（Scheduler）：负责将协程分配到操作系统线程上执行。

Go运行时的调度机制

Go运行时的调度器是协程实现的核心，它通过以下组件协同工作：

a. G（Goroutine）
- 定义：G是协程的抽象表示，包含协程的栈、状态信息等。
- 特点：G是轻量级的，创建和销毁的开销极小。

b. M（Machine）
- 定义：M是操作系统线程的抽象表示，负责执行Go代码。
- 特点：M的数量通常远少于G的数量，由Go运行时动态管理。

c. P（Processor）
- 定义：P是执行Go代码的逻辑单元，负责从调度器获取G并分配给M执行。
- 特点：P的数量通常与CPU核心数相匹配，用于充分利用多核CPU的计算能力。

调度流程

Go运行时的调度器通过以下步骤管理协程的执行：

- 创建协程：当调用go func()时，Go运行时会创建一个新的G，并将其加入到调度队列中。
- 调度协程：调度器从队列中取出G，并将其分配给一个P。P将G绑定到一个M上，M开始执行G中的代码。

阻塞与唤醒

- 如果G在执行过程中阻塞（如等待I/O操作），调度器会将该G挂起，并从队列中取出另一个G继续执行。
- 当阻塞的G被唤醒时，调度器会将其重新加入到队列中，等待执行。

线程复用：

- 如果所有M都被阻塞，调度器会创建新的M来执行队列中的G。
- 如果某个M完成任务，调度器会将该M回收，用于执行其他G。

换一种方式来理解的话，在 Go 的 GMP 模型中：

- G（goroutine）：是用户态线程的抽象，可以在 M 上运行，存储于全局队列和 P 的本地队列（大小 256）中。
- M（Machine）：是操作系统线程的抽象，一个 M 代表一个线程，最多绑定一个 P。M 阻塞时会释放 P，允许 P 与其他空闲的 M 绑定，如果没有空闲的 M，则创建新的 M。
- P（Processor）：逻辑处理器，抽象代表 CPU 核心。P 的数量决定了程序的并行能力，可以通过 GOMAXPROCS 设置。每个 M 需要绑定一个 P 进行任务调度。
  
### GMP 模型引入了 P 来解决 GM 模型的缺陷：
- 无锁访问本地队列：P 保存 goroutine 的本地队列，M 优先从 P 的本地队列中取 G 执行，仅必要时访问全局队列，减少了锁竞争，提升了并发性。
- 优化数据局部性：新创建的 G 会优先放入创建它的 M 绑定的 P 的本地队列中，避免频繁的数据交互，提升内存使用效率。
- Hand Off 交接机制：M 阻塞时，将绑定的 P 和其任务交给其他 M，提升了资源利用率和并发度。

GMP 调度时机：介绍 GMP 的调度时机，主要分为正常调度、主动调度、被动调度和抢占调度四种情况

> 参考：https://golang.design/go-questions/sched/goroutine-vs-thread/ 
