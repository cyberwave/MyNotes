[Go:Understand the Design of Sync.Pool](https://medium.com/a-journey-with-go/go-understand-the-design-of-sync-pool-2dde3024e277)

*这篇文章基于 Go 1.12 和 Go 1.13 并解释了这两个版本间 `sync/pool.go` 的演变。*

包 `sync` 提供一个提供了一个强大的 pool 实例，它可以被重用以减轻垃圾收集器的压力。在使用这个包之前，使用 pool 前后应对应用程序进行基准测试非常重要，因为如果您不了解它内部的工作方式，则可能会降低性能。

# pool 的限制

我们以一个基本示例来看看它如何在非常简单的情况下（分配1k）工作：

```go
type Small struct {
   a int
}

var pool = sync.Pool{
   New: func() interface{} { return new(Small) },
}

//go:noinline
func inc(s *Small) { s.a++ }

func BenchmarkWithoutPool(b *testing.B) {
   var s *Small
   for i := 0; i < b.N; i++ {
      for j := 0; j < 10000; j++ {
         s = &Small{ a: 1, }
         b.StopTimer(); inc(s); b.StartTimer()
      }
   }
}

func BenchmarkWithPool(b *testing.B) {
   var s *Small
   for i := 0; i < b.N; i++ {
      for j := 0; j < 10000; j++ {
         s = pool.Get().(*Small)
         s.a = 1
         b.StopTimer(); inc(s); b.StartTimer()
         pool.Put(s)
      }
   }
}
```

下面是两个基准测试，其中一个没有使用 `sync.Pool` ，而另一个利用了它：

```shell
name           time/op        alloc/op        allocs/op
WithoutPool-8  3.02ms ± 1%    160kB ± 0%      1.05kB ± 1%
WithPool-8     1.36ms ± 6%   1.05kB ± 0%        3.00 ± 0%
```

由于循环有一万次迭代，没有使用 pool 的基准在堆上进行了一万次分配而使用了 pool 的基准只使用了3次分配。这三次分配是 pool 进行的，但仅分配了该结构体的一个实例。目前为止，一次正常；使用 `sync.Pool` 更快并且消耗更少的内存。

但在现实场景中，当你使用 pool，你的应用程序将在堆上进行很多分配。在这种情况下，当内存增加时，它将触发垃圾收集器。我们也可以在基准中使用命令 `runtime.GC()`来强制垃圾回收来模拟这种行为：

```shell
name           time/op        alloc/op        allocs/op
WithoutPool-8  993ms ± 1%    249kB ± 2%      10.9k ± 0%
WithPool-8     1.03s ± 4%    10.6MB ± 0%     31.0k ± 0%
```

现在我们可以看到使用 pool 的性能较低，分配的数量和使用的内存也更高。我们来深入研究这个包来理解为什么。

# 内部的工作流

挖掘 `sync/pool.go` 将向我们展示该软件包的初始化，它可以回答我们之前的关注:

```go
func init(){
  runtime_registerPoolCleanup(poolCleanup)
}
```

它向 runtime 注册一个方法来清理 pools。同样的方法将由垃圾回收器在其专用文件 `runtime/mgc.go` 中触发:

```go
func gcStart(trigger gcTrigger){
	[...]
  // clearpools before we start the GC
  cleanpools()
}
```

这解释了为什么当垃圾回收器被调用时它的性能会比较低。每次垃圾回收运行时都会清理 pools。[文档](https://golang.org/pkg/sync/#Pool)也警告了我们：

> 池中存储的任何项目都可以随时自动删除，恕不另行通知(*Any item stored in the Pool may be removed automatically at any time without notification*)

现在，让我们创建创建一个工作池来理解项目是如何被管理的：

![sync.Pool workflow in Go 1.12](https://miro.medium.com/max/572/1*OXMSVCef1UByrMBfK0_viQ.png)

对于我们创建的每个 `sync.Pool`，go会生成一个内部的池 `poolLocal` 连接到每个处理器。这个内部的池由两个属性组成：`private` 和 `shared`。第一个只能由它的所属者访问( push 和 pop -- 因此不需要任何锁 )而 `shared` 属性可以被任何其他的处理器读取，并且需要并发安全。实际上，pool 并不是一个简单的本地缓存，它有可能被我们应用程序中的任何线程/协程使用。

在 Go 的 1.13 版本将改善对共享项的访问，还将带来一个新的缓存，该缓存应解决与垃圾回收器和清理池有关的问题。

# 新的无锁池和受害者缓存(New lock-free pool and victim cache)

Go的 1.13版本带来了一个[新的双向链表](https://github.com/golang/go/commit/d5fd2dd6a17a816b7dfd99d4df70a85f1bf0de31#diff-491b0013c82345bf6cfa937bd78b690d)作为共享池，该链表移除了锁并改善了共享访问。这是改善缓存的基础。下面是共享访问的新的工作流:

![new shared pools in Go 1.13](https://miro.medium.com/max/886/1*BAH0gDeO2OuF-m2qwvQVpA.png)

有了这个新的链接池，每个处理器在其队列的头部都具有 push 和 pop 功能，而共享访问将从尾部弹出。队列的头部可以通过分配一个新的结构体的两倍来增加，该结构体将由 `next/prev` 属性连接到上一个。初始结构的默认大小是8个项目。这意味着第二个结构将有16个项目，第三个是32，等等。

同时，锁不是必须的，代码可以依赖原子操作。

关于新的缓存，新的策略相当简单。现在有2个池集合：活动的和存档的。当垃圾回收器运行的时候，它将把每个池的引用保存到池内的新属性上，然后将在清理当前池之前将池的集合复制到归档池中：

```go
// Drop victim caches from all pools.
for _, p := range oldPools {
   p.victim = nil
   p.victimSize = 0
}

// Move primary cache to victim cache.
for _, p := range allPools {
   p.victim = p.local
   p.victimSize = p.localSize
   p.local = nil
   p.localSize = 0
}

// The pools with non-empty primary caches now have non-empty
// victim caches and no pools have primary caches.
oldPools, allPools = allPools, nil
```

使用此策略，由于受害者缓存，应用程序现在将有一个更多的垃圾收集器周期来创建/收集带有备份的新项目。在工作流中，将在共享池后的过程结束时请求受害者缓存。



















