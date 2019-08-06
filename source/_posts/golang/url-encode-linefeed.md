---
title: The %0A in URL Encode
date: 2019-08-06 09:50:36
tags:
    - golang:net/url
    - url encode
categories: golang
---

## %0A的问题
在使用`Go`语言的`net/url`包进行编码组装url的时候，遇到如下报错：
```shell
2019-07-31 16:55:46.850 ERROR   executor/driver_rollback.go:41  encounter an error:bad response code: 404
github.com/glory-cd/agent/executor.(*HttpFileHandler).Get
        /home/liupeng/cdp/src/agent/executor/file_http.go:115
github.com/glory-cd/agent/executor.(*Client).Get
        /home/liupeng/cdp/src/agent/executor/client.go:66
github.com/glory-cd/agent/executor.Get
        /home/liupeng/cdp/src/agent/executor/filehandler.go:33
github.com/glory-cd/agent/executor.(*Roll).getCode
        /home/liupeng/cdp/src/agent/executor/driver_rollback.go:77
github.com/glory-cd/agent/executor.(*Roll).Exec
        /home/liupeng/cdp/src/agent/executor/driver_rollback.go:52
runtime.goexit
        /usr/lib/go/src/runtime/asm_amd64.s:1337
the kv is: {"url":"http://admin:xxxx@192.168.1.75:32749/test/1.0.0/Gateway.zip%0A"}
```

可见，最终组装成的`url`末尾多了`%0A`，从而导致**http**请求返回`404`，那么`%0A`是怎么来的呢？ 那就回溯一下url的组装过程吧。

我用来组装url的代码如下：

```go
//创建url.URL
func (hu *HttpFileHandler) newPostUrl() string {
	requestURL := new(url.URL)

	requestURL.Scheme = "http"

	requestURL.User = url.UserPassword(hu.client.User, hu.client.Pass)

	requestURL.Host = hu.client.Addr

	requestURL.Path += hu.client.RelativePath

	return requestURL.String()
}
```

其中`Path`部分是使用`hu.client.RelativePath`进行拼接的，`RelativePathient`的来源如下：

```go
func (d *driver) readServiceVerion() (string, error) {
	versionFile := filepath.Join(d.Dir, common.PathFile)
	path, err := ioutil.ReadFile(versionFile)
	if err != nil {
		return "", errors.WithStack(err)
	}

	return string(path), nil
}
```

发现`RelativePathient`是从文件中读取的，我这个文件中只有一行内容，那么再结合`url encode`来看，这个`%0A`就是一个`linefeed`，是一个换行符，那么处理方法就简单了，返回的时候`Trim`一下就可以解决换行和空格的问题。

```go
return strings.TrimSpace(string(path)), nil
```
接下来再简单聊一下`url encode`

## url encode

***
百分号编码（英语：Percent-encoding，又称：URL编码（英语：URL encoding）），是特定上下文的统一资源定位符 (URL)的编码机制. 实际上也适用于统一资源标志符（URI）的编码。也用于为application/x-www-form-urlencodedMIME准备数据，因为它用于通过HTTP的请求操作（request）提交HTML表单数据。
***

上面是`维基百科`对url编码的解释，通常如果一样东西需要编码，说明这样东西并不适合传输。原因多种多样，如Size过大，包含隐私数据，对于Url来说，之所以要进行编码，是因为Url中有些字符会引起歧义。

例如，Url参数字符串中使用key=value键值对这样的形式来传参，键值对之间以&符号分隔，如/s?q=abc& ie=utf-8。如果你的value字符串中包含了=或者&，那么势必会造成接收Url的服务器解析错误，因此必须将引起歧义的&和= 符号进行转义，也就是对其进行编码。

又如，Url的编码格式采用的是ASCII码，而不是Unicode，这也就是说你不能在Url中包含任何非ASCII字符，例如中文。否则如果客户端浏览器和服务端浏览器支持的字符集不同的情况下，中文可能会造成问题。

Url编码的原则就是使用安全的字符（没有特殊用途或者特殊意义的可打印字符）去表示那些不安全的字符。

### 语法构成

URI是统一资源标识的意思，通常我们所说的URL只是URI的一种。典型URL的格式如下所示。下面提到的URL编码，实际上应该指的是URI编码。

[rfc3986](https://tools.ietf.org/html/rfc3986)中解释URI的构成如下：

```
   The generic URI syntax consists of a hierarchical sequence of
   components referred to as the scheme, authority, path, query, and
   fragment.

      URI         = scheme ":" hier-part [ "?" query ] [ "#" fragment ]

      hier-part   = "//" authority path-abempty
                  / path-absolute
                  / path-rootless
                  / path-empty

The following are two example URIs and their component parts:

         foo://example.com:8042/over/there?name=ferret#nose
         \_/   \______________/\_________/ \_________/ \__/
          |           |            |            |        |
       scheme     authority       path        query   fragment
          |   _____________________|__
         / \ /                        \
         urn:example:animal:ferret:nose
```

### 哪些字符需要编码

RFC3986文档规定，Url中只允许包含英文字母（a-zA-Z）、数字（0-9）、-_.~4个特殊字符以及所有保留字符。 RFC3986文档对Url的编解码问题做出了详细的建议，指出了哪些字符需要被编码才不会引起Url语义的转变，以及对为什么这些字符需要编码做出了相 应的解释。

US-ASCII字符集中没有对应的可打印字符：Url中只允许使用可打印字符。US-ASCII码中的10-7F字节全都表示控制字符，这些 字符都不能直接出现在Url中。同时，对于80-FF字节（ISO-8859-1），由于已经超出了US-ACII定义的字节范围，因此也不可以放在 Url中。

保留字符：Url可以划分成若干个组件，协议、主机、路径等。有一些字符（:/?#[]@）是用作分隔不同组件的。例如：冒号用于分隔协议和主 机，/用于分隔主机和路径，?用于分隔路径和查询参数，等等。还有一些字符（!$&'()*+,;=）用于在每个组件中起到分隔作用的，如=用于 表示查询参数中的键值对，&符号用于分隔查询多个键值对。当组件中的普通数据包含这些特殊字符时，需要对其进行编码。

RFC3986中指定了以下字符为保留字符：! * ' ( ) ; : @ & = + $ , / ? # [ ]

不安全字符：还有一些字符，当他们直接放在Url中的时候，可能会引起解析程序的歧义。这些字符被视为不安全字符，原因有很多。

- 空格：Url在传输的过程，或者用户在排版的过程，或者文本处理程序在处理Url的过程，都有可能引入无关紧要的空格，或者将那些有意义的空格给去掉。
- 引号以及<>：引号和尖括号通常用于在普通文本中起到分隔Url的作用
- #：通常用于表示书签或者锚点
- %：百分号本身用作对不安全字符进行编码时使用的特殊字符，因此本身需要编码
- {}|\^[]`~：某一些网关或者传输代理会篡改这些字符
***
需要注意的是，对于Url中的合法字符，编码和不编码是等价的，但是对于上面提到的这些字符，如果不经过编码，那么它们有可能会造成Url语义 的不同。因此对于Url而言，只有普通英文字符和数字，特殊字符$-_.+!*'()还有保留字符，才能出现在未经编码的Url之中。其他字符均需要经过 编码之后才能出现在Url中。
***

### 如何对url进行编码

Url编码通常也被称为百分号编码（Url Encoding，also known as percent-encoding），是因为它的编码方式非常简单，使用%百分号加上两位的字符——0123456789ABCDEF——代表一个字节的 十六进制形式。Url编码默认使用的字符集是US-ASCII。例如a在US-ASCII码中对应的字节是0x61，那么Url编码之后得到的就 是%61，我们在地址栏上输入http://g.cn/search?q=%61%62%63，实际上就等同于在google上搜索abc了。又如@符号 在ASCII字符集中对应的字节为0x40，经过Url编码之后得到的是%40。

对于非ASCII字符，需要使用ASCII字符集的超集进行编码得到相应的字节，然后对每个字节执行百分号编码。对于Unicode字 符，RFC文档建议使用utf-8对其进行编码得到相应的字节，然后对每个字节执行百分号编码。如"中文"使用UTF-8字符集得到的字节为0xE4 0xB8 0xAD 0xE6 0x96 0x87，经过Url编码之后得到"%E4%B8%AD%E6%96%87"。

如果某个字节对应着ASCII字符集中的某个非保留字符，则此字节无需使用百分号表示。例如"Url编码"，使用UTF-8编码得到的字节是 0x55 0x72 0x6C 0xE7 0xBC 0x96 0xE7 0xA0 0x81，由于前三个字节对应着ASCII中的非保留字符"Url"，因此这三个字节可以用非保留字符"Url"表示。最终的Url编码可以简化 成"Url%E7%BC%96%E7%A0%81" ，当然，如果你用"%55%72%6C%E7%BC%96%E7%A0%81"也是可以的。

## 使用vim去除EOL

还可以使用vim去除文件末尾的换行

```
You can turn off the 'eol' option and turn on the 'binary' option to write
a file without the EOL at the end of the file:

    :set binary
    :set noeol
    :w
```