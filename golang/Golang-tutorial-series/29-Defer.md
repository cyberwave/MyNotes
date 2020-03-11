[Go系列教程](https://golangbot.com/learn-golang-series/) [Part29 Golang tutorial series](https://golangbot.com/defer/)

# 为什么是 Defer?

Defer 语句是用于在存在 defer 语句的函数返回前执行一个函数调用。定义看起来可能复杂，但通过一个例子很容易理解。

```go
package main

import (  
    "fmt"
)

func finished() {  
    fmt.Println("Finished finding largest")
}

func largest(nums []int) {  
    defer finished()    
    fmt.Println("Started finding largest")
    max := nums[0]
    for _, v := range nums {
        if v > max {
            max = v
        }
    }
    fmt.Println("Largest number in", nums, "is", max)
}

func main() {  
    nums := []int{78, 109, 2, 563, 300}
    largest(nums)
}
```

[Run in playground](https://play.golang.org/p/IlccOsuSUE)

上面是一个简单的程序，在给定的切片中找到最大的数字。`largest` 函数使用一个 *int [slice](https://golangbot.com/arrays-and-slices/#slices)* 作为参数并打印输入切片的最大数。`largest` 函数的第一行包含语句 `defer finished()`。它意味着 `finished()` 函数在 `largest` 函数之前被调用。运行该程序，可以看到下列的输出:

```shell
Started finding largest
Largest number in [78 109 2 563 300] is 563
Finished finding largest
```

函数 largest 开始执行并打印上面输出的前两行。在它返回之前，我们的延迟函数 `finished` 执行并打印文本 `Finished finding largest`

# 延迟方法(Deferred methods)

Defer 并不仅限制到[函数functions]()，延迟[方法method]()调用也是非常适合的。我们来写一个小程序测试它。

```go
package main

import (  
    "fmt"
)


type person struct {  
    firstName string
    lastName string
}

func (p person) fullName() {  
    fmt.Printf("%s %s",p.firstName,p.lastName)
}

func main() {  
    p := person {
        firstName: "John",
        lastName: "Smith",
    }
    defer p.fullName()
    fmt.Printf("Welcome ")  
}
```

[Run in playground](https://play.golang.org/p/lZ74OAwnRD)

在上面的程序中，在第22行，我们有一个延迟方法调用。程序的其余部分不言自明。程序输出：

```shell
Welcome John Smith
```

# 参数评估

延迟函数的参数评估是在当 `defer` 语句被执行时而不是在实际函数调用结束时。

我们通过一个例子来理解它。

```go
package main

import (
	"fmt"
)

func printA(a int) {
  fmt.Println("value of a in deferred function", a)
}
func main() {
  a := 5
  defer printA(a)
  a = 10
  fmt.Println("value of a before deferred function call", a)
}
```

[Run in playground](https://play.golang.org/p/sBnwrUgObd)

在上面的程序中 ，在第11行 `a` 的有一个初始值是 `5`。当 defer 语句在第12行被执行，`a` 的值是 5 因为这将是被延迟的 `printA` 的参数，我们在第13行修改 `a` 的值为10.下一行打印 `a` 的值，程序输出：

```shell
value of a before deferred function call 10  
value of a in deferred function 5
```

从上面的输出可以理解尽管 `a` 的值在 defer 语句被执行后被修改到 `10`，实际延迟函数调用 `printA(a)` 仍然打印 `5`。

# defers栈

当一个函数有多个 defer 调用，它们被加入到栈中并以后进先出(LIFO -- Last In First Out)的顺序被执行。

我们写一个程序使用延迟栈逆序打印字符串。

```go
package main

import (
	"fmt"
)

func main() {
  name := "Naveen"
  fmt.Printf("Original String: %s\n", string(name))
  fmt.Printf("Reversed String: ")
  for _, v := range []rune(name) {
      defer fmt.Printf("%c", v)
  }
}
```

[Run in playground](https://play.golang.org/p/x05NlTsDPuT)

在上面的程序中，第11行的 `for range` 循环，在字符串上进行迭代并调用 `defer fmt.Printf("%c", v)`。这些延迟调用将被添加到栈中并以后进先出的顺序被执行，因此字符串将会被逆序打印：

```shell
Original Strings: Naveen
Reversed Strings: neevaN
```

# defer的实际应用

目前为止我们看到的代码示例没有显示 defer 的实际应用。在这个部分我们将看一些 defer 的实际应用。

Defer 应用在无论函数的代码流如何都应该被执行的地方。我们来用一个使用 [WaitGroup](https://golangbot.com/buffered-channels-worker-pools/#waitgroup)的程序的例子来理解它。

我们首先写一个没有使用 defer 的程序然后修改为使用 defer ，来理解defer 有多有用。

```go
package main

import (  
    "fmt"
    "sync"
)

type rect struct {  
    length int
    width  int
}

func (r rect) area(wg *sync.WaitGroup) {  
    if r.length < 0 {
        fmt.Printf("rect %v's length should be greater than zero\n", r)
        wg.Done()
        return
    }
    if r.width < 0 {
        fmt.Printf("rect %v's width should be greater than zero\n", r)
        wg.Done()
        return
    }
    area := r.length * r.width
    fmt.Printf("rect %v's area %d\n", r, area)
    wg.Done()
}

func main() {  
    var wg sync.WaitGroup
    r1 := rect{-67, 89}
    r2 := rect{5, -67}
    r3 := rect{8, 9}
    rects := []rect{r1, r2, r3}
    for _, v := range rects {
        wg.Add(1)
        go v.area(&wg)
    }
    wg.Wait()
    fmt.Println("All go routines finished executing")
}
```

[Run in playground](https://play.golang.org/p/kXL85U0Dd_)

在上面的程序，我们在第8行创建一个 rect 结构体，并在第13 行中创建结构体 `rect` 上的 `area` 方法，该方法计算长方形的面积。该方法检查长方形的长和宽是否小于零。如果为零，则打印相应的消息，否则的话打印长方形的面积。

函数 `main` 创建类型为 `rect` 的三个变量 `r1`，`r2` 和 `r3`。它们在后面第34行被添加到 `rects` 切片中。该切片在后面使用 `for range` 循环进行迭代并在第37行使用 [Goroutine](https://golangbot.com/goroutines/)并发地调用 `area` 方法。`WaitGroup wg` 是用来确保主函数阻塞，直到所有协程执行完成。该 WaitGroup 作为参数传递到 area 方法中，area 方法在16，21 和 26 行调用 `wg.Done()` 通知主函数该协程完成了它的工作。*如果你观察得够细致，你可以看到这些调用发生在 area 方法返回之前。wg.Done() 应该在方法返回之前被调用，与代码流 的路径无关，因此这些调用可以被单个 `defer` 有效地替换。

我们来使用 defer 重写上面的程序。

在下面的程序中，我们移除3个 `wg.Done()` 调用，使用一个 `defer wg.Done()` 调用替换它。这使代码更简单和被理解。

```go
package main

import (  
    "fmt"
    "sync"
)

type rect struct {  
    length int
    width  int
}

func (r rect) area(wg *sync.WaitGroup) {  
    defer wg.Done()
    if r.length < 0 {
        fmt.Printf("rect %v's length should be greater than zero\n", r)
        return
    }
    if r.width < 0 {
        fmt.Printf("rect %v's width should be greater than zero\n", r)
        return
    }
    area := r.length * r.width
    fmt.Printf("rect %v's area %d\n", r, area)
}

func main() {  
    var wg sync.WaitGroup
    r1 := rect{-67, 89}
    r2 := rect{5, -67}
    r3 := rect{8, 9}
    rects := []rect{r1, r2, r3}
    for _, v := range rects {
        wg.Add(1)
        go v.area(&wg)
    }
    wg.Wait()
    fmt.Println("All go routines finished executing")
}
```

[Run in playground](https://play.golang.org/p/JuUvytLfBv)

程序输出：

```shell
rect {8 9}'s area 72  
rect {-67 89}'s length should be greater than zero  
rect {5 -67}'s width should be greater than zero  
All go routines finished executing  
```

上面程序使用 defer 还有一个好处。假设我们在 `area` 方法中使用一个新的 `if` 语句增加一个 return 路径。如果不使用延迟调用 `wg.Done()`，我们必须小心并确保在新的 return 路径中调用了 `wg.Done()`。但是由于使用了延迟调用 `wg.Done()` ，我们不必担心这个方法的新的 return 路径。

**下一教程 - [错误处理(Error Handling)]() **

















