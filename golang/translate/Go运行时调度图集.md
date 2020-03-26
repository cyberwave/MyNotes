[Illustrated Tales of Go Runtime Scheduler](https://medium.com/@ankur_anand/illustrated-tales-of-go-runtime-scheduler-74809ef6d19b)

以 `goroutines`的形式进行的Go语言的并发对于写现代并发软件是一种非常方便的方法。但是您的Go程序是如何有效地运行这些 `goroutines` 的？

在这篇文章中，我们将深入了解如何通过从设计角度研究 Go 运行时调度程序如何实现此魔术，以及如何在性能调试过程中使用它来解释 Go 程序的调试程序跟踪信息。

所有的工程奇迹都不需要，因此，要理解**为什么需要 go 运行时调度程序**和**它是如何工作的**，让时间回溯到操作系统的历史，这将使我们深入了解问题，正如如果不了解问题的根源，就没有希望解决它。这就是历史所做的。

# 操作系统的历史

1. 单用户（无操作系统）
2. 批处理，单程序，运行完成
3. **多程序**

> *多程序的目的是共用CPU和I/O*

怎样做？

# **多批次**和**时间共享**。

**多批次**

* IBM OS/MFT(具有固定数量的任务多重编程)
* **IBM OS/MVT(具有可变数量的任务的多重编程)** -- 这里每个工作仅获得其所需的内存里。即，随着作业的进出，内存的分区发生变化。

**时间共享**

* 这是**可以在作业之间快速切换的多程序设计**。决定何时切换以及切换到哪个作业称为调度。

当前，大多数操作系统使用分时调度程序。

但是这些调度程序将调度哪个实体？

1. 不同的程序下在执行（进程）
2. 作为进程的子集存在于**CPU利用率（线程）的基本单位**

# 调度开销

![State variable of process and thread](https://miro.medium.com/max/823/1*x5r0JHcobOkrGckser7yIA.png)

所以，使用**一个包含多线程的进程效率更高**，因为进程创建即耗时又耗费资源。但是随后出现了多线程问题：C10K问题是成为了主要问题。

例如，如果你**定义调度器周期为10ms(毫秒)**,并且有2个线程，每个线程将分别获得5毫秒。如果你有5个线程，则每个线程占用2毫秒。但是如果有1000个线程会发生什么？给每个线程一个10 μs（微秒）的时间片？错，这样做很愚蠢，因为你将花费大量的时间进行上下文的切换，但是无法完成真正的工作。

你需要限制时间片的长度。在最后一种情况下，如果最小时间片为2ms并且有1000个线程，则调度程序周期需要增加到2s(秒)。如果有10000个线程，调度程序周期为20秒。在这个简单示例中，如果每个线程使用其全时切片 ，则所有线程运行一次需要20秒。因此，我们需要一些可以使并发成本降低而又不会造成过多开销的东西。

# 用户线线程

* 线程完全由运行时系统（用户级库）管理。
* 理想情况下，**快速高效：切换线程不比函数调用开销高多少**。
* 内核对用户级线程一无所知，并像对待单线程进程一样对其进行管理。

在Go中，我们通过他的名字“Goroutines"知道它（逻辑上）。

## Goroutine

![goroutine vs thread](https://miro.medium.com/max/489/1*1cPTpW115WWM6hPkNGSSqg.png)

*goroutine*是一个由Go运行时管理的**轻量级线程**（逻辑执行一个线程）。在函数调用`go add(a,b)`前添加`go`关键字来开始一个新的goroutine运行。

## 简单的goroutine游览

```go
func main() {
    var wg sync.WaitGroup
    for i := 0; i <= 10; i++ {
        wg.Add(1)
        go func(i int) {
        defer wg.Done()
        fmt.Printf("loop i is - %d\n", i)
        }(i)
    }
    wg.Wait()
    fmt.Println("Hello, Welcome to Go")
}
```

你能猜出上面代码片断的输出吗？

```shell
loop i is - 10
loop i is - 0
loop i is - 1
loop i is - 2
loop i is - 3
loop i is - 4
loop i is - 5
loop i is - 6
loop i is - 7
loop i is - 8
loop i is - 9
Hello, Welcome to Go
```

如果我们看一下输出的一种组合，那么马上会有两个问题

1. **11个goroutine如何并发运行**？
2. **11个goroutine的执行顺序是什么**?

![picture2](https://miro.medium.com/max/777/1*ZBwMnb_BWxQ-EzlhW_Fxcg.png)

这两个问题给我们带来一个问题

# 问题大纲

1. 如何在可用的CPU处理器上运行的多个OS线程分配这些goroutines.
2. 为了公平，这些goroutine应该以什么顺序执行。

接下来的讨论将主要围绕从设计角度解决Go运行时调度程序特有的这些问题。但是，与所有问题一样，我们的也需要定义一个明确的边界来处理。否则，问题陈述可能太含糊，无法进行结论性讨论。调度程序可能针对多个目标中的一个或多个，对我们来说，我们将自己限制在以下要求之内。

1. 应该是并行且可扩展且公平的。
2. 每个进程**应可扩展到数百万个goroutine**(10⁶)
3. 内存有效（RAM很便宜，但不是免费的）。
4. 系统调用不应导致性能下降（最大化[吞吐量](https://en.wikipedia.org/wiki/Throughput)，最小化[等待时间](https://en.wikipedia.org/wiki/Throughput)）。

因此，让我们开始为调度程序建模，以逐步解决这些问题。

# 1. 每个goroutine一个线程

使用用户级线程

局限性。

1. 并行且可扩展。
    * 并行（是）
    * 可扩展（不是真的）
2. 每个进程不能扩展到数百万个goroutine(10⁶)

# 2. M:N线程

混合线程

M个内核线程执行N个goroutine

![M kernel threads to execute N ”goroutine"](https://miro.medium.com/max/656/1*-AhqTn5KKXR1dqasGqzh4Q.png)

实际代码执行和并行性需要内核线程。但是它的创建很昂贵，所以我们将N个goroutines映射到M个内核线程。goroutine是Go代码，所以我们拥有所有的控制权。同时它也在用户空间中，因此它的创建很便宜。

但是由于操作系统对goroutine一无所知。每个goroutine有一个状态，以帮助**调度器根据goroutine状态知道要运行哪个goroutine**。与内核线程相比，状态信息很小，goroutiner的上下文切换变得非常快。

* **Running** - goroutine当前在内核线程上运行
* **Runnable** - goroutine等待内核线程来运行。
* **Blocked** - goroutine等待某些条件（例如，在通道，系统调用，互斥体等上阻塞）

![2 Thread running 2 goroutine at a time](https://miro.medium.com/max/1051/1*pTT97066_HBaBgM8OCoDrQ@2x.png)

因此，Go运行时调度器管理通过将N个协程复用到M个内核线程中来管理这些处于各种状态的goroutine。

# 简单的M:N调度器

在我们简单的M:N调度器中，我们有一个全局运行队列，某些操作将一个新的goroutine放入运行队列。M个内核线程可以访问调度器以从“运行队列”中获取goroutine来运行。多个线程尝试访问同一块内存，**我们将使用用于内存访问同步的互斥体来锁住此结构**。

![simple M:N scheduler](https://miro.medium.com/max/1053/0*fkqa9yA81-bnWOvz)

# 阻塞的goutine在哪里？

goroutine可能会阻塞的某些情况。

1. 在通道上发送接收
2. 网络IO
3. 系统调用阻塞
4. 计时器（Timers）
5. 互斥体

那么我们将这些阻塞的goroutine放到了哪里？将这些阻塞的goroutine放到哪里的设计决定基本上是围绕一个基本原理进行的：

**阻塞的goroutine 不应阻塞底层的内核线程！（为了避免上下文切换的开销）**

## 在通道操作中阻塞的goroutine

每个通道有一个***recvq(waitq)*** ，用于*存储试图从通道读取数据的阻塞的goroutine*。

**Sendq(waitq)**存储试图向通道发送数据的goroutine。（通道内部https://codeburst.io/diving-deep-into-the-golang-channels-549fd4ed21a8）

![Blocked Goroutine during Channel Operation](https://miro.medium.com/max/1128/0*KFYf5C7uO0-cLdlX)

通道自己将通道操作后的未阻塞goroutine放入运行队列。

![Unblocked goroutine after channel operation](https://miro.medium.com/max/1128/0*jMJEMNyItzDHNpMF)

# 系统调用呢？

首先，我们来研究**系统调用的阻塞**。系统调用会阻塞底层的内核线程，所以我们无法在该线程上调度任何其他的goroutine。

隐含阻塞系统调用可以降低并行度。

![Blocking System Call reduces the Parallelism level](https://miro.medium.com/max/1053/0*pzDsDmnI0nE8EMDf)

我们有工作要做，但是没有运行它，因此无法在M2线程上调度任何其他goroutine，进而导致了**CPU浪费**。

恢复并行度的方法是在进入系统调用时，我们可以唤醒另一个线程，该线程将从运行队列中选择可运行的goroutine。

![Way of restoring parallelism level](https://miro.medium.com/max/1129/0*cdTXGvqqIIPTtpyt)

在系统调用完成后，**我们有超额调度**。为了避免我们不会立即执行从阻塞系统调用中返回的goroutine。我们会将其放入调度器运行队列中。

![Avoiding Oversubscribed Scheduling](https://miro.medium.com/max/1129/0*_NiOWGDMyBgMtw1v)

> 所以当程序运行时，线程数大于内核数。尽管没有明确说明线程数大于内核数，并所有空闲线程也由运行时管理，以避免过多的线程。

**[初始设置10000线程](https://golang.org/pkg/runtime/debug/#SetMaxThreads)，如果超过它，程序将崩溃。**

**非阻塞系统调用** - 阻塞**[集成的运行轮询器](https://morsmachine.dk/netpoller)**上的goroutine，释放线程以运行另一个goroutine。

![](https://miro.medium.com/max/431/1*W6ZQSRgKf6PsiTlql7c-KA.png)

例如在如HTTP调用的非阻塞I/O情况下。由于资源尚未准备就绪，第一个系统调用，前一个工作流后面的，将不会成功，这将迫使Go使用网络轮询器并将goroutine停放。

函数 `net.Read` 的部分实现

```go
n, err := syscall.Read(fd.Sysfd, p)
if err != nil {
  n = 0
  if err == syscall.EAGAIN && fd.pd.pollable() {
    if err = fd.pd.waitRead(fd.isFile); err == nil {
      continue
    }
  }
```

当第一个系统调用完成，并明确指出资源尚未准备就绪，**goroutine将驻留，直到网络轮询器通知它资源已经就绪。在这种情况下，线程M将不会被阻塞**。

轮询器（Poller）将使用基于操作系统的 select **/kqueue/epoll/IOCP**来知道哪一个文件描述符已经准备就绪，当文件描述符已经准备好读或写，它将把goroutine放回到**运行队列**中。

> 还有一个 Sysmon OS线程，如果轮询时间不超过10ms，它将定期轮询网络，并将就绪的G添加到队列中。

基本上所有的goroutine都阻塞在操作

1. 通道
2. 互斥体
3. 网络IO
4. 计时器

有**某种队列**，帮助调度这些goroutine。

### 现在，运行时具有以下功能的调度器

* 它可以处理并行执行（多线程）
* 处理阻塞系统调用和网络I/O。
* 处理阻塞用户级(在通道上)调用

# 它这是不可伸缩的

![使用互斥体的全局运行队列](https://miro.medium.com/max/1053/0*F2xrNB_StWjSdp_N)

我们有一个拥有Mutex的全局运行队列，最终会遇到一些问题，例如

1. 缓存一致性保证的开销
2. 在创建，销毁和调度Goroutine G时进行激烈的锁竞争。

使用分布式调度器克服可伸缩性的问题。

# 分布式调度器 - 每个线程运行队列

![分布式运行队列调度器](https://miro.medium.com/max/1168/1*oET_Y5yGWznkfiWu8rAdHw@2x.png)

这样，我们可以看到的直接好处是每个线程本地运行队列现在都没有**互斥体**。仍然有一个带有互斥量全局运行队列，在特殊情况下使用。它不**影响可伸缩性**。

## 但是现在，我们有多个运行队列

1. 本地运行队列
2. 全局运行队列
3. 网络轮询器

> 我们应该从哪里运行下一个goroutine?

**在Go中，轮询顺序定义如下。**

1. 本地运行队列
2. 全局运行队列
3. 网络轮询器
4. 工作偷窥

即首先运行本地运行队列，如果为空则检查全局运行队列，然后检查网络轮询器，最后进行工作偷窃。我们已经对第1，2，3有个大概了理解。我们来看一下工作窃取

# 工作窃取

> 如果本地工作队列为空，尝试“从其他队列中窃工作”

![偷窃工作的概览](https://miro.medium.com/max/1168/1*GcY5wvGclTjWCHfQ07-FeQ@2x.png)

工作窃取解决当一个线程有太多的工作要做而另一个线程只是空闲的问题。在Go中，如果本地队列为空，窃取工作将尝试满足以下条件之一。

* 从全局队列中拉取工作
* 从网络轮询器中拉取工作
* 从其他本地队列中窃取工作

### 到目前为止，Go运行时有一个具有以下功能的调度器

* 它可以处理并行执行(多线程)
* 处理阻塞系统调用和网络I/O
* 处理阻塞用户级（通道上）调用
* 可扩展

## 但是它不是有效的。

还刻我们在阻塞系统调用中恢复并行度的方式吗？

![系统调用工作](https://miro.medium.com/max/867/1*0aHCGN8AEJN8HsdPhu51Sg.png)

它的含义是，在系统调用中我们可以有多个内核线程（可以是10或者1000），这可能会比内核数要多。我们最张望在下列司徒中产生了恒定的开销：

* **工作窃取**，它必须同时扫描所有的内核线程（理想情况下在goroutine中运行）本地运行队列，并且大多数都将为空
* **垃圾回收，内存分配器**都将遭受同样的扫描问题(https://blog.learngoprogramming.com/a-visual-guide-to-golang-memory-allocator-from-ground-up-e132258453ed)
* 使用M:P:N线程克服问题

# 3级M:P:N（3级调度器）线程 - 逻辑处理器P简介

P - 代表处理器，可以将其视为运行在线程上的本地调度程序；

![M:P:N 线程](https://miro.medium.com/max/1051/1*-EckdwGh4Nwij9WVZh7mCQ@2x.png)

> **逻辑进程P的数目始终是固定的。（默认为当前进程可以使用的逻辑CPU）**

我们将本地运行队列（Local Run Queue -  LRQ）放入固定数量的逻辑处理器（P）中。

![](https://miro.medium.com/max/1170/1*eyBZv_M-gaPiWWoL90DfHw@2x.png)

Go运行时将首先根据计算机的逻辑CPU数量（或根据请求）创建固定数量的逻辑处理器P。

> 每个Goroutine（G）将运行在分配给逻辑CPU（P）的OS线程（M）上。

所以现在我们以以下情况中**没有固定开销**：

* **工作窃取** - 只需要扫描固定数量的逻辑处理器（P）本地运行队列。
* **垃圾回收，内存分配器**也获得相同的好处

# 固定的逻辑处理器（P）的系统调用怎么办？

无论是否阻塞，Go通过将它们封装在运行时来优化系统调用

![](https://miro.medium.com/max/604/0*kqKDs8z2wphKWWMS)

阻塞SYSCALL方法封装在

**runtime.entersyscall(SB)**

**runtime.exitsyscall(SB)**

之间。

从字面上看，某些逻辑在进入系统调用前执行，而某些逻辑在退出系统调用之后执行。当进行系统调用阻塞时，此封装器将自动将P从线程M中分离，并允许另一个线程在其上运行。

![](https://miro.medium.com/max/1098/0*LskT9LC66ynnf4XA)

这使Go运行时可以有效地处理阻塞的系统调用，而无需增加运行队列。

# 一旦阻塞系统调用退出，会发生什么？

* 运行时尝试获取完全相同的P，然后继续执行
* 运行时尝试尝试在空闲列表中获取一个P并恢复执行。
* 运行时将goroutine放到全局队列中，并将**关联的M放回到空闲列表中**

## Spinning 线程和理想线程

当M2线程在系统调用返回后变成理想时，这个理想M2线程如何处理。理论上来讲，如果线程完成了它所需要做的工作，它将被OS销毁，然后安排其他进程中的线程由CPU执行。这就是我们经说的操作系统中线程的”抢占式调度“。

考虑上述系统调用的情况 ，如果我们销毁了M2线程，而M3线程即将进入系统调用。此时，在新的内核线程被创建并计划由OS执行之前，无法处理可运行的goroutine。频繁的线程预抢占操作不仅会增加OS的负担，还会对性能要求更高的程序来说是不可接受的。

因此，为了适当地利用操作系统的资源并防止频繁的线程抢占操作系统的负载，我们不会销毁内核线程M2，而是进行旋转操作，并保存它以备将来使用。尽管这似乎是在浪费一些资源。但是，与线程间频繁的抢占以及频繁的创建和销毁操作相比，”理想的线程“仍然要付出更少的代价 。

**旋转线程** - 例如，在具有一个内核线程M(1)和一个逻辑处理器(P)，如果正在执行的M被系统调用阻塞。而和P数量一样的旋转线程被允许等等可运行goroutine继续执行。因此，在这期间，内核线程M比P数量多（一个旋转线程和一个阻塞线程）。所以，即使将 `runtime.GOMAXPROCS`的值设置为1，程序也将处于多线程状态。

# 调度的公平性如何?

公平地选择下一步要执行的 goroutine

和很多其他调度器一下，Go也具有公平性约束，并且由goroutine的实现所强加，因为可运行的goroutine应该最终运行。

这里是四个Go运行时调度器的典型的公平性约束

任何运行超过10ms的goroutine被标记为可抢占（**软限制**）。但是，仅在函数序言中完成可抢占。**Go当前在函数序言中使用了由编译器插入的协作抢占点**。

* **无限循环** - 抢占（约10ms的时间片）- **软限制**

但是请注意无限循环，因为Go的调度器不是可抢占的（直到1.13）。如果循环不包含任何抢占点（如函数调用，或内存分配），它将阻止其他goroutine的运行。一个简单的例子：

```go
package main
func main() {
    go println("goroutine ran")
    for {}
}
```

如果使用下列运行：

```shell
GOMAXPROCS=1 go run main.go
```

直到Go1.13版本才会打印该语句。由于缺少抢占点，因此main的Goroutine可以占用处理器。

* **本地运行队列** - 抢占（约10ms时间片）- **软限制**
* 每61个调度器时刻检查一次全局队列运行，可以**避免全局运行队列的不足**。
* **网络轮询器饥饿**后台线程会在main线程未轮询的情况下偶尔轮询网络。

Go 1.14有一个新的“[非合作抢占](https://github.com/golang/proposal/blob/master/design/24543-non-cooperative-preemption.md)”

# Go运行时必需的调度器

* 可以处理并行执行（多线程）
* 处理阻塞的系统调用和网络I/O
* 处理阻塞的用户级（在通道上）调用
* 可扩展
* 高效
* 公平

这提供了大量的并发性，并且始终尝试实现最大和利用率和最小的延迟

# 调度跟踪

使用**GODEBUG=schedtrace=DURATION**环境变量运行Go程序可以开启调度程序跟踪（DURATION是以毫秒为单位周期输出）



参考：[可扩展Go调度器设计文档](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit)

https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit