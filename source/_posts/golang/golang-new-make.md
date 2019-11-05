---
title: Go语言中的new和make-从函数阻击战到nil遭遇战
date: 2019-10-18 09:30:46
tags: 
	- golang
	- go nil
categories: golang
---

Go语言中有两个`builtin`函数`new`和`make`，这个两个函数经常让初学者摸不着头脑，也许使用过程中并未有什么阻碍，但回过头看细想又难以说清道明。本文将针对这两个函数进行分析，希望能抽丝剥茧，彻底搞清楚他们的区别。

##  函数定义

当然，我们先看Go官方对这两个函数的解释：

### func new(Type) *Typel

> The new built-in function allocates memory. The first argument is a type, not a value, and the value returned is a pointer to a newly allocated zero value of that type.

**大意：new函数会分配内存，它唯一的一个参数是`type`不是value，返回值是一个指针，指向刚刚分配的那块内存，并且这块内存中存储着`type`的`零值（zero value）`。**

重点：
1. 分配内存
2. 返回指向这块内存的指针
3. 内存存储`type`的`零值`

### func make(t Type, size ...IntegerType) Type

> The make built-in function allocates and initializes an object of type slice, map, or chan (only). Like new, the first argument is a type, not a value. Unlike new, make's return type is the same as the type of its argument, not a pointer to it.

**大意：make仅用于分配和初始化`slice、map、chanel`，同new一样，第一个参数要传入一个`type`；不同的是，make返回的是初始化过之后的`type`的一个值，而不是指针。**

重点：
1. 分配内存并初始化
2. 仅用于`slice`、`map`、`chanel`
3. 返回的是值，不是指针

## zero value

上文提到`new()`会分配内存，并且为相应的`Type`存储`零值`，`那什么是零值呢？`官方对于`zero value`的描述如下：

> When storage is allocated for a variable, either through a declaration or a call of new, or when a new value is created, either through a composite literal or a call of make, and no explicit initialization is provided, the variable or value is given a default value. Each element of such a variable or value is set to the zero value for its type: false for booleans, 0 for numeric types, "" for strings, and nil for pointers, functions, interfaces, slices, channels, and maps. This initialization is done recursively, so for instance each element of an array of structs will have its fields zeroed if no value is specified.


**大意：当通过声明或调用new为变量分配存储时，或者在创建新值时(通过复合字面值或调用make)，并且没有提供显式初始化时，将为变量或值提供默认值。此类变量或值的每个元素的类型都设置为该类型的零值:布尔值为false，数值类型为0，字符串为""，指针、函数、接口、片、通道和映射为nil。这个初始化是递归完成的，因此，如果结构体中没有指定相应field的值，那么默认将是该field的零值。**


下面表格中展示了Go中主要类型的零值：

| Type      | Zero Value |
|-----------|------------|
| boolean   | false      |
| numeric   | 0          |
| string    | ""         |
| pointer   | nil        |
| function  | nil        |
| interface | nil        |
| slice     | nil        |
| map       | nil        |
| channel   | nil        |

其中，array和struct两个复合类型的零值为其承载的基础类型的零值，因为array和struct都是值类型，不像slice、map是引用类型。

但是，个人感觉上面一段话中关于`通过复合字面值或调用make`创造值的相关描述略有不准确。因为`Composite literals(复合字面值)`是为`structs`、`arrays`、`slices`、`maps`构造值，而`make`仅用于分配并初始化`slice`、`map`、`chanel`。如果使用`字面值`构造一个值且不显示的初始化，那么该值就是一个`空值（empty）`，和`make`的结果相同,`而不是零值nil`。看下面这段代码：

```go
var nilSlice []string
newNilSlice := new([]string)
emptySlice := make([]string, 0)
emptySliceLiteral := []string{}
fmt.Println(nilSlice)                                       // Output: []
fmt.Println(len(nilSlice), cap(nilSlice))                   // Output: 0 0
fmt.Println(nilSlice == nil)                                // Output: true
fmt.Println(*newNilSlice)                                   // Output: []
fmt.Println(len(*newNilSlice), cap(*newNilSlice))           // Output: 0 0
fmt.Println(*newNilSlice == nil)							// Output: true
fmt.Println(emptySlice)                                     // Output: []
fmt.Println(len(emptySlice), cap(emptySlice))               // Output: 0 0
fmt.Println(emptySlice == nil)                              // Output: false
fmt.Println(emptySliceLiteral)                              // Output: []
fmt.Println(len(emptySliceLiteral), cap(emptySliceLiteral)) // Output: 0 0
fmt.Println(emptySliceLiteral == nil)                       // Output: false
```

像`slice`、`map`这样的引用类型的`零值是nil`，并不是`“the variable or value is given a default value”`，因为`通过复合字面值或调用make`构造变量时`default value`是`empty`。（***如果有人觉得这是咬文嚼字，那我也只能承认，毕竟nil让我很痛苦，后面我会再论述default value***）。


一个`nil的slice`和一个`Empty的slice`很容易让你迷惑，它们都被`fmt.Println`打印出`[]`，它们拥有相同的`length`和`capacity`。

除非`nil`或者`Empty`会对你的逻辑产生影响，否则不用区别对待它们。如果你需要测试一个slice是否是空的，使用`len(s) == 0`来判断，而不应该用`s == nil`来判断。除了和nil相等比较外，一个nil值的slice的行为和其它任意0长度的slice一样。

另一个值得注意的问题是，当你使用`encoding/json`时，你要特别注意：Golang的`encoding/json`会将`Nil Slice`编码为`null`。


## what is nil？

### nil的定义

nil 为预声明的标示符，定义在builtin/builtin.go，

```
// nil is a predeclared identifier representing the zero value for a
// pointer, channel, func, interface, map, or slice type.
// Type must be a pointer, channel, func, interface, map, or slice type
var nil Type 

// Type is here for the purposes of documentation only. It is a stand-in
// for any Go type, but represents the same type for any given function
// invocation.
type Type int
```

可见，`nil没有默认类型`，它是一个预定义好的变量，有多种可能的类型（`pointer`、`map`、`slice`、`function`、`channel`、`interface`）。它代表指针、通道、函数、接口、映射或切片的`零值`。

你必须给编译器以足够的信息，使得编译器可以推导出nil的类型，因此下面的使用方式是非法的：

```go
var n = nil // illegal， doesn't compile
```

正确的做法如下：

```go
func main() {
	// There must be sufficient information for
	// compiler to deduce the type of a nil value.
	_ = (*struct{})(nil)
	_ = []int(nil)
	_ = map[int]bool(nil)
	_ = chan string(nil)
	_ = (func())(nil)
	_ = interface{}(nil)

	// This lines are equivalent to the above lines.
	var _ *struct{} = nil
	var _ []int = nil
	var _ map[int]bool = nil
	var _ chan string = nil
	var _ func() = nil
	var _ interface{} = nil
}
```


### nil的地址

各种类型的`nil`的内存布局始终相同,换一句话说就是：不同类型nil的内存地址是一样的。

```go
func main() {

	var m map[int]string
	var ptr *int
	var sl []int
	testNewAddr := new([]string)
	testMakeAddr := make([]string, 0)
	fmt.Printf("%p\n", m)            	//0x0
	fmt.Printf("%p\n", ptr)          	//0x0
	fmt.Printf("%p\n", sl)           	//0x0
	fmt.Printf("%p\n", *testNewAddr) 	//0x0
	fmt.Printf("%p\n", testMakeAddr) 	//0x57db60

}
```
可见，值为`nil`的变量都指向同样的内存地址`0x0`，这是一个无效的地址，如果对这个地址进行读写，会引发`panic`：

```go
func main() {
	var p *int  // Declare a nil value pointer
	*p = 10		// Write the value 10 to address 0x0
}
```

```
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x4525b2]

goroutine 1 [running]:
main.main()
	/home/alarm/go/test/test.go:5 +0x2
```

我们来看一下这段代码的`plan9汇编`指令：

```x86asm
go tool objdump -s main.main test

TEXT main.main(SB) /home/alarm/go/test/test.go
  test.go:5		0x4525b0		31c0				XORL AX, AX		
  test.go:5		0x4525b2		48c7000a000000		MOVQ $0xa, 0(AX)	
  test.go:6		0x4525b9		c3					RET				
```
从汇编指令可以看到，`AX`寄存器被清零，之后试图将`0xa`（10）写入`AX指向的内存地址`，然后就导致了panic。

***现在我们可以作如下总结：***
>`非引用类型`一旦赋予`default value`（或者说`zero value`），那么将会实际分配内存，并且内存中数据初始化为相应类型的`zero value`；而零值为`nil`的类型都是`引用类型`，其背后引用了使用前必须初始化的数据结构，它们的`default value`为`nil`，尚未分配内存，相应的数据结构也未被初始化。例如，slice是一个三元描述符，包含一个指向数据（在数组中）的指针、长度、以及容量，在这些项被初始化之前，slice都是nil的。对于slice，map和channel，make初始化这些内部数据结构，并准备好可用的值（对应类型的`zero value`）。

### 不是关键字

另一个值得注意的地方：`nil不是Go的关键字`，你可以重写他，但是最好不要这样做：
```go
func main() {
	nil := 123
	fmt.Println(nil) // 123

	// The following line fails to compile,
	// for nil represents an int value now
	// in this scope.
	var _ map[string]int = nil
}
```

## kinds of nil

我们知道nil的种类有`pointer`, `channel`, `func`, `interface`, `map`, or `slice`，下面分别讨论一下这几种类型的nil行为。

先说明一下这几种类型的nil含义：

| Type      | meaning    |
|-----------|------------|
| pointer   | 什么也不指向 |
| function  | 没有初始化   |
| interface | 没有赋值，空指针 |
| slice     | 没有底层数组  |
| map       | 没有初始化   |
| channel   | 没有初始化    |



### pointer

在go中，指针指向一个内存地址，同c/c++中一样，但go中的指针没有指针运算，所以是安全的。可以有以下几种方式生成pointer：

```go
func main() {
	// 第一种：直接声明指针类型
	var p *[]int
	fmt.Println(p == nil)		// Output: true
	fmt.Println(*p == nil)		// panic: runtime error: index out of range
	// 第二种：使用new()函数返回指针
	pNew := new([]int)
	fmt.Println(pNew == nil)	// Output: false
	fmt.Println(*pNew == nil)	// Output: true

	// 第三种：通过取址符&，产生一个指向变量的指针
	s := []int{}
	pAnd := &s
	fmt.Println(pAnd == nil)	// Output: false
	fmt.Println(*pAnd == nil)	// Output: false
	// 给nil的指针赋值
	p = &s
	fmt.Println(p == nil)		// Output: false
	fmt.Println(*p == nil)		// Output: false
}
```
上述代码我故意用了`slice`的指针类型，因此当pNew不是nil的时候，`*pNew依然是nil`。

`nil的指针无法解引用，如果试图对一个nil的指针解引用的话会产生panic。`

`但是Nil却可以作为合法的接收器：`

```go

type PointerReciver int

func (a *PointerReciver) showme(){
	fmt.Printf("Yeah, it works!")
}

func main() {
	var ta *PointerReciver
	ta.showme()		// Yeah, it works!
}
```

### slice

slice的底层是一个数组，那么一个nil的slice是没有底层数组的。一个`nil`的或者`长度为0的非nil`的slice都无法被索引，但是可以使用`append`去填充值。下面代码展示了各种构造slice的方式，以及nil的情况：

```go
func main() {
	//  显示声明
	var ts []int
	fmt.Println(ts == nil) // Output: true
	ts[0] = 1 // panic: runtime error: index out of range
	ts = append(ts, 2) // slice: [2]

	// 使用new函数
	p := new([]int)
	fmt.Println(*p == nil) // Output: true
	(*p)[0] = 1 // panic: runtime error: index out of range
	*p = append(*p, 2) // slice: [2]

	// 使用字面值初始化
	literal := []int{}
	fmt.Println(literal == nil) // Output: false
	literal[0] = 1 // panic: runtime error: index out of range
	literal = append(literal, 2) // slice: [2]

	// 使用make初始化
	mp := make([]int, 0)
	fmt.Println(mp == nil) // Output: false
	mp[0] = 1 // panic: runtime error: index out of range
	mp = append(mp, 2) // slice: [2]

	// 从数组截取
	arrayExample := [10]int{}
	sliceFromArray := arrayExample[2:5]
	fmt.Println(sliceFromArray == nil) // Output: false
	sliceFromArray[0] = 1
	sliceFromArray = append(sliceFromArray, 2) // slice: [1 0 0 2]

	var tnilslice []int
	// 不会迭代
	for k ,v := range tnilslice{
		fmt.Printf("k: %d, v: %d\n", k, v)
	}
}
```
**Slice小结：**


***1. 显示声明、未初始化值时为nil***
***2. 使用new函数生成slice为nil***
***3. 字面值初始化的slice，未提供具体值的为Empty，不是nil，但长度和容量与nil相同都为0***
***4. 使用make初始化的slice未提供长度和容量的为Empty，提供的初始化为相应类型的零值***
***5. nil和Empty的slice（长度为0）无法被索引***
***6. 使用for...range遍历nil的slice不会迭代，也不会报错***


### map

map的底层是一个哈希表，通过内置的make函数可以快速构建一个map。make创建map时，实际底层调用的是`makemap`函数，返回值是一个结构体指针。

>func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap

使用make可以选填`capacity `，capacity 不限制map的大小，map会自适应增长，但是nil的map除外。除了不允许添加元素以外，nil的map等价于Empty的map。

```go
func main() {

	// 直接声明
	var dmap map[string]int
	fmt.Println(dmap == nil) // Output: true
	fmt.Println(dmap["a"])   // Output: 0
	dmap["b"] = 1            // panic: assignment to entry in nil map
	delete(dmap, "c")        // no-op

	// 使用new函数
	nmap := new(map[string]int)
	fmt.Println(*nmap == nil) // Output: true
	fmt.Println((*nmap)["a"]) // Output: 0
	(*nmap)["b"] = 1          // panic: assignment to entry in nil map
	delete(*nmap, "c")        // no-op

	// 字面量初始化
	lmap := map[string]int{}
	fmt.Println(lmap == nil) // Output: false
	fmt.Println(lmap["a"])   // Output: 0
	lmap["b"] = 1            // sucess
	delete(lmap, "c")        // no-op

	// 使用make初始化
	mmap := make(map[string]int)
	fmt.Println(mmap == nil) // Output: false
	fmt.Println(mmap["a"])   // Output: 0
	mmap["b"] = 1            // sucess
	delete(mmap, "c")        // no-op

	var tnilmap map[string]int
	// 不会迭代
	for key, value := range tnilmap{
		fmt.Printf("k: %s, v: %d\n", key, value)
	}
}
```

**map小结：**


***1. 显示声明、未初始化值时为nil***
***2. 使用new函数生成的map为nil***
***3. nil的map只读，无法写入***
***4. map读取，如果没有要读取的key，则返回key对应类型的零值***
***5. delete时如果map为nil或者key不存在则什么也不做***
***5. map 并不是一个线程安全的数据结构。同时读写一个 map 是未定义的行为，如果被检测到，会直接 panic。***
***6. 使用for...range遍历nil的slice不会迭代，也不会报错***


### channel

一个未被make初始化的channel是nil的，channel是通过make来初始化的，make在创建channel时底层调用了`makechan`函数，返回值是一个结构体指针。
> func makechan(t *chantype, size int64) *hchan

这里不去讨论详细的channel用法，仅仅对nil的情况做一下阐述。

1. 读写一个nil的channel会造成永远阻塞。

	```go
	var ch chan int

	<- ch // block
	```

	```go
	var ch chan int

	ch <- 1  // block
	```

2. 关闭一个nil的channel会产生panic。

	```go
	var ch chan int

	close(ch) // panic: close of nil channel
	```

3. 往一个已经关闭的channel发送数据会产生panic

	```go
	ch := make(chan int)

	close(ch)

	ch <- 1 // panic: send on closed channel
	```

4. 从一个已关闭的channel接收数据会收到最后发送的数据或者对应类型的零值

	```go
		ch := make(chan int, 1)
		ch <- 1
		close(ch)
		fmt.Println(<-ch) // Output: 1
		fmt.Println(<-ch) // Output: 0
	```

5. for...range语句会自动感知channel的关闭，但遇到nil会永远阻塞

	利用for...range优雅退出协程：

	```go
	go func(in <-chan int) {
		// Using for-range to exit goroutine
		// range has the ability to detect the close/end of a channel
		for x := range in {
			fmt.Printf("Process %d\n", x)
		}
	}(inCh)
	```

	遍历nil的通道：
	```go
	var tc  chan int
	// 永远阻塞
	for v := range tc {
		fmt.Println(v)
	}
	```

### function

function和map、channel一样底层是一个指针，如果一个函数没有被初始化，那么它就是nil的

```go
var myFun func(int) string

fmt.Println(myFun == nil) // Output: true
```

### interface

`interface`是比较有意思的一个类型，也是go能够具有`面向对象特征`以及`多态`基石，它是一个结构体，包含了`动态类型`和`动态值`。

![interface](http://qiniu.liupzmin.com/interface.png)

对于一个接口的零值就是它的类型和值的部分都是nil， 只有`都为nil`的情况下`接口值 == nil`才成立。

![nil interface](http://qiniu.liupzmin.com/nil-interface.jpg)

1. 调用一个空接口值上的任意方法都会产生panic

	```go
	var w io.Writer

	fmt.Println(w == nil) // Output: true

	w.Write([]byte("hello")) // panic: nil pointer dereference
	```

2. 一个不包含任何值的nil接口值和一个刚好包含nil指针的接口值是不同的(`此时nil不是nil`)

	```go
	func main() {
		var buf *bytes.Buffer
		f(buf) // NOTE: subtly incorrect!
	}

	// If out is non-nil, output will be written to it.
	func f(out io.Writer) {
		// ...do something...
		if out != nil {
			out.Write([]byte("done!\n"))
		}
	}
	```
	上述示例，虽然我们给函数`f`传入了一个nil的指针（`*bytes.Buffer`），但是go将out的动态类型设为了`*bytes.Buffer`，动态值设为nil，意思就是out变量是一个包含了nil指针值的非nil接口，所以`out != nil`仍然为`true`，nil经过一道赋值的关卡后已不再是nil。

![non nin interface with nil pointer](http://qiniu.liupzmin.com/non-nil-interface.png)

## the use of nil

`nil`并不总是为我们制造困难，有些时候也有其妙用。

1. `nil的指针可以作为合理的方法接收者`

	```go

	type PointerReciver int

	func (a *PointerReciver) showme(){
		fmt.Printf("Yeah, it works!")
	}

	func main() {
		var ta *PointerReciver
		ta.showme()		// Yeah, it works!
	}
	```

2. `nil的slice可以正常的append`

	```go
	var s []int
	for i:=0; i <10; i++ {
		s = append(s,i)
	}
	```

3. `nil的map是只读的， 当你需要一个空map参数时可以使用nil`

	```go
	func NewGet(url string, headers map[string]string) (*http.Request, error) {
		req, err := http.NewRequest(http.MethodGet, url, nil)
		if err != nil {
			return nil, err
		}
		for k, v := range headers {
			req.Header.Set(k, v)
		}
		return req, nil
	}
	```

	你可能想做如下调用，传入空的map

	```go
	req, err := NewGet(
			"http://google.com",
			map[string]string{},
		)
	```
	你只需传入一个nil即可：

	```go
	req, err := NewGet("http://google.com", nil)
	```

4. `nil的通道永远阻塞`

	有时候我们可以利用`nil通道的阻塞特性`，比如有如下代码：

	```go
	func merge(out chan<- int, a, b <-chan int) {
		for {
			select {
				case v := <-a:
					out <- v
				case v := <-b:
					out <- v
			}
		}
	}
	```

	这个函数不断的从通道a和b读取数据，然后写入out通道，如果a和b其中有通道关闭，根据关闭通道的特性，我们知道会从	读取到对应类型的零值，那么如何才能跳过已经close的分支呢？	
	对一个nil的channel发送和接收操作会永远阻塞，在select语句中操作nil的channel永远都不会被select到。
	```go
	case v, ok := <-a:
		if !ok {
			a = nil
			fmt.Println("a is now closed")
			continue
		}
		out <- v
	```

5. nil的接口

	不用多说了

	```go
	if err != nil {
		// do somthing
	}
	```

## 总结

本篇文章通过探索`new`和`make`的用法，揭示了go中初始化变量的一些规律，避免新手gopher使用过程中的困惑，对于这两个内置方法的使用时机，我个人的看法是：`如果你需要初始化一个slice、map、chan类型的变量，那么优先使用make`；`如果你需要一个指针接收器或者明确需要一个指针的时候，new会是一个不错的选择`。

在查阅new和make的过程中，我们遭遇了恼人的`nil`，本文也通过一些浅薄的分析，总结出了nil的一些规律，希望能给阅读本文的人带来一些帮助，同时也作为个人学习中的笔记可以随时翻阅，加强理解。

**参考文章：**

1. [The Go Programming Language Specification](https://golang.org/ref/spec)
2. [video:Understanding nil](https://www.youtube.com/watch?v=ynoY2xz-F8s)
3. [nils in Go](https://go101.org/article/nil.html)
4. [Golang: Nil vs Empty Slice](https://medium.com/@habibridho/golang-nil-vs-empty-slice-87fd51c0a4d)
5. [深入学习golang(4)—new与make](https://www.cnblogs.com/hustcat/p/4004889.html)
6. [深度解密Go语言之slice](https://qcrao.com/2019/04/02/dive-into-go-slice/)
7. [深度解密Go语言之map](https://qcrao.com/2019/05/22/dive-into-go-map/)
8. [深度解密Go语言之channel](https://qcrao.com/2019/07/22/dive-into-go-channel/)
9. [深度解密Go语言之关于interface的 10 个问题](https://qcrao.com/2019/04/25/dive-into-go-interface/)
