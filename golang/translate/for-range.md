[通过两个例子介绍一下 Golang For Range 循环原理](https://blog.cyeam.com/golang/2018/10/30/for-interals)

for-range 是非常方便的，但我发现不止我一人觉得Go的 range 循环有点儿神秘！

# 下面的程序会终止吗？

```go
func main() {
  v := []int{1, 2, 3}
  for i := range v {
  	v = append(v, i)
  }
}
```

当然不会，对于切片的 `for range` ，它的底层代码是：

```go
for_temp := range
len_temp := len(for_temp)
for index_temp = 0; index_temp < len_temp; index_temp++ {
  value_temp = for_temp[index_temp]
  index = index_temp
  value = value_temp
  original body
}
```

在遍历之前就获取的切片的长度 `len_temp := len(for_temp)`，遍历次数不会随着切片的变化而变化。

# 下面的代码有问题吗？

```go
package main

import "fmt"

func main() {
	sli := []int{0, 1, 2, 3}
	myMap := make(map[int]*int)
	for idx, value := range sli {
		fmt.Printf("value address:%p\n", &value)
		myMap[idx] = &value
	}
	fmt.Println()
	for k, v := range myMap {
		fmt.Printf("%d=>%d, address=%p\n", k, *v, v)
	}
}
```

遍历切片，将切片值的地址保存到 `myMap` 中，结果为：

```shell
value address:0xc0000b4008
value address:0xc0000b4008
value address:0xc0000b4008
value address:0xc0000b4008

3=>3, address=0xc0000b4008
0=>3, address=0xc0000b4008
1=>3, address=0xc0000b4008
2=>3, address=0xc0000b4008
```

通过上面的底层代码可知，遍历的值都赋给了 `value`，而 value 类似一个全局变量，在例子中将 `value` 的地址保存到 `myMap` 中。在遍历后，`value` 的值保存是最后一次遍历的值，每次遍历 `value` 的地址不变，这就导致遍历后，`myMap` 的值都一样。

其它数据结构的 `for range`

## map

```go
// Lower a for range over a map.
// The loop we generate:
//   var hiter map_iteration_struct
//   for mapiterinit(type, range, &hiter); hiter.key != nil; mapiternext(&hiter) {
//           index_temp = *hiter.key
//           value_temp = *hiter.val
//           index = index_temp
//           value = value_temp
//           original body
//   }
```

## channel

```go
// Lower a for range over a channel.
// The loop we generate:
//   for {
//           index_temp, ok_temp = <-range
//           if !ok_temp {
//                   break
//           }
//           index = index_temp
//           original body
//   }
```

## 数组

```go
// Lower a for range over an array.
// The loop we generate:
//   len_temp := len(range)
//   range_temp := range
//   for index_temp = 0; index_temp < len_temp; index_temp++ {
//           value_temp = range_temp[index_temp]
//           index = index_temp
//           value = value_temp
//           original body
//   }
```

## 字符串

```go
// Lower a for range over a string.
// The loop we generate:
//   len_temp := len(range)
//   var next_index_temp int
//   for index_temp = 0; index_temp < len_temp; index_temp = next_index_temp {
//           value_temp = rune(range[index_temp])
//           if value_temp < utf8.RuneSelf {
//                   next_index_temp = index_temp + 1
//           } else {
//                   value_temp, next_index_temp = decoderune(range, index_temp)
//           }
//           index = index_temp
//           value = value_temp
//           // original body
//   }
```



参考：

1. [Go Range Loop Internals](https://garbagecollected.org/2017/02/22/go-range-loop-internals/)

