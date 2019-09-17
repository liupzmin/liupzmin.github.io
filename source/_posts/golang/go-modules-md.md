---
title: 浅谈Go1.13中的modules
date: 2019-09-16 10:41:24
tags: 
    - golang
    - modules
categories: golang
---

> 前几天go1.13发布，modules默认开启，从此modules转正成为golang官方原生的包依赖管理方式；除了modules，go1.13中还增加了新的语法，如二进制、八进制、十六进制字面量表示法，defer性能的增强，新的errors等等，社区已有很多相关特性的论述文章；此文仅简单讨论一下go1.13中modules的一些改变，毕竟包的管理跟我们日常开发是息息相关的，行文仓促，若有不当之处，希望读者斧正。

**本文仅介绍modules在1.11/1.12/1.13等版本中的变化，并不介绍modules的使用，如需了解modules的详细使用方法，请参考官方文档或其他社区文章**

## Go Moudles

`modules`是go1.11推出的特性，官方称是`GOPATH`的替代品，是一个完整的支持包分发和版本控制的工具，使用modules，工作区不再局限于`GOPATH`之内，从而使构建更加可靠和可重复，但modules在go1.11版本中仅仅是一个实验性的功能，紧接着在go1.12中得到了增强，而刚刚发布的go1.13中得到了转正，`GOPATH`的作用进一步被弱化，Go Moudles开始大规模使用。

### 两个模式
对于modules这种模式官网有一个称呼是`Module-aware`，我不知道如何去翻译这个组合词，与之相对的，就是在`Module-aware mode`之前我们使用的包管理方式称为`GOPATH mode`，他们的区别如下：

- **GOPATH mode：** go command从`vendor`和`GOPATH`下寻找依赖,依赖会被下载至`GOPATH/src`

- **Module-aware mode:** go command不再考虑`GOPATH`，仅仅使用`GOPATH/pkg/mod`存储下载的依赖，并且是多版本并存

**注意：Module-aware开启和关闭的情况下，go get 的使用方式不是完全相同的。在 modules 模式开启的情况下，可以通过在 package 后面添加 @version 来表明要升级（降级）到某个版本。如果没有指明 version 的情况下，则默认先下载打了 tag 的 release 版本，比如 v0.4.5 或者 v1.2.3；如果没有 release 版本，则下载最新的 pre release 版本，比如 v0.0.1-pre1。如果还没有则下载最新的 commit。这个地方给我们的一个启示是如果我们不按规范的方式来命名我们的 package 的 tag，则 modules 是无法管理的。version 的格式为 v(major).(minor).(patch) ，**

在 modules 开启的模式下，go get 还支持 version 模糊查询，比如 > v1.0.0 表示大于 v1.0.0 的可使用版本；< v1.12.0 表示小于 v1.12.0  版本下最近可用的版本。version 的比较规则按照 version 的各个字段来展开。

除了指定版本，我们还可以使用如下命名使用最近的可行的版本：

- go get -u 使用最新的 minor 或者 patch 版本

- go get -u=patch 使用最新的 patch 版本

## GO111MODULE
在 1.12 版本之前，使用 Go modules 之前需要环境变量 GO111MODULE：

- GO111MODULE=off: 不使用 Module-aware mode。

- GO111MODULE=on: 使用 Module-aware mode，不会去 GOPATH 下面查找依赖包。

- GO111MODULE=auto或unset: Golang 自己检测是不是使用Module-aware mode。

go1.11时GO111MODULE=on有一个很不好的体验，就是go command依赖go.mod文件，也就是如果在module文件夹外使用go get等命令会报如下错误：
```shell
[root@k8s-node1 ~]# go get github.com/google/go-cmp
go: cannot find main module; see 'go help modules'
[root@k8s-node1 ~]# touch go.mod
[root@k8s-node1 ~]# go get github.com/google/go-cmp
go: cannot determine module path for source directory /root (outside GOPATH, no import comments)
```

这个情况在go1.12中得到了解决，可以在`module directory`之外使用`go` command。

但我个人比较喜欢不去设置`GO111MODULE`，根据官方描述在不设置`GO111MODULE`的情况下或者设为auto的时候，如果在当前目录或者父目录中有go.mod文件，那么就使用`Module-aware mode`， 而go1.12中，如果包位于`GOPATH/src`下，且`GO111MODULE=auto`, 即使有go.mod的存在，go仍然使用`GOPATH mode`:

```shell
[root@VM_0_6_centos test]# go get github.com/jlaffaye/ftp        
go get: warning: modules disabled by GO111MODULE=auto in GOPATH/src;
        ignoring go.mod;
        see 'go help modules'
```

这个现象在go1.13中又发生了改变.

## modules in go1.13

### GO111MODULE


> The GO111MODULE environment variable continues to default to auto, but the auto setting now activates the module-aware mode of the go command whenever the current working directory contains, or is below a directory containing, a go.mod file — even if the current directory is within GOPATH/src.

Go 1.13 includes support for Go modules. Module-aware mode is active by default whenever a go.mod file is found in, or in a parent of, the current directory.


可见，modules 在 Go 1.13 的版本下是默认开启的，GOPATH的地位进一步被弱化。

### GOPROXY

`GOPROXY`环境变量是伴随着modules而生的，在go1.13中得到了增强，可以设置为逗号分隔的url列表来指定多个代理，其默认值为`https://proxy.golang.org,direct`，也就是说google为我们维护了一个代理服务器，但是因为墙的存在，这个默认设置对中国的gopher并无卵用，应第一时间修改。

`go`命令在需要下载库包的时候将逐个试用设置中的各个代理，直到发现一个可用的为止。`direct`表示直连，所有`direct`之后的proxy都不会被使用，一个设置例子：

```shell
GOPROXY=https://proxy.golang.org,https://myproxy.mysite:8888,direct
```

`GOPROXY`环境变量可以帮助我们下载墙外的第三方库包，比较知名的中国区代理[goproxy.cn](https://github.com/goproxy/goproxy.cn)。当然，通过设置https_proxy环境变量设也可以达到此目的。但是一个公司通过在内部架设一个自己的goproxy服务器来缓存第三方库包，库包下载速度会更快,可以感觉到module有一点`maven`的意思了，但是易用性上还有很长的路要走。

### GOPRIVATE

使用`GOPROXY`可以获取公共的包，这些包在获取的时候会去https://sum.golang.org进行校验，这对中国的gopher来说又是一个比较坑的地方，Go为了安全性推出了Go checksum database（sumdb），环境变量为`GOSUMDB`，go命令将在必要的时候连接此服务来检查下载的第三方依赖包的哈希是否和sumdb的记录相匹配。很遗憾，在中国也被墙了，可以选择设置为一个第三方的校验库，也可更直接点将`GOSUMDB`设为`off`关闭哈希校验，当然就不是很安全了。

除了`public`的包，在现实开发中我们更多的是使用很多`private`的包，因此就不适合走代理，所以go1.13推出了一个新的环境变量`GOPRIVATE`，它的值是一个以逗号分隔的列表，支持正则（正则语法遵守 Golang 的 包 path.Match）。在`GOPRIVATE`中设置的包不走proxy和checksum database，例如：

```shell
GOPRIVATE=*.corp.example.com,rsc.io/private
```

### GONOSUMDB 和 GONOPROXY 

这两个环境变量根据字面意思就能明白是设置不进行校验和不走代理的包，设置方法也是以逗号分隔

### go env -w

可能是go也觉得环境变量有点多了，干脆为go env增加了一个选项`-w`，来设置全局环境变量，在Linux系统上我们可以这样用：

```shell
go env -w GOPROXY=https://goproxy.cn,direct
go env -w GOPRIVATE=*.gitlab.com,*.gitee.com
go env -w GOSUMDB=off
```

## 总结

就`1.13`中包管理的改变来看，有些乏善可陈，go的包管理很难让大多数开发者满意，当我看到越来越多的环境变量时，心里忍不住唾弃，使用这么多环境变量是一个多么蠢的方法，希望未来Go能给大家带来更好的包管理方式吧，就像Java的maven那样。

**参考文献：**

1. [Go 1.13 Release Notes](https://golang.org/doc/go1.13)
2. [Go 1.12 Release Notes](https://golang.org/doc/go1.12)
3. [Go 1.11 Release Notes](https://golang.org/doc/go1.11)
2. [Go Modules 不完全教程](https://mp.weixin.qq.com/s/v-NdYEJBgKbiKsdoQaRsQg)