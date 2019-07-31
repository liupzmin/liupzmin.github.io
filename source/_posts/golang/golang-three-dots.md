---
title: golang三个点(...)的用法
date: 2019-07-31 13:17:31
tags: golang
categories: golang
---

## Passing arguments to ... parameters

If f is variadic with a final parameter p of type ...T, then within f the type of p is equivalent to type []T. If f is invoked with no actual arguments for p, the value passed to p is nil. Otherwise, the value passed is a new slice of type []T with a new underlying array whose successive elements are the actual arguments, which all must be assignable to T. The length and capacity of the slice is therefore the number of arguments bound to p and may differ for each call site.

Given the function and calls

```go
func Greeting(prefix string, who ...string)
Greeting("nobody")
Greeting("hello:", "Joe", "Anna", "Eileen")
```
within Greeting, who will have the value nil in the first call, and []string{"Joe", "Anna", "Eileen"} in the second.

If the final argument is assignable to a slice type []T, it may be passed unchanged as the value for a ...T parameter if the argument is followed by .... In this case no new slice is created.

Given the slice s and call

```go
s := []string{"James", "Jasmine"}
Greeting("goodbye:", s...)
```

within Greeting, who will have the same value as s with the same underlying array.

**上面是官方文档中关于`...`的用法的描述，主要有两种使用形式：**

### 参数列表

如果`最后一个`函数参数的类型的是...T，那么在调用这个函数的时候，我们可以在参数列表的最后使用若干个类型为T的参数。这里，...T在函数内部的类型实际是[]T,是隐式创建的slice:

```go
func Sum(nums ...int) int {
    res := 0
    for _, n := range nums {
        res += n
    }
    return res
}

Sum(1,2,3)
```

### slice参数

如果传入的`最后一个`函数参数是slice []T,直接在slice后跟`...`，那么就不会创建新的slice:

```go
primes := []int{2, 3, 5, 7}
fmt.Println(Sum(primes...)) // 17
```

因此，当我们想把一个slice追加到另外一个slice时，可以使用如下方式:

```go
s1 := []int{0, 1, 2, 3}
s2 := []int{4, 5, 6, 7}
s1 = append(s1, s2...) // instead of FOR
fmt.Println(s1)
```

或者，当我们想在slice中删除一个元素时:

```go
s1 := []int{0, 1, 2, 3}
fo index, v := range s1{
    if index == 2{
        s1 = append(s1[:index], s1[index+1:]...)
    }
}
```

## 标识数组元素个数

默认情况下，数组的每个元素都被初始化为元素类型对应的零值，对于数字类型来说就是0。我们也可以使用数组字面值语法用一组值来初始化数组：

```go
var q [3]int = [3]int{1, 2, 3}
var r [3]int = [3]int{1, 2}
fmt.Println(r[2]) // "0"
```

在数组字面值中，如果在数组的长度位置出现的是“...”省略号，则表示数组的长度是根据初始化值的个数来计算。因此，上面q数组的定义可以简化为

```go
q := [...]int{1, 2, 3}
fmt.Printf("%T\n", q) // "[3]int"
```

## Go命令行中的通配符

描述包文件的通配符。
在这个例子中，会单元测试当前目录和所有子目录的所有包：

```bash
go test ./...
```

**参考文献：**

1. [golang 三个点(three dots)的用法](https://github.com/zhangyachen/zhangyachen.github.io/issues/137)
2. [Passing arguments to ... parameters](https://golang.org/ref/spec#Passing_arguments_to_..._parameters)