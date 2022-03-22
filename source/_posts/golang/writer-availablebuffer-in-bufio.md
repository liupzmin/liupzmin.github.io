---
title: go 1.18 bufio 包中的 Writer.AvailableBuffer
date: 2022-03-22 10:08:30
tags:
	- Go1.18	
	- bufio
categories: golang
---

go 1.18 于近日发布，带来了 go 历史上最大的一次语言级改变——`泛型`！但本文只聚焦于本次发布中标准库 bufio 包中的一个小小的改变——**Writer.AvailableBuffer**。go 每次版本发布都会伴随着标准库的些许变动，本次发布即在 bufio 包中增加了一个 `Writer.AvailableBuffer` 方法。

该方法的添加源于一条名为 [bufio: add Writer.AvailableBuffer](https://github.com/golang/go/issues/47527) 的 issue。作者认为 go 中很多 appendX 类型的 API 在与 `bufio.Writer` 一起工作的时候比较低效，原因在于 `Write(p []byte) (nn int, err error)` 方法只接受 `[]byte` ，却不向外提供 `[]byte`。这在一定程度上需要调用者自行分配内存，然后传给 `write` 使其再进行 `copy`，所以这里固定有一次内存的 `allocation` 和 `copy`。

作者的提议是：由 bufio 的 `Writer` 向外暴露自己的 buffer 以为 appendx 函数使用：

```go
// AvailableBuffer returns an empty buffer with b.Available capacity.
// This buffer is intended to be appended to and
// passed to an immediately succeeding Write call.
func (b *Writer) AvailableBuffer() []byte {
    return b.buf[b.n:][:0]
}
```

`AvailableBuffer` 向外暴露了一个空的 `[]byte`， 但是其 `capacity` 是和 `Writer` 的 buffer 余量相同的，这意味着返回的 `[]byte` 和 `Writer` 共享同一个底层数组。那这样做的好处是什么呢？ 其好处就是在理想情况下，能够避免那固有的一次 `allocation` 和 `copy`，且看如下示例：

```go
package main

import (
	"bufio"
	"os"
	"strconv"
)

func main() {
	w := bufio.NewWriter(os.Stdout)
	for _, i := range []int64{1, 2, 3, 4} {
		b := w.AvailableBuffer()
		b = strconv.AppendInt(b, i, 10)
		b = append(b, ' ')
		w.Write(b)
	}
	w.Flush()
}

```

该例循环体内使用的 buffer `b` 都返回自 `AvailableBuffer`，此时没有额外的内存分配。理想情况下 append 的 byte 数量不超过 `Writer` 的 buffer 余量时， copy 会立即返回，因为 `b` 和 `Writer` 使用的同一个底层数组，不需要 copy，因此连 copy 的操作都省掉了。

极端情况下，由于 append 的扩张导致 b 的底层数组重新分配，那么 Write 也只是回到了其最初的工作方式。在很大程度上，该调整还是让 `Writer` 变得高效了不少！

那有什么不好的地方吗？ 唯一缺点应该就是暴露了 `bufio.Writer` 内部的 `buffer`。然而该包中的其它类型，诸如 `Reader`、 `Scanner` 已经向外提供了 `Reader.Peek`、 `Reader.ReadSlice`、以及 `Scanner.Bytes` 等对底层数组不安全的访问方法，故也不能独怪其罪。至少并没有破坏包的整体风格，而其利弊皆在人之为用！

