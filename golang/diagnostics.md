# 诊断

> [原文 https://go.dev/doc/diagnostics](https://go.dev/doc/diagnostics)

目录：

- [介绍](#介绍)
- [分析](#分析)
- [追踪](#追踪)
- [调试](#调试)
- [运行时统计信息和事件](#运行时统计信息和事件)
  - [执行追踪器](#执行追踪器)
  - [GODEBUG](#godebug)



## 介绍

Go生态系统提供了大量API和工具来诊断Go程序中的逻辑和性能问题。
该文档总结了可用的工具，并帮助Go用户针对他们的特定问题选择适合的工具。

诊断解决方案可分为以下几类：

- **Profiling** 分析工具分析Go程序的复杂性和成本，例如其内存使用情况和频繁调用的函数，以识别Go程序的重要部分。
- **Tracing**   追踪是一种检测代码以分析整个调研或用户请求生命周期中的延迟的方法。
                追踪概述了每个组件对整个系统延迟的影响。它还可以横跨多个Go进程。
- **Debugging** 调试允许我们暂停Go程序并检查其执行情况。从而验证程序状态和流程。
- **Runtime statistics and events** 收集并分析运行时的状态和事件提供了Go程序健康状况的高级概览。
        指标的峰值/下降有助于我们识别吞吐量、利用率和性能的变化。

注意：一些诊断工具可能会互相干扰。例如，精确的内存分析会扭曲性能分析，goroutine阻塞分析会影响调度程序追踪。
单独使用工具从而获得更精确的性息。



## 分析

分析对于确认代码中的昂贵或是经常调用的代码块很有帮助。
Go运行时以[pprof可视化工具](https://github.com/google/pprof/blob/master/doc/README.md)预期的格式提供[分析数据](https://go.dev/pkg/runtime/pprof/)。
分析数据可以通过`go test`在测试时收集，或是通过[net/http/pprof](https://go.dev/pkg/net/http/pprof/)包提供的API。
用户需要收集分析数据并使用pprof工具来过滤和可视化顶层代码路径。

[runtime/pprof](https://go.dev/pkg/runtime/pprof)提供的预定义的分析：

- **cpu**:  性能分析确定程序在主动消耗CPU周期时将时间花在哪里（而不是sleep或等待I/O时）。
- **heap**: 堆分析报告内存分配样本；用于监视当前和历史内存使用情况，并检查内存泄漏。
- **threadcreate:** 线程创建分析报告了创建新操作系统线程的程序部分。
- **goroutine:**    goroutine分析报告了所有当前存在的goroutine的调用栈追踪。
- **block:**    块分析显示goroutines在何处阻塞等待同步原语（包括timer channels）。
    默认情况下不启用块分析；使用`runtime.SetBlockProfileRate`来启用。
- **mutex:**    mutex分析报告锁竞争。当你觉得你的CPU由于锁互斥竞争而没有充分利用时，用这个分析。
    默认情况下不启用这个，用`runtime.SetMutexProfileFraction`来启用。


### 有什么其他的工具可以用来分析Go程序？

在Linux, [perf tools](https://perf.wiki.kernel.org/index.php/Tutorial)可以用来分析Go程序。
Perf可以分析和展开cgo/SWIG代码以及内核，所以它有助于深入了解本机/内核性能瓶颈。
在macOS，可以使用[Instruments](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/)套件分析Go程序。


### 我可以分析已经部署的生产环境服务么？

是的。在生产环境分析程序很安全，但启用一些分析（例如，性能分析）会增加额外开销。
你应该会看到性能下降。可以在生产环境中打开分析之前通过测量分析器的开销来估计性能损失。
你可能希望定期分析你的生产环境服务。特别是对有多个副本的单进程系统，周期性的随机选择一个副本是很安全的。
选择一个生产环境进程，每Y秒分析X秒，并保存结果进行可视化和分析；然后定期重复。
可以手动或自动进行审查以发现问题。分析文件的收集会互相干扰，因此建议一次只收集一个分析文件。


### 可视化分析数据的最佳方式？

Go工具通过[go tool pprof](https://github.com/google/pprof/blob/master/doc/README.md)提供了分析数据的文本，图形和[callgrind](http://valgrind.org/docs/manual/cl-manual.html)可视化。
阅读[Profiling Go programs](https://blog.golang.org/profiling-go-programs)查看它们的运行情况。

<img style="width: 100%" src="https://go.dev/images/diagnostics/pprof-text.png" />

以文本形式列出最昂贵的调用。

<img src="https://go.dev/images/diagnostics/pprof-dot.png" />

将最昂贵的调用可视化为图形。

Weblist视图在HTML页面上逐行显示源代码中的昂贵部分。在下面的示例中，`runime.concatstrings`花费了530ms，同时清单中列出了每行的成本。

<img src="https://go.dev/images/diagnostics/pprof-weblist.png" />

将最昂贵的调用可视化为weblist。


另一种可视化分析数据的方式是[flame graph](http://www.brendangregg.com/flamegraphs.html)。（火焰图）
火焰图允许你在特定的根路径中移动，因此你可以放大/缩小特定的部分代码。
[upstream pprof](https://github.com/google/pprof)支持火焰图。

<img src="https://go.dev/images/diagnostics/flame.png" />

火焰图提供了可视化来发现最昂贵的代码路径。


### 我是否只能使用内置的分析？

除了运行时提供的内容以外，Go用户可以通过[pprof.Profile](https://go.dev/pkg/runtime/pprof/#Profile)创建他们的自定义分析，并使用现有的工具来检查它们。


### 我可以在不同的路径和端口上提供分析服务（`/debug/pprof/...`）么？

是的。`net/http/pprof`包默认在默认的mux上注册了这些处理函数，但你可以使用从包中导出的处理函数自行注册。

例如，下面的例子在`:7777/custom_debug_path/profile`上提供`pprof.Profile`的服务：

```go
package main

import (
    "log"
    "net/http"
    "net/http/pprof"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/custom_debug_path/profile", pprof.Profile)
    log.Fatal(http.ListenAndServe(":7777", mux))
}
```



## 追踪

追踪是一种检测代码以分析整个调用链生命周期中的延迟的方法。Go提供了[golang.org/x/net/trace](https://godoc.org/golang.org/x/net/trace)包作为每个Go节点的最小追踪后端，并提供带有一个简单仪表板的最小检测库。
Go还提供了一个执行追踪器来追踪一个时间间隔内的运行时事件。

追踪允许我们：

- 在Go进程中检测和分析应用程序延迟。
- 测量一个长调用的指定调用的成本。
- 找出利用率和性能改进。如果没有追踪数据，瓶颈不会很容易找到。

在单体系统中，从程序的构建块中收集诊断数据相对容易。所有模块都在一个进程中并共享公共资源来报告日志，错误和其他诊断信息。
一旦你的系统超出了单个进程并开始变为分布式，就很难追踪从前端Web服务器开始到其所有后端的调用，直到将响应返回给用户。
这就是分布式追踪在检测和分析你的生产环境系统方面发挥重要作用的地方。

分布式追踪是一种检测代码以分析用户请求在整个生命周期中的延迟的方法。
当系统是分布式的，且传统的分析和调试工具无法扩展时，你可能希望使用分布式追踪工具来分析用户请求和RPC的性能。

分布式追踪允许我们：

- 在大型系统中对应用程序的延迟进行检测和分析
- 追踪用户请求生命周期内的所有RPC，查看只有在生产环境中才能看到的集成问题。
- 找出可以应用于我们系统的性能改进。在收集追踪数据之前，很多瓶颈并不明显。

Go生态系统为每个追踪系统提供了各种分布式追踪库，以及与后端无关的追踪库。


### 是否能够自动拦截每个函数调用并创建追踪？

Go没有提供能够自动拦截每个函数调用并创建持续追踪的方法。
你需要手动检测你的代码来创建、终止以及注释追踪范围。


### 我应该如何在Go库中传播追踪头信息？

你可以在`context.Context`中传播追踪标识符和标签。
目前还没有规范的追踪键或是追踪头的通用表示。
每个追踪提供者负责在其Go库中提供传播工具。


### 哪些来自标准库或运行时的其他底层事件可以包含在追踪中？

标准库和运行时正在尝试公开几个额外的API来通知底层的内部事件。
例如，`httptrace.ClientTrace`提供了API来追踪在生命周期中传出请求的底层事件。
目前正在努力从运行时执行追踪器中检索底层的运行时事件，并允许用户定义和记录他们的用户事件。



## 调试

调试是识别程序行为不当原因的过程。调试器让我们了解程序的执行流程和当前状态。
有好几种调试方式；本节只关注将调试器附加到程序和核心转储调试。

Go用户最常用的调试器：

- [Devle](https://github.com/go-delve/delve):    Delve是一个专为Go编程语言的调试器。它支持Go的运行时内容和内建类型。
    Delve正在尝试成为一个功能齐全的可靠Go程序调试器。
- [GDB](https://go.dev/doc/gdb):  Go通过标准Go编译器和Gccgo获得GDB支持。堆栈管理，线程与运行时包含与GDB预期的执行模型有很大不同的方面，
    即使程序是用gccgo编译的，他们也会混淆调试器。所以尽管GDB可以用来调试Go程序，但它并不理想，还可能会造成混乱。


### 调试器如何调试Go程序？

gc编译器执行函数内联与变量注册等优化。这些优化有时会使调试器进行调试变得更难。
目前正在努力提高为优化二进制文件生成的DWARF信息的质量。
在这些改进可用之前，我们建议在构建需要调试的代码时禁用优化。
以下命令构建了一个没有编译器优化的包：

```bash
go build -gcflags=all="-N -l"
```

作为改进工作的一部分，Go1.10引入了一个新的编译器标志`-dwarflacationlists`。
这个标志导致编译器添加位置列表，帮助调试器使用优化的二进制文件。
下面的命令构建了一个具有优化但是有DWARF位置列表的包：

```bash
go build -gcflags="-dwarflocationlists=true"
```


### 推荐的调试器用户界面？

尽管Delve和GDB都提供了CLI，但大多数编辑器和IDE都提供了调试专用的用户界面。


### 是否可以用Go程序进行事后调试？

核心转储文件是包含正在运行的进程的内存转储以及进程状态的文件。
它主要用于程序的事后调试，并在程序仍在运行时了解其状态。
这两种情况使核心转储的调试成为一种很好的诊断辅助手段，可以对生产服务进行后期分析。
可以从Go程序中获取核心文件并使用Delve或GDB进行调试，查看[核心转储的调试](https://go.dev/wiki/CoreDumpDebugging)页面，获得逐步的指导。



## 运行时统计信息和事件

运行时为用户提供内部事件的统计和报告，以诊断运行时级别的性能和利用率问题。
用户可以检测这些统计数据，以更好的了解Go程序的整体健康和性能。  
一些经常检测的统计信息和状态：

- [runtime.ReadMemStats](https://go.dev/pkg/runtime/#ReadMemStats)  报告包含堆分配与垃圾回收相关的指标。
    内存统计对于监控一个进程消耗了多少内存资源，该进程是否能够很好的利用内存以及捕捉内存泄漏很有帮助。
- [debug.ReadGCStats](https://go.dev/pkg/runtime/debug/#ReadGCStats) 读取关于GC的统计数据。查看有多少资源花费在GC暂停上很有用。
    它还报告了GC暂停的事件线和暂停事件的百分比。
- [debug.Stack](https://go.dev/pkg/runtime/debug/#Stack)    返回当前的堆栈追踪。堆栈追踪对于查看当前有多少个goroutine在运行，它们在做什么，以及它们是否被阻塞是非常有用的。
- [debug.WriteHeapDump](https://go.dev/pkg/runtime/debug/#WriteHeapDump)    暂停所有goroutine的执行，并允许你将堆转储到一个文件中。堆转储是Go进程在一特定事件的内存快照。包含了所有分配的对象以及goroutine, finalizer等等。
- [runtime.NumGoroutine](https://go.dev/pkg/runtime#NumGoroutine)   返回当前goroutine的数量。可以监控该值查看是否使用了足够的goroutine，或检测goroutine泄漏。



### 执行追踪器

Go附带了一个运行时执行追踪器，可以用来捕获各种运行时事件。
调度、系统调用、GC、堆大小以及其他由运行时收集的事件，且可通过`go tool trace`进行可视化。
执行追踪器是用来检测延迟和利用率问题的工具。你可以检查CPU的使用情况，以及网络或系统调用是否是导致goroutine抢占的原因。

追踪器对这些情况很有用：

- 了解你的goroutine如何执行。
- 了解一些例如GC运行这样的核心运行时事件。
- 确认不良的并行化执行。

然而，它对于识别热点，如分析内存或是CPU使用过量的原因，并不是很好。
首先使用分析工具来解决它们。

<img src="https://go.dev/images/diagnostics/tracer-lock.png" />

以上，`go tool trace`可视化显示执行开始时很好，然后就变成了序列化。
这表明，可能存在对共享资源的锁竞争，从而造成了瓶颈。

查看[go tool trace](https://go.dev/cmd/trace/)来收集并分析运行时追踪。



### GODEBUG

如果GODEBUG环境变量被相应的设置，运行时也会抛出事件和信息。

- `GODEBUG=gctrace=1`   在每次收集时打印GC事件，总结收集的内存量以及暂停的时长
- `GODEBUG=inittrace=1` 打印已完成包初始化工作的执行事件和内存分配信息的摘要。
- `GODEBUG=schedtrace=X`    每X毫秒打印调度事件。

GODEBUG环境变量可用于禁止在标准库和运行时中使用指令集扩展。

- `GODEBUG=cpu.all=off` 禁用所有可选的指令集扩展。
- `GODEBUG=cpu.extension=off`   禁止使用指定指令集扩展的指令。

*扩展 extension*是指令集扩展的小写名称，如`sse41`或`avx`。
