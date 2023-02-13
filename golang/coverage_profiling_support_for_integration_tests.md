# 集成测试的覆盖率分析支持

> [原文 go.dev/testing/coverage](https://go.dev/testing/coverage/)

目录：

- [概述](#概述)
- [构建一个用于覆盖率分析的可执行文件](#构建一个用于覆盖率分析的可执行文件)
- [运行一个覆盖率测试的可执行文件](#运行一个用于覆盖率分析的可执行文件)
- [使用覆盖率数据文件](#使用覆盖率数据文件)
- [常见问题](#常见问题)
- [资源](#资源)
- [词汇表](#词汇表)


从Go1.20开始，Go支持从应用和集成测试、更大更复杂的测试中收集覆盖率分析文件。



## 概述

Go通过`go test -coverprofile=... <pkg_target>`命令在包单元测试级别收集覆盖率分析文件提供了简单易用的支持。
从Go1.20开始，用户现在可以为更大的[集成测试](#词汇表)收集覆盖率分析：
更重量级，更复杂的测试，对一个给定的应用程序二进制文件进行多次测试。

对于单元测试，收集覆盖率分析文件并生成报告需要两个步骤：运行`go test -coverprofile=...`，然后调用`go tool cover {-func,-html}`生成报告。

对于集成测试，一共需要三步：一步[构建](#构建一个用于覆盖率分析的可执行文件)，
一步[运行](#运行一个用于覆盖率分析的可执行文件)（这可能涉及到对构建步骤中的二进制文件进行多次调用），
最后一步[报告](#创建覆盖率分析报告)，如下所述。



## 构建一个用于覆盖率分析的可执行文件

在`go build`构建可执行文件时传入`-cover`标志来构建一个用于收集覆盖率分析的应用。
[下面](#如何选择用于分析的包)是一个调用`go build -cover`的例子。
目标可执行文件可以通过环境变量设置来运行并捕获覆盖率分析（查看下一节[运行](#运行一个用于覆盖率分析的可执行文件)）。


### 如何选择用于分析的包

在一个给定的`go build -cover`调用，Go命令会在主包中选择包用于覆盖率分析；
其他涉及到构建的包（`go.mod`中列出的依赖，或是部分Go标准库）默认不会包含。

例如，这里有一个玩具程序，包含主包，一个本地主模块包`greetings`以及一些模块外部引入的包，
包括（除其他外）`rsc.io/quote`以及`fmt`（[完整程序代码](https://go.dev/play/p/VSQJN8xkkf-?v=gotip)）。

```
$ cat go.mod

module mydomain.com

go 1.20

require rsc.io/quote v1.5.2

require (
    golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c // indirect
    rsc.io/sampler v1.3.0 // indirect
)


$ cat myprogram.go
package main

import (
    "fmt"
    "mydomain.com/greetings"
    "rsc.io/quote"
)

func main() {
    fmt.Printf("I say %q and %q\n", quote.Hello(), greetings.Goodbye())
}


$ cat greetings/greetings.go
package greetings

func Goodbye() string {
    return "see ya"
}


$ go build -cover -o myprogram.exe .
$
```

如果你用`-cover`这个命令行标记构建程序，只有两个包会被包含在分析里：`main`和`mydomain.com/greetings`；
其他的依赖包会被排除。

想要对哪些包可以包含进覆盖率分析有更多控制的用户可以构建时用`-coverpkg`标志。例如：

```bash
go build -cover -o myprogramMorePkgs.exe -coverpkg=io,mydomain.com,rsc.io/quote .
```

在上面的构建中，`mydomain.com`的主包，`rsc.io/quote`以及`io`包都会被用于分析；
然而`mydomain.com/greetings`没有被列出，它会被分析排除掉，尽管它位于主模块中。



## 运行一个用于覆盖率分析的可执行文件

用`-cover`构建的可执行文件在执行完时，会输出分析数据文件到通过环境变量`GOCOVERDIR`指定的文件夹。例如：

```
$ go build -cover -o myprogram.exe myprogram.go

$ mkdir somedata
$ GOCOVERDIR=somedata ./myprogram.exe
I say "Hello, world." and "see ya"

$ ls somedata
covcounters.c6de772f99010ef5925877a7b05db4cc.2424989.1670252383678349347
covmeta.c6de772f99010ef5925877a7b05db4cc

$
```

注意写入到文件夹`somedata`的这两个文件：这些（二进制）文件包含了覆盖率分析结果。
查看接下来的[报告](#创建覆盖率分析报告)部分，了解更多如何从这些数据文件中产生人类可读的结果。

如果环境变量`GOCOVERDIR`未指定，这个可执行文件依然正常执行，但会抛出一个警告。例如：

```
$ ./myprogram.exe
warning: GOCOVERDIR not set, no coverage data emitted
I say "Hello, world." and "see ya"
$
```


### 涉及多次运行的测试

在很多情况下，集成测试可能会涉及多个程序运行；当程序构建时用了`-cover`标志，每次运行都会生成一个新的数据文件。例如：

```
$ mkdir somedata2
$ GOCOVERDIR=somedata2 ./myprogram.exe          // first run
I say "Hello, world." and "see ya"
$ GOCOVERDIR=somedata2 ./myprogram.exe          // second run
I say "Hello, world." and "see ya"
$ ls somedata2
covcounters.890814fca98ac3a4d41b9bd2a7ec9f7f.2456041.1670259309405583534
covcounters.890814fca98ac3a4d41b9bd2a7ec9f7f.2456047.1670259309410891043
covmeta.890814fca98ac3a4d41b9bd2a7ec9f7f
$
```

覆盖率数据输出文件有两种类型：元数据文件（包含每次运行都不变的项目，例如源文件名和函数名）
和计数器数据文件（记录程序执行的部分）。

在上面的例子，第一次运行生成两个文件（计数器和元数据），
而第二次运行只生成了一个计数器数据文件：既然元数据运行时不变，它就只需要输出一次。



## 使用覆盖率数据文件

Go1.20引入了一个新工具，"covdata"，可用于从GOCOVERDIR文件夹下读取和操作覆盖率数据文件。

Go的`covdata`工具有多个运行模式。一般是以下形式调用：

```bash
go tool covdata <mode> -i=<dir1,dir2,...> ...flags...
```

其中`-i`标志可提供一组文件夹用于读取，每个文件夹都来自执行覆盖检测可执行文件（通过GOCOVERDIR）。


### 创建覆盖率分析报告

这一节讨论如何使用`go tool covdata`从覆盖率数据文件生成人类可读的报告。


#### 报告覆盖语句的百分比

使用命令`go tool covdata percent -i=<directory>`来报告每个检测包的“覆盖语句百分比”指标。
使用上面[运行](#运行一个用于覆盖率分析的可执行文件)章的例子：

```
$ ls somedata
covcounters.c6de772f99010ef5925877a7b05db4cc.2424989.1670252383678349347
covmeta.c6de772f99010ef5925877a7b05db4cc

$ go tool covdata percent -i=somedata
    main    coverage: 100.0% of statements
    mydomain.com/greetings  coverage: 100.0% of statements
$
```

这里的“覆盖语句”百分比直接对应于`go test -cover`报告的百分比。


### 转换为传统的文本格式

你可以使用covdata textfmt选择器由`go test -coverprofile=<outfile>`将二进制覆盖数据文件转换为传统的文本格式。
然后通过`go tool cover -func`或是`go tool cover -html`将生成的文本文件创建为其他报告。例如：

```
$ ls somedata
covcounters.c6de772f99010ef5925877a7b05db4cc.2424989.1670252383678349347
covmeta.c6de772f99010ef5925877a7b05db4cc

$ go tool covdata textfmt -i=somedata -o profile.txt

$ cat profile.txt
mode: set
mydomain.com/myprogram.go:10.13,12.2 1 1
mydomain.com/greetings/greetings.go:3.23,5.2 1 1

$ go tool cover -func=profile.txt
mydomain.com/greetings/greetings.go:3:  Goodbye     100.0%
mydomain.com/myprogram.go:10:       main        100.0%
total:                  (statements)    100.0%
```


### 合并

`go tool covdata`的`merge`子命令可以用来将多个数据文件夹合并为一个分析。

例如，一个程序可以运行在macOS和Windows上。
程序的作者可能会想将来自每个操作系统上单独运行的覆盖率分析文件组合到一个分析文件语料库中，以便生成跨平台的覆盖率总结。例如：

```
$ ls windows_datadir
covcounters.f3833f80c91d8229544b25a855285890.1025623.1667481441036838252
covcounters.f3833f80c91d8229544b25a855285890.1025628.1667481441042785007
covmeta.f3833f80c91d8229544b25a855285890

$ ls macos_datadir
covcounters.b245ad845b5068d116a4e25033b429fb.1025358.1667481440551734165
covcounters.b245ad845b5068d116a4e25033b429fb.1025364.1667481440557770197
covmeta.b245ad845b5068d116a4e25033b429fb

$ ls macos_datadir

$ mkdir merged

$ go tool covdata merge -i=windows_datadir,macos_datadir -o merged
$
```

上面的合并操作会将指定的输入文件夹中的数据组合并生成新的合并数据文件集合写入到文件夹“merged”下。


### 包的选择

大部分`go tool covdata`命令支持`-pkg`标记，用于选择指定的包作为操作的一部分；
`-pkg`的参数和之前的`-coverpkg`一样。例如：

```
$ ls somedata
covcounters.c6de772f99010ef5925877a7b05db4cc.2424989.1670252383678349347
covmeta.c6de772f99010ef5925877a7b05db4cc
$ go tool covdata percent -i=somedata -pkg=mydomain.com/greetings
    mydomain.com/greetings  coverage: 100.0% of statements
$ go tool covdata percent -i=somedata -pkg=nonexistentpackage
$
```

`-pkg`标志可用于给指定报告选择感兴趣的包的特定子集。



## 常见问题

### 如何才能对go.mod中所有的导入的包进行覆盖率检测

默认情况下，`go build -cover`会检测所有的主包，但不会检测从主模块外部导入的包（例如：标准库或是go.mod中例出的依赖）。
请求对所有非标准库依赖项进行检测的一种方法是将`go list`的输出提供给`-coverpkg`。
这是一个例子，还是用的上面的[示例程序](https://go.dev/play/p/VSQJN8xkkf-?v=gotip)：

```
$ go list -f '{{if not .Standard}}{{.ImportPath}}{{end}}' -deps . | paste -sd "," > pkgs.txt
$ go build -o myprogram.exe -coverpkg=`cat pkgs.txt` .
$ mkdir somedata
$ GOCOVERDIR=somedata ./myprogram.exe
$ go tool covdata percent -i=somedata
    golang.org/x/text/internal/tag  coverage: 78.4% of statements
    golang.org/x/text/language  coverage: 35.5% of statements
    mydomain.com    coverage: 100.0% of statements
    mydomain.com/greetings  coverage: 100.0% of statements
    rsc.io/quote    coverage: 25.0% of statements
    rsc.io/sampler  coverage: 86.7% of statements
$ 
```


### 在`GO111MODULE=off`的情况下可以使用`go build -cover`么

是的。`go build -cover`和`GO111MODULE=off`没有关联。
当在`GO111MODULE=off`模式下构建程序时，只有在命令行上特别指定为目标的包才会被检测以进行分析。
使用`-coverpkg`标志在分析文件中包含其他包。


### 如果程序崩溃了，覆盖率文件还会输出么？

`go build -cover`构建的程序只会在程序执行结束时，可能是调用了`os.Exit()`或是正常从`main.main`返回，才会输出完成的分析数据。
如果程序是未捕获的`panic`终止的，又或者是遇到致命异常（例如分段违规，除以0,等等）则运行期间执行的语句的分析文件数据会丢失。


### `-coverpkg=main`会选择我的主包进行分析么

`-coverpkg`标志接收一组导入路径，而不是一组包名。
如果你想选择主包进行覆盖率检测，请确认是导入路径，不是名。
例如（[示例程序](https://go.dev/play/p/VSQJN8xkkf-?v=gotip)）：

```
$ go list -m
mydomain.com
$ go build -coverpkg=main -o oops.exe .
warning: no packages being built depend on matches for pattern main
$ go build -coverpkg=mydomain.com -o myprogram.exe .
$ mkdir somedata
$ GOCOVERDIR=somedata ./myprogram.exe
I say "Hello, world." and "see ya"
$ go tool covdata percent -i=somedata
    mydomain.com    coverage: 100.0% of statements
$
```



## 资源 

- 介绍Go1.2中单元测试覆盖率的博客：
  - 单元测试的覆盖率分析作为Go1.2版本的一部分引入；参阅这篇[博客](https://go.dev/blog/cover)获取详细信息
- 文档：
  - [cmd/go](https://pkg.go.dev/cmd/go)包的文档描述了与覆盖率相关的构建和测试标志。
- 技术细节：
  - [设计稿](https://go.googlesource.com/proposal/+/master/design/51430-revamp-code-coverage.md)
  - [提案](https://golang.org/issue/51430)



## 词汇表

**单元测试：**  使用Go的`testing`包在指定的Go包关联的`*_test.go`文件中进行测试。

**集成测试：**  针对给定应用程序或是可执行文件的更全面，更重的测试。集成测试通常涉及构建一个程序或一组程序，然后使用多个输入和场景执行一系列程序运行，测试工具可能基于Go的测试包或是其他。
