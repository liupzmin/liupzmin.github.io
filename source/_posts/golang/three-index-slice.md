---
title: Go 语言中的 Three-index slices
date: 2020-07-16 10:41:24
tags: 
    - golang
    - slice
categories: 遇见Go
---

最近在看一段源码的时候，发现了一个从未见过的 slice 的用法：

```go
for batchSize < len(readValues) {
    rawValues[address] = readValues[0:batchSize:batchSize]
    address = address + 1
    readValues = readValues[batchSize:]
}
```

其中`readValues[0:batchSize:batchSize]`为一个切片操作，但令人困惑的是其拥有3个索引值；这种书写形式在我读过的有限书籍中从未见过，官方的 `[Specification](https://golang.org/ref/spec)` 也没有找到相应的描述，所以打算将其弄个一清二楚。

经过一番 google 之后，[golang slice, slicing a slice with slice[a:b:c]](https://stackoverflow.com/questions/27938177/golang-slice-slicing-a-slice-with-sliceabc) 和 [Re-slicing slices in Golang](https://stackoverflow.com/questions/12768744/re-slicing-slices-in-golang) 两篇问答让我顺藤摸瓜找到了源头，它源自 Go1.2 中的新特性——[Three-index slices](https://tip.golang.org/doc/go1.2#three_index)。

我们知道，一个 slice 由三个部分构成：**指针、长度和容量**:

- 指针, 指向其第一个元素对应的底层数组元素的地址
- 长度,对应的是slice中的元素个数，也就是通过下标对slice中的元素进行访问时，不得超过长度的大小，否则会`panic`
- 容量,`一般是`从slice的第一个元素位置到底层数据的结尾位置。

`slice[i:j:k]` 第二个冒号之后的索引便是`容量`，默认情况下的容量是这个slice能`hold`住的最大元素个数，也就是上文提到的从slice的第一个元素位置到底层数据的结尾位置，即使这个slice再被截取，容量也是从起始位置到底层数组的结尾位置。

那什么叫`hold`住呢？`长度以外容量以内`的元素又不能通过下标访问，这能叫`hold`么？当然，`长度以外容量以内`意味着你可以`append`，我们试举一例：

```go
var a = []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
b := a[2:5]
fmt.Println(b[3]) // panic
b = append(b, 100)
fmt.Println(b[3])
fmt.Println(a) // [1 2 3 4 5 100 7 8 9 10]
```

可以看到切片b的长度为3，直接通过下标访问b[3]会报运行时恐慌，而它的默认容量是8，所以可以append新值进去，并且最终修改了底层数组的值，通过打印切片a就可以看到底层数组也被修改了，a与b共享底层数组。

接下来我们使用第三个索引来指定切片的容量和长度一样：

```go
c := a[2:5:5]
c = append(c, 200)
fmt.Println(c) // [3 4 5 200]
fmt.Println(a) // [1 2 3 4 5 100 7 8 9 10]
```

可见，切片a的内容并没有变，因为c的容量有限，append操作引起了扩容，从而使得切片c的底层数组与切片a的底层数组分道扬镳。[码农桃花源](https://qcrao.com/)的[深度解密Go语言之Slice](https://www.cnblogs.com/qcrao-2018/p/10631989.html)中对于数组扩容有更精彩的图文论述，其中也有`data[low:high:max]`的论述，只是当时阅读数未曾引起重视。

官网用于说明的例子是：

```go
var array [10]int
slice = array[2:4:7]
```

一句话来表述容量被限制之后的结果：` It is impossible to use this new slice value to access the last three elements of the original array.` 

这个特性偶尔会很有用，比如在处理底层的[]byte时，如果很清楚调用者不会去修改slice中的值，那么就可以使用这种方式来更好的保护底层的数组，正如我开篇提到的那段代码一样。

三个索引值，第一个可以省略，如果省略就代表 0.
