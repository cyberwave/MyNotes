5个switch 语句模式

# 基本开关默认

* `switch` 语句运行第一种情况属于条件表达式
* `case` 从上到下被评估，在case成功时停止
* 如果没有匹配任何 `case` 并且有一个默认的 `case` ，它的语句被执行

```go
switch time.Now().Weekday(){
case time.Saturday:
    fmt.Println("Today is Saturday."))
case time.Sunday:
    fmt.Println("Today is Sunday.")
default:
    fmt.Println("Today is a weekday.")    
}
```

> 不像 C 和 Java 那样，case 表达式并不需要是常量。

# 没有条件

一个没有条件的 `switch` 和 `swithch` true 一样。

```go
switch hour := time.Now().Hour();{
case hour < 12:
    fmt.Println("Good morning!")
case hour < 17:
    fmt.Println("Good afternoon!")
default:
    fmt.Println("Good evening!")
}
```

# case 列表

```go
func WhiteSpace( c rune) bool {
    switch c {
    case ' ', '\t', '\n','\f','\r':
        return true
    }
    return false
}
```

# Fallthrough

* 一个 `fallthrought`语句将控制转换到了下一个 case.
* 它可能只作为一个语句的最后一个语句被使用.

```go
switch 2 {
case 1:
    fmt.Println("1")
    fallthrough
case 2:
    fmt.Println("2")
    fallthrough
case 3:
    fmt.Println("3")
}
```

```shell
2
3
```

# 使用 Exit 退出

一个 `break` 语句终止执行 `for`,`switch` 或 `select` **最里面** 语句

如果你需要跳出 loop 循环，而不是 `switch` ，你可以在循环上放一个 **标签** 并跳到该标签。这个示例展示他们如何使用：

```go
Loop:
for _, ch := range "a b\nc" {
    switch ch {
        case ' ': // 跳过空格
        break
    case '\n': //换行
        break Loop
    default:
        fmt.Println("%c\n",ch)
    }
}
```

```shell
a
b
```

# 执行顺序

*  首先，对switch 表达式求值一次。
* 然后从左到右和从上到下评估 case 表达式。
  * 第一个等于 switch 表达式的触发器执行相关 case 语句
  * 其他 case 被跳过

```go
// Foo prints and return n.
func Foo(n int) int {
    fmt.Println(n)
    return n
}

func main() {
    switch Foo(2) {
    case Foo(1), Foo(2), Foo(3):
        fmt.Println("First case")
    	fallthrough
    case Foo(4):
        fmt.Println("Second case")    
    }
}
```

```shell
2
1
2
First case
Second case
```

**注意：**

