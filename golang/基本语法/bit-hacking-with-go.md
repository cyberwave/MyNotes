在内存非常昂贵，处理器还处于奔腾系列的计算机旧年代，直接进行位操作是处理信息的首选（在某情情况下是唯一的选项）。如今，直接的位操作在许多计算用例中仍然是至关重要的，例如底导的系统编程，图像处理，密码学等。

Go编程语言支持几种位运算符，包括：

```javascript
& 按位与
| 按位或
^ 按位异或
&^ AND NOT
<< 左移
>> 右移
```

本文的其他部分将详细讨论每个运算符，并包括如何使用的示例。

# & 运算符

在Go中，`&`运算符在两个整数操作数之间执行按位与运算。与运算（AND）具有以下属性：

```javascript
给出操作数a,b
AND(a,b)=1;当前仅当 a = b = 1 时才为1，否则为0
```

**AND**运算符有一个很好的作用，即有选择地将整数值的位清零。例如，我们可以使用 `&`运算符将最后4个有效位（LSB - least significant bits)清除（设置为零）置零。

```go
func main(){
  var x uint8 = 0xAC // x = 10101100
  x = x & 0xF0  // x = 10100000
}
```

所有的二进制运算符都运行简写算命赋值形式。例如，上面的例子可以写成：

```go
func main(){
  var x uint8 = 0xAC // x = 10101100
  x &= 0xF0 // x = 10100000
}
```

另外一个简单的技巧是测试一个数字是奇数不是偶数。这是因为当一个数字的最低有效位被设置为1时，它是奇数。我们可以使用 `&` 操作符与一个整数值 1 进行按位与操作。如果它的结果是1，则原数字为奇数。

```go
import (
    “fmt”
    “math/rand”
)
func main() {
    for x := 0; x < 100; x++ {
        num := rand.Int()
        if num&1 == 1 {
            fmt.Printf(“%d is odd\n”, num)
        } else {
            fmt.Printf(“%d is even\n”, num)
        }
    }
}
```

# | 操作符

`|` 在它的整数操作符上进行或运算。它的特点是当一个为真时，结果为真，同假为假。

```
给定操作数a,b
OR(a,b)=1; when a = 1 or b = 1 else = 0
```

我们可以使用按位或运算符的性质来有选择地为给定整数设置某一位。例如，下列例子中，我们使用或运算将设置（从最低有效位到最高有校位）第3，第7和第8位为1。

```go
func main() {
    var a uint8 = 0
    a |= 196
    fmt.Printf(“%b”, a)
}
// prints 11000100
```

当位屏蔽操作为给定整数设置任意位时，使用或操作非常有用。例如，我们可以将上一个程序中存储于 `a` 中的值设置为更多位。

```go
func main(){
  var a uint8 = 0
  a |= 196
  a |= 3
  fmt.Printf(“%b”, a)
}
// prints 11000111
```

# 位作为配置

现在，回想一下`AND(a, 1) = a if and only if a = 1`。我们可以使用这个事实来查询某值的设置位。例如，上面代码中 `a & 196` 返回 `196`，因为该值的位确实是在 `a` 中设置。因此，我们可以结合 OR 和 AND 来指定配置值并分别读取它们。

下面的代码片断展示了这个工作。函数 `procstr` 转换字符串的值。它使用两个参数：第一个参数 `str` 是待转换的字符串，第二个参数 `conf` 是个整数，它使用位掩码指定多个转换配置。

```go
const (
    UPPER  = 1 // upper case
    LOWER  = 2 // lower case
    CAP    = 4 // capitalizes
    REV    = 8 // reverses
)
func main() {
    fmt.Println(procstr(“HELLO PEOPLE!”, LOWER|REV|CAP))
}
func procstr(str string, conf byte) string {
    // reverse string
    rev := func(s string) string {
        runes := []rune(s)
        n := len(runes)
        for i := 0; i < n/2; i++ {
            runes[i], runes[n-1-i] = runes[n-1-i], runes[i]
        }
        return string(runes)
    }
 
    // query config bits
    if (conf & UPPER) != 0 {
        str = strings.ToUpper(str)
    }
    if (conf & LOWER) != 0 {
        str = strings.ToLower(str)
    }
    if (conf & CAP) != 0 {
        str = strings.Title(str)
    }
    if (conf & REV) != 0 {
        str = rev(str)
    }
    return str
}
```

函数调用 `procstr("HELLO PEOPLE!", LOWER |REV|CAP)`将字符串转为小写，逆序并大写每个单词。这通过设置参数 `conf `的值为14，即设置第2，第3，和第4位完成。然后代码使用连续的 `if` 语句提取这些位并应用适当的字符串转换。

# 异或 ^ 操作

异常操作在Go中使用 `^`应用。它的属性为:

```javascript
给定操作数 a, b
XOR( a, b ) = 1; 仅当 if a != b 才为1，否则为0
```

这个定义的含义是， XOR 可以用于将位从一个值切换为另一个值 。例如，给定一个16位的值，我们可以使用如下代码切换首8位的值。

```go
func main() {
    var a uint16 = 0xCEFF
    a ^= 0xFF00 // same a = a ^ 0xFF00
}
// a = 0xCEFF   (11001110 11111111)
// a ^=0xFF00   (00110001 11111111)
```

上面程序片断，与 1 异或的位被翻转（从0到1或从1到0）。例如，XOR的一个实际用途是比较符号量，当 `(a^b )>= 0` (或 `(a^b) < 0` 表示相反符号)为真时，两个整数 `a`、`b` 具有相同的符号，如下所示：

```go
func main() {
    a, b := -12, 25
    fmt.Println(“a and b have same sign?“, (a ^ b) >= 0)
}
```

当执行上面的程序，它将打印：`a and b have same sign? False`。

# ^ 按位补码（NOT)

不像其他语言（c/c++，Java，Python，Javascript等），Go 没有专用的一元逐位补码运算符。相反，异或运算符 `^`也可以用作一元运算符，将补码应用于数字。给定位 `x`，在Go中 `^x = 1 ^ x`，该位反转。我们可以在下面的代码中看到这一点，该代码片断斂 `^a` 作为变量 a 的补码

```go
func main() {
    var a byte = 0x0F
    fmt.Printf(“%08b\n”, a)
    fmt.Printf(“%08b\n”, ^a)
}

// prints
// 00001111     // var a
// 11110000     // ^a
```

# &^ 运算符

运算符 `&^`，读作 AND NOT，它是使用 `AND` 和 `NOT` 来操作它的操作数的简写形式。如下面的定义

```javascript
给定操作数 a, b
AND_NOT(a,b) = AND(a,NOT(B))
```

它有一个有趣的特性，即如果第二个操作数是 1，清除第一个操作数中的位。

```javascript
AND_NOT(a,1) = 0; clears a
AND_NOT(a,0) = a;
```

下面代码片断使用 AND NOT 操作来清除变量 `a` 的低四位从 `1010 1011` 到 `1010 0000`。

```go
func main() {
    var a byte = 0xAB
     fmt.Printf("%08b\n", a)
     a &^= 0x0F
     fmt.Printf("%08b\n", a)
}
// prints:
// 10101011
// 10100000
```

# 左移 << 和 右移 >> 操作符

类似其他C派生语言，Go 使用 `<<` 和 `>>` 表示左移和右移操作，相应的定义如下；

```javascript
给定整数操作数 a 和 n
a << n; 将 a 中所有的位左移n次
a >> n; 将 a 中所有的位右移n次
```

例如，在下面的片断中，左移操作符用来将存储于 `a` (`00000011`) 向左移动 3 次。为了说明目的，每次将打印结果

```go
func main() {
    var a int8 = 3
    fmt.Printf(“%08b\n”, a)
    fmt.Printf(“%08b\n”, a<<1)
    fmt.Printf(“%08b\n”, a<<2)
    fmt.Printf(“%08b\n”, a<<3)
}
// prints:
// 00000011
// 00000110
// 00001100
// 00011000
```

注意，每次移动，右边的第低有效位用零填充。相反地，使用右移操作符，值中的每一位都可以右移，左侧的第高有效位为零，如下面的示例所示（带符号的数字有一个异常，请参阅下面关于算术移位的说明）。

```go
func main() {
 var a uint8 = 120
 fmt.Printf(“%08b\n”, a)
 fmt.Printf(“%08b\n”, a>>1)
 fmt.Printf(“%08b\n”, a>>2)
}
// prints:
// 01111000
// 00111100
```

使用左移和右移运算符可以完成一些最简单的技巧是乘法和除法，其中每个移位位置表示二的幂，例如下面将200（存储在 `a` 中）除以2

```go
func main() {
    a := 200
    fmt.Printf(“%d\n”, a>>1)
}
// prints:
100
```

或乘以4

```go
func main(){
  a := 12
  fmt.Printf("%d\n", a << 2)
}
// prints:
// 48
```

移位运算符提供了在二进制值的指定位置操作位的有效方法。例如，在下面的代码片段中，`|` 和 `<<` 运算符用于设置 `a` 的第 3 位。

```go
func main() {
    var a int8 = 8
    fmt.Printf(“%08b\n”, a)
    a = a | (1<<2)
    fmt.Printf(“%08b\n”, a)
}
// prints:
// 00001000
// 00001100
```

或联合移位和 `&` 操作符来测试 `a` 的第 n 位是否被设置了值

```go
func main() {
    var a int8 = 12
    if a&(1<<2) != 0 {
        fmt.Println(“take action”)
    }
}
// prints:
// take action
```

使用 `&^` 和移动操作符，可以取消某个值的第 `n` 位的设置，下面片段取消了变量 `a` 的第3位。

```go
func main() {
    var a int8 = 13 
    fmt.Printf(“%04b\n”, a)
    a = a &^ (1 << 2)
    fmt.Printf(“%04b\n”, a)
}
// prints:
// 1101
// 1001
```

#  关于算术移位的注记

当要移位的值（）是有符号值时，Go 自动执行应用算术移位。在右移操作中，符号位（2的补码）被复制（或扩展）以填充移位的槽。

# 结论

与其他现代语言一样，Go 支持所有的按位运算符。这篇文章只提供了一小部分可以使用这些运算符的示例，可以在网上找到更多的内容，特别是Sean Eron Anderson的[*Bit Twiddling Hacks*](https://graphics.stanford.edu/~seander/bithacks.html)











参考：

1. [Bit Hacking with Go](https://dev.to/vladimirvivien/bit-hacking-with-go)
2. [Bit Hacking with Go](https://medium.com/learning-the-go-programming-language/bit-hacking-with-go-e0acee258827)
3. 