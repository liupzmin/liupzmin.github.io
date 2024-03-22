---
title: Go 语法糖 for range 中的 copy 问题
date: 2024-03-21 22:17:31
tags: 
    - Go
    - For-range
    - Iterating
    - Copy
categories: golang
---

Go 的赋值、参数传递都是值传递，也就是说你得到的是一份 copy，对于如下的 for range 循环：

```go
var someslice = []int{0,1}
for i, v := range someslice {
    // do something
}
```

短变量`v`是`someslice`中的元素的 copy，在 Go 1.22 以前`v`只会创建一次，每次循环会复用这一变量，从 1.22 起每次循环会创建新的变量。

现在的问题是： range 后的`someslice`还是原来的`someslice`吗？会不会也是一个 copy 呢？

你能猜出下面一段代码的打印结果吗？

```go
fib := []int{0, 1}
for i, f1 := range fib {
  f2 := fib[i+1]
  fib = append(fib, f1+f2)
  if f1+f2 > 100 {
    break
  }
}
fmt.Println(fib)
```

结果是：`[0 1 1 2]`，是不是不像推想的那样`fib`中的元素会超过 100？但如果分别在 for 循环的前、中、后打印一下`fib`的地址，你会发现地址没有变（在这个例子中底层数组是会变得，你可以使用`fmt.Printf("fib1:%p\n", fib)`来验证，对切片使用`%p`打印会打印第0个元素的地址）。

```go
fib := []int{0, 1}
fmt.Printf("fib outer:%p\n", &fib)
for i, f1 := range fib {
	fmt.Printf("fib inner:%p, f1:%p\n", &fib, &f1)
	f1, f2 := fib[i], fib[i+1]
	fib = append(fib, f1+f2)
	if f1+f2 > 100 {
		break
	}
}
fmt.Printf("fib outer:%p\n", &fib)
fmt.Println(fib)

//打印结果
fib outer:0xc00011c000
fib inner:0xc00011c000, f1:0xc000110030
fib inner:0xc00011c000, f1:0xc000110038
fib outer:0xc00011c000
[0 1 1 2]
```

这是不是说明 range 就是在遍历原`fib`本身呢？如果是遍历原`fib`，又为什么这里只循环了 2 次呢？

让我们看看[spec](https://go.dev/ref/spec#For_statements)中的说明：

```
The range expression x is evaluated once before beginning the loop, with one exception: if at most one iteration variable is present and len(x) is constant, the range expression is not evaluated.
```

这里的意思是：**在进入循环之前，range 表达式只会计算一次！**但这个`evaluate`具体指何意，spec 没有解答，看来只能在编译器源码中寻找答案了，我是没有大海捞针的精力了，不过已经有人替我们做了，美中不足的是参考的 gcc 的代码，不过想来都遵循语言规约的话，[行为方式总是大差不差的](https://github.com/golang/go/blob/ea020ff3de9482726ce7019ac43c1d301ce5e3de/src/cmd/compile/internal/gc/range.go#L169)，来看看 gcc 的 Go 编译器源码中 range 子句的注释：

```
// Arrange to do a loop appropriate for the type.  We will produce
//   for INIT ; COND ; POST {
//           ITER_INIT
//           INDEX = INDEX_TEMP
//           VALUE = VALUE_TEMP // If there is a value
//           original statements
//   }
```

可见 range 循环仅仅是 C-style 循环的语法糖，所以当你 range 一个 array 时：

```
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

range slice 时：

```
//   for_temp := range
//   len_temp := len(for_temp)
//   for index_temp = 0; index_temp < len_temp; index_temp++ {
//           value_temp = for_temp[index_temp]
//           index = index_temp
//           value = value_temp
//           original body
//   }
```

我们可以从中得到至少4点启示：

1. 循环最终都是 C-style 的。
2. 循环遍历的对象都会被赋值给一个临时变量。
3. 由第 2 点可知，range 一个数组的成本要大于 range 切片。
4. for range 居然涉及到 2 次 copy，一次是 copy 迭代的对象，一次是集合中的元素 copy 到临时变量。

我们还原一下开篇提到的代码，大致是如下的样子：

```go
fib := []int{0, 1}

var f1 int
// copy 迭代对象
temp := fib
for i := 0; i < len(temp); i++ {
  // copy 元素
  f1 = temp[i]
  f2 := fib[i+1]
  fib = append(fib, f1+f2)
  if f1+f2 > 100 {
    break
  }
}
fmt.Println(fib)
```

这就解释了代码为何只迭代两次，`fib`在循环开始前被复制，循环次数就被固定为 2 了。

因此，如果我们迭代一个切片，并且想修改里面的东西，除了在切片中存储指针以外，使用传统的 C-style 风格的循环也是个不错的选择。

```go
for i := 0; i < len(slice); i++ { ... }
```

如果我遍历的是个 map 呢？众所周知，map 是一个指针，range 计算就算 copy 也是 copy 得指针，遍历的时候仍然是同一个 map，事实上 spec 上也说明了这种情况：

``` 
If a map entry that has not yet been reached is removed during iteration, the corresponding iteration value will not be produced. If a map entry is created during iteration, that entry may be produced during the iteration or may be skipped. 
```

概括来说，你可以在 range 循环中对 map 进行增删，删掉的元素不会在接下来被遍历到，增加的元素则不一定，也许会被遍历，也许不会，这是由 map 底层使用哈希表实现以及随机遍历机制决定的。

**参考文献：**

1. [The Go Programming Language Specification](https://go.dev/ref/spec#For_statements)
2. [Go Range Loop Internals](https://garbagecollected.org/2017/02/22/go-range-loop-internals/)
3. [Does Go's range Copy a Slice Before Iterating Over It?](https://www.calhoun.io/does-range-copy-the-slice-in-go/)