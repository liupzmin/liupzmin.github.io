---
title: 译:探索 Go1.16 io/fs 包以提高测试性能和可测试性
date: 2021-02-28 10:08:30
tags:
	- Go1.16	
	- io/fs
categories: golang
---

原文出处：[Exploring "io/fs" to Improve Test Performance and Testability](https://www.gopherguides.com/articles/golang-1.16-io-fs-improve-test-performance)

## `io/fs`概述及其存在的必要性

​要理解为什么 `Go`在`1.16` 版本引入`io/fs`，就必须先要理解 **embedding（内嵌）**的基本原理。当开发一个工具的时候，嵌入那些日后需要被访问（寻址）的内容涉及到很多方面，但本文仅仅讨论其中之一。

​对于每个要嵌入静态文件的工具来说，其工作本质都大同小异。当它们运行的时候，每个静态文件都会被转换为字节，放入一个`.go`文件之中，最后被编译成二进制文件。一旦进行编译，工具本身就必须负责将**针对文件系统的调用转换为对一个虚拟文件系统的调用**。

​当运行嵌入了`assets`静态文件的程序后，代码访问这些文件的方式依然是针对文件系统的调用，我们必须把这种调用转换为一种虚拟调用（因为实际访问的文件内容已被转换为字节，并编译进程序本身）。此时，我们面临一个问题：如何在代码中确定一个调用是针对虚拟的`assets`还是真实的文件系统？

​想象一下这样一个工具：它会遍历一个目录，并返回所能找到的所有以`.go`结尾的文件名称。如果此工具不能和文件系统交互，那么它将毫无用处。现在，假设有一个 web 应用，它内嵌了一些静态文件，比如`images, templates, and style sheets`等等。那这个 Web 应用程序在访问这些相关`assets`时应使用虚拟文件系统，而不是真实文件系统。

​要分辨出这两种不同的调用，就需要引入一个供开发人员使用的API，该API可以指导该工具何时访问虚拟化，何时访问文件系统。这类API都各有特色，像早期的嵌入工具  [Packr](https://pkg.go.dev/github.com/gobuffalo/packr/v2)，它使用的就是自定义的 API。

```go
type Box
	func Folder(path string) *Box
	func New(name string, path string) *Box
	func NewBox(path string) *Box
	func (b *Box) AddBytes(path string, t []byte) error
	func (b *Box) AddString(path string, t string) error
	func (b *Box) Bytes(name string) []byte
	func (b *Box) Find(name string) ([]byte, error)
	func (b *Box) FindString(name string) (string, error)
	func (b *Box) Has(name string) bool
	func (b *Box) HasDir(name string) bool
	func (b *Box) List() []string
	func (b *Box) MustBytes(name string) ([]byte, error)
	func (b *Box) MustString(name string) (string, error)
	func (b *Box) Open(name string) (http.File, error)
	func (b *Box) Resolve(key string) (file.File, error)
	func (b *Box) SetResolver(file string, res resolver.Resolver)
	func (b *Box) String(name string) string
	func (b *Box) Walk(wf WalkFunc) error
	func (b *Box) WalkPrefix(prefix string, wf WalkFunc) error
```

​使用自定义 API 的好处就是工具开发者可以完全掌控用户体验。这包括使开发人员更轻松地管理需要在幕后维护的复杂关系。缺点也很明显，那就是使用者需要去学习这种新的 API。其代码也就严重依赖于这种自定义的 API，这使得它们难以随时间升级。

​另一种方式就是提供一种模拟标准库的 API ， [Pkger](https://pkg.go.dev/github.com/markbates/pkger@v0.17.1/pkging#File) 就是此例之一：

```go
type File interface {
	Close() error
	Name() string
	Open(name string) (http.File, error)
	Read(p []byte) (int, error)
	Readdir(count int) ([]os.FileInfo, error)
	Seek(offset int64, whence int) (int64, error)
	Stat() (os.FileInfo, error)
	Write(b []byte) (int, error)
}
```

​这种方式使用已知的、大家都熟悉的 API，会更容易吸引用户，而且也避免了再去学习新的 API 。

​Go 1.16`标准库引入的`io/fs`包就采用了此种方式，其优点就是使用了用户熟知的 API 接口，因此也就降低了学习成本，使得用户更加容易接受。

​但有其利必有其弊，虽然使用现有 API 迎合了用户使用习惯、增加了程序的兼容性，但同时也导致了大而复杂的接口。这亦是`io/fs`所面临的问题，不幸的是，要正确模拟对文件系统的调用，需要很大的接口占用空间，我们很快就会看到。

## 测试基于文件系统的代码

`io/fs`包不仅仅只是支撑`1.16` 版本**嵌入**功能这么简单，它带来的最大便利之一就是丰富了单元测试，它可以让我们编写更加易于测试的文件系统交互方面的代码。

除了增加代码的可测试性之外，`io/fs`还可以帮助我们编写更加易读的测试用例，并且在我们测试文件系统交互代码时拥有不寻常的性能表现。

为了更深入地了解`io/fs`包，我们来实现一段代码，它的功能是遍历一个给定的根目录，并从中搜索以`.go`结尾的文件。在循环遍历过程中，程序需要跳过一些符合我们预先设定前缀的目录，比如`.git` , `node_modules` , `testdata`等等。我们没必要取搜寻`.git` , `node_modules`文件夹，因为我们清楚它们肯定不会包含`.go`文件。一旦我们找到了符合要求的文件，我们就把文件的路径加入到一个列表中然后继续搜索。

```go
func GoFiles(root string) ([]string, error) {
	var data []string

	err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
		if err != nil {
			return err
		}
		base := filepath.Base(path)
		for _, sp := range SkipPaths {
			// if the name of the folder has a prefix listed in SkipPaths
			// then we should skip the directory.
			// e.g. node_modules, testdata, _foo, .git
			if strings.HasPrefix(base, sp) {
				return filepath.SkipDir
			}
		}

		// skip non-go files
		if filepath.Ext(path) != ".go" {
			return nil
		}

		data = append(data, path)

		return nil
	})

	return data, err
}
```

这个函数的执行结果将产生一个类似于下面这样的`slice`：

```go
[
	"benchmarks_test.go",
	"cmd/fsdemo/main.go",
	"cmd/fsdemo/main_test.go",
	"fsdemo.go",
	"fsdemo_test.go",
	"mock_file.go",
]
```

我现在提出的问题是：**我们该如何测试这段代码？**因为这段代码直接和文件系统交互，我们该如何保证在文件系统上呈现一个准确无误的测试场景呢？

鉴于测试方法会有很多种，在深入`io/fs`之前，我们先看一下最常见的两种方法，从而对比一下`io/fs`能带给我们怎样的便利。

## JIT Test File Creation

第一个测试文件系统代码的方法就是在运行时刻创建必须的文件夹结构。

> 本文将以`benchmark`的方式呈现单元测试，如此我们就可以对比各种测试方法的性能。这也是为何`setup`（创建测试用的文件结构）的代码会被包含在基准代码当中，我们基准测试的目标就是`setup`的过程。在此情况下，各种测试方法都不会改变`setup`的底层函数。

```go
func BenchmarkGoFilesJIT(b *testing.B) {
	for i := 0; i < b.N; i++ {

		dir, err := ioutil.TempDir("", "fsdemo")
		if err != nil {
			b.Fatal(err)
		}

		names := []string{"foo.go", "web/routes.go"}

		for _, s := range SkipPaths {
			// ex: ./.git/git.go
			// ex: ./node_modules/node_modules.go
			names = append(names, filepath.Join(s, s+".go"))
		}

		for _, f := range names {
			if err := os.MkdirAll(filepath.Join(dir, filepath.Dir(f)), 0755); err != nil {
				b.Fatal(err)
			}
			if err := ioutil.WriteFile(filepath.Join(dir, f), nil, 0666); err != nil {
				b.Fatal(err)
			}
		}

		list, err := GoFiles(dir)

		if err != nil {
			b.Fatal(err)
		}

		lexp := 2
		lact := len(list)
		if lact != lexp {
			b.Fatalf("expected list to have %d files, but got %d", lexp, lact)
		}

		sort.Strings(list)

		exp := []string{"foo.go", "web/routes.go"}
		for i, a := range list {
			e := exp[i]
			if !strings.HasSuffix(a, e) {
				b.Fatalf("expected %q to match expected %q", list, exp)
			}
		}

	}
}
```

​在`BenchmarkGoFilesJIT`测试用例中，我们使用`io/ioutil`包来为测试创建满足需求场景的临时文件夹和文件。此刻，意味着要创建包含`.go`文件的`node_modules`和`.git`目录，以便于确认这些`.go`文件不会出现在处理结果中。如果`GoFiles`函数正常工作的话，我们在结果集中将看到两个条目，`foo.go` 以及 `web/routes.go`。

​这种`JIT`方式有两大缺点：其一，随着时间的推移，编写和维护`setup`部分的代码将会变得非常麻烦，为测试用例做大量的`setup`本身也会引入更多的 bug。其二，也是最大的弊端，`JIT`测试会创建大量的文件和文件夹，这势必会在文件系统上产生大量的`i/o`竞争和`i/o`操作，从而让我们的任务性能非常低效。

```tex
goos: darwin
goarch: amd64
pkg: fsdemo
cpu: Intel(R) Xeon(R) W-2140B CPU @ 3.20GHz
BenchmarkGoFilesJIT-16										1470			819064 ns/op
```

## Pre-Existing File Fixtures

另一种测试`GoFiles`的方法是创建一个名为`testdata`的目录，并且在里面创建好测试场景所需的全部文件。

```shell
testdata
└── scenario1
		├── _ignore
		│   └── ignore.go
		├── foo.go
		├── node_modules
		│   └── node_modules.go
		├── testdata
		│   └── testdata.go
		└── web
				└── routes.go

5 directories, 5 files
```

使用这种方法，我们就可以清理掉很多我们的测试代码，让`GoFiles`函数指向事先准备好的已包含相应测试场景的文件夹。

```go
func BenchmarkGoFilesExistingFiles(b *testing.B) {
	for i := 0; i < b.N; i++ {

		list, err := GoFiles("./testdata/scenario1")

		if err != nil {
			b.Fatal(err)
		}

		lexp := 2
		lact := len(list)
		if lact != lexp {
			b.Fatalf("expected list to have %d files, but got %d", lexp, lact)
		}

		sort.Strings(list)

		exp := []string{"foo.go", "web/routes.go"}
		for i, a := range list {
			e := exp[i]
			if !strings.HasSuffix(a, e) {
				b.Fatalf("expected %q to match expected %q", list, exp)
			}
		}

	}
}
```

这种方法大大减少了测试的消耗，从而提高了测试的可靠性和可读性。与JIT方法相比，此方法呈现的测试速度也快得多。

```shell
goos: darwin
goarch: amd64
pkg: fsdemo
cpu: Intel(R) Xeon(R) W-2140B CPU @ 3.20GHz
BenchmarkGoFilesExistingFiles-16							9795			120648 ns/op
BenchmarkGoFilesJIT-16										1470			819064 ns/op
```

这种方法的缺点是为GoFiles函数创建可靠测试所需的文件/文件夹的数量和组合（意指数量和组合可能都很巨大）。到目前为止，我们仅仅测试了“成功”的情况，我们还没有为错误场景或其它潜在的情况编写测试。

使用这种方式，一个很常见的问题就是，开发者会逐渐的为多个测试重复使用这些场景（指testdata中的测试场景）。随时间推移，开发者并非为新的测试创建新的结构，而是去更改现有的场景以满足新的测试。这将测试全部耦合在了一起，是测试代码变得异常脆弱。

使用`io/fs`重写`GoFiles`函数，我们将会解决所有的问题！

## 使用  FS

​通过上面的了解，我们知道`io/fs`包支持针对`virtual file system`的实现（译者注：意指`io/fs`包提供了很多针对`fs.FS`的功能）。为了利用`io/fs`提供的功能，我们可以通过重写`GoFiles`函数让它接受一个`fs.FS`作为参数。在正式的代码中，我们可以调用[`os.DirFS`](https://pkg.go.dev/os/#DirFS)来获得一个由底层文件系统支持的`fs.FS`接口的实现。

​为了遍历一个`fs.FS`的实现，我们需要使用`fs.WalkDir `函数，`fs.WalkDir `函数的功能近乎等同于`filepath.Walk`函数。尽管这些差异很值得推敲一番，但这超出了本文的范围，因此我们将在以后的文章中另行阐述。

```go
func GoFilesFS(root string, sys fs.FS) ([]string, error) {
	var data []string

	err := fs.WalkDir(sys, ".", func(path string, de fs.DirEntry, err error) error {
		if err != nil {
			return err
		}

		base := filepath.Base(path)
		for _, sp := range SkipPaths {
			// if the name of the folder has a prefix listed in SkipPaths
			// then we should skip the directory.
			// e.g. node_modules, testdata, _foo, .git
			if strings.HasPrefix(base, sp) {
				return filepath.SkipDir
			}
		}

		// skip non-go files
		if filepath.Ext(path) != ".go" {
			return nil
		}

		data = append(data, path)

		return nil
	})

	return data, err
}
```

得益于`io/fs`包兼容性API带来的便利，`GoFilesFS`函数避免了昂贵的重写，仅需要很小的修改就可完工。

## 实现 FS

现在，该函数已更新为使用`fs.FS`，让我们看看如何为它编写测试。在此之前，我们先来实现`fs.FS`。

```go
type FS interface {
	// Open opens the named file.
	//
	// When Open returns an error, it should be of type *PathError
	// with the Op field set to "open", the Path field set to name,
	// and the Err field describing the problem.
	//
	// Open should reject attempts to open names that do not satisfy
	// ValidPath(name), returning a *PathError with Err set to
	// ErrInvalid or ErrNotExist.
	Open(name string) (File, error)
}
```

`Open`函数接收一个文件的路径，然后返回一个` fs.File `和一个`error`。如文档所述，需要满足某些关于错误的需求。

对于我们的测试来说，我们将会使用一个模拟文件类型的切片，并稍后将其实现为`fs.FS`，该切片还将实现所有本次测试所需的功能。

```go
type MockFS []*MockFile

func (mfs MockFS) Open(name string) (fs.File, error) {
	for _, f := range mfs {
		if f.Name() == name {
			return f, nil
		}
	}

	if len(mfs) > 0 {
		return mfs[0].FS.Open(name)
	}

	return nil, &fs.PathError{
		Op:   "read",
		Path: name,
		Err:  os.ErrNotExist,
	}
}
```

在`MockFS.Open`中，我们在已知文件列表中循环匹配请求的名称，如果匹配成功则返回该文件；如果没有找到，则尝试在第一个文件中递归打开。最后，如果没有找到，则按文档要求返回适当的`error`。

我们的`MockFS`目前还未实现完整，我们还需要实现`fs.ReadDirFS`接口来模拟文件。尽管`fs.ReadDirFS`文档未提及以下约束，但`fs.ReadDirFile`和`File.ReadDir`则需要它们。因此，它们也值得留意和实现。

```go
// ReadDir reads the contents of the directory and returns
// a slice of up to n DirEntry values in directory order.
// Subsequent calls on the same file will yield further DirEntry values.
//
// If n > 0, ReadDir returns at most n DirEntry structures.
// In this case, if ReadDir returns an empty slice, it will return
// a non-nil error explaining why.
// At the end of a directory, the error is io.EOF.
//
// If n <= 0, ReadDir returns all the DirEntry values from the directory
// in a single slice. In this case, if ReadDir succeeds (reads all the way
// to the end of the directory), it returns the slice and a nil error.
// If it encounters an error before the end of the directory,
// ReadDir returns the DirEntry list read until that point and a non-nil error.
```

尽管这些规则听起来令人困惑，但实际上，这种逻辑非常简单。

```go
func (mfs MockFS) ReadDir(n int) ([]fs.DirEntry, error) {
	list := make([]fs.DirEntry, 0, len(mfs))

	for _, v := range mfs {
		list = append(list, v)
	}

	sort.Slice(list, func(a, b int) bool {
		return list[a].Name() > list[b].Name()
	})

	if n < 0 {
		return list, nil
	}

	if n > len(list) {
		return list, io.EOF
	}
	return list[:n], io.EOF
}
```

## 实现 File 接口

我们已经完成了`fs.FS`的实现，但仍需要实现一组接口来满足`fs`包的需要。幸运的是，我们可以将所有接口实现到一个类型当中，从而使我们的测试更加简便。

> 继续之前，我要申明一点：我故意没有完全实现接口的文件读取部分，因为这将增加不必要的复杂度，而这些复杂度不是本文所需要的。所以我们将在后续的文章中探讨相关主题。

为了测试我们的代码，我们将要实现四个接口： `fs.File` , `fs.FileInfo` , `fs.ReadDirFile` , and `fs.DirEntry` 。

```go
type File interface {
	Stat() (FileInfo, error)
	Read([]byte) (int, error)
	Close() error
}

type FileInfo interface {
	Name() string
	Size() int64
	Mode() FileMode
	ModTime() time.Time
	IsDir() bool
	Sys() interface{}
}

type ReadDirFile interface {
	File
	ReadDir(n int) ([]DirEntry, error)
}

type DirEntry interface {
	Name() string
	IsDir() bool
	Type() FileMode
	Info() (FileInfo, error)
}
```

乍看之下，这些接口的体量之大似乎压人心魄。但是不用多虑，因为它们很多重叠的功能，所以我们可以把他们凝聚到一个类型当中。

```go
type MockFile struct {
	FS      MockFS
	isDir   bool
	modTime time.Time
	mode    fs.FileMode
	name    string
	size    int64
	sys     interface{}
}
```

`MockFile`类型包含一个`fs.FS`的实现`MockFS`，它将持有我们测试用到的所有文件。`MockFile` 类型中的其余字段供我们设置为其相应功能的返回值。

```go
func (m *MockFile) Name() string {
	return m.name
}

func (m *MockFile) IsDir() bool {
	return m.isDir
}

func (mf *MockFile) Info() (fs.FileInfo, error) {
	return mf.Stat()
}

func (mf *MockFile) Stat() (fs.FileInfo, error) {
	return mf, nil
}

func (m *MockFile) Size() int64 {
	return m.size
}

func (m *MockFile) Mode() os.FileMode {
	return m.mode
}

func (m *MockFile) ModTime() time.Time {
	return m.modTime
}

func (m *MockFile) Sys() interface{} {
	return m.sys
}

func (m *MockFile) Type() fs.FileMode {
	return m.Mode().Type()
}

func (mf *MockFile) Read(p []byte) (int, error) {
	panic("not implemented")
}

func (mf *MockFile) Close() error {
	return nil
}

func (m *MockFile) ReadDir(n int) ([]fs.DirEntry, error) {
	if !m.IsDir() {
		return nil, os.ErrNotExist
	}

	if m.FS == nil {
		return nil, nil
	}
	return m.FS.ReadDir(n)
}
```

像`Stat() (fs.FileInfo, error)`方法可以返回`MockFile `本身，因为它已经实现了`fs.FileInfo`接口，此为我们如何用一个`MockFile`类型实现众多所需的接口的一个例证！

## 使用 FS 进行测试

鉴于我们已经拥有了`MockFS` 和 `MockFile`，那么是时候为`GoFilesFS`函数编写测试了。依例，我们首先要为测试设置文件夹和文件结构。通过两个辅助函数`NewFile`和`NewDir`、以及使用切片直接构建一个`fs.FS`（指 `MockFS`）的便捷性，我们可以在内存中快速的构建出复杂的文件夹和文件结构。

```go

func NewFile(name string) *MockFile {
	return &MockFile{
		name: name,
	}
}

func NewDir(name string, files ...*MockFile) *MockFile {
	return &MockFile{
		FS:    files,
		isDir: true,
		name:  name,
	}
}

func BenchmarkGoFilesFS(b *testing.B) {
	for i := 0; i < b.N; i++ {
		files := MockFS{
			// ./foo.go
			NewFile("foo.go"),
			// ./web/routes.go
			NewDir("web", NewFile("routes.go")),
		}

		for _, s := range SkipPaths {
			// ex: ./.git/git.go
			// ex: ./node_modules/node_modules.go
			files = append(files, NewDir(s, NewFile(s+".go")))
		}

		mfs := MockFS{
			// ./
			NewDir(".", files...),
		}

		list, err := GoFilesFS("/", mfs)

		if err != nil {
			b.Fatal(err)
		}

		lexp := 2
		lact := len(list)
		if lact != lexp {
			b.Fatalf("expected list to have %d files, but got %d", lexp, lact)
		}

		sort.Strings(list)

		exp := []string{"foo.go", "web/routes.go"}
		for i, a := range list {
			e := exp[i]
			if e != a {
				b.Fatalf("expected %q to match expected %q", list, exp)
			}
		}

	}
}
```

本次` setup`的代码非常简单高效地完成了我们所需的工作，如果我们需要在测试中增加文件或文件夹，可以通过插入一行或两行来迅速完成。更重要的是，在尝试编写测试时，我们不会因复杂的`setup`代码而分心。

```shell
BenchmarkGoFilesFS-16										432418				2605 ns/op
```

## 总结

​使用`BenchmarkGoFilesJIT`方式，我们有很多直接操作`filesystem`的文件`setup`和`teardown `代码（译者注：指文件结构的创建和销毁），这会让测试代码本身引入很多潜在的`error`和`bug`。`setup`和`teardown `代码会让测试的重心偏移，其复杂性使得很难对测试方案进行更改。而且，这种方式的基准测试性能最差。

​不同的是，`BenchmarkGoFilesExistingFiles`方式使用预先在`testdata`中准备好的文件结构场景。这是的测试过程不再需要`setup`代码，仅仅需要为测试代码指明场景在磁盘中的位置。这种方式还有其它便利之处，例如其使用的是可以用标准工具轻松编辑和操纵的真实文件。与`JIT`方式相比，因其使用了已存在的场景数据，这极大地增加了测试的性能。其成本是需要在`repo`中创建和提交很多的场景数据，而且这些场景数据很容易被其他的测试代码滥用，最终导致测试用例变得脆弱不堪。

​这两种方式都有其它的一些缺陷，比如难以模拟大文件、文件的权限、错误等等，而`io/fs`，可以帮我们解决这些问题！

​我们已经看到了如何通微小的代码改动来使用`io/fs`包，得益于此，我们的测试代码变得更易于编写。这种方式不需要`teardown`代码，设置场景数据就像为切片追加数据一样简单，测试中的修改也变得游刃有余。我们的`MockFile`类型可以让我们像`MockFS`类型一样模拟出文件的大小、文件的权限、错误甚至更多。最重要的是，我们看到，通过使用io / fs并实现其接口，与JIT测试相比，我们可以将文件系统测试的速度提高300％以上。

```go
goos: darwin
goarch: amd64
pkg: fsdemo
cpu: Intel(R) Xeon(R) W-2140B CPU @ 3.20GHz
BenchmarkGoFilesFS-16										432418				2605 ns/op
BenchmarkGoFilesExistingFiles-16							  9795			  120648 ns/op
BenchmarkGoFilesJIT-16										  1470			  819064 ns/op
```

​虽然本文介绍了如何使用新的`io/fs`包来增强我们的测试，但这只是该包的冰山一角。比如，考虑一个文件转换管道，该管道根据文件的类型在文件上运行转换程序。再比如，将.md文件从Markdown转换为HTML，等等。使用`io/fs`包，您可以轻松创建带有接口的管道，并且测试该管道也相对简单。 Go 1.16有很多令人兴奋的地方，但是，对我来说，`io/fs`包是最让我兴奋的一个。