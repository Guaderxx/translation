# 分析文件引导优化

从Go1.20开始，Go编译器支持分析文件引导优化(profile-guided optimization *PGO*)以进一步优化构建。

注意：从Go1.20开始，PGO处于公开预览阶段(public preview)。
我们鼓励大家进行尝试，但仍有一些粗糙的角落（如下所示）可能会妨碍生产环境使用。
请向[https://go.dev/issue/new](https://go.dev/issue/new)报告您遇到的问题。
我们希望PGO在未来的版本中普遍可用。


目录：

- [概述](#概述)
- [收集分析文件](#收集分析文件)
- [使用PGO构建](#使用pgo构建)
- [注意](#注意)
- [FAQ - 常见问题](#faq常见问题)
- [附录：替代分析文件来源](#附录替代分析文件来源)


## 概述

分析文件引导优化（PGO），也称为反馈导向优化(feedback-directed optimization *FDO*)，
是一种编译器优化技术，它将来自应用程序的代表性运行的信息（分析文件）反馈给编译器，
用于应用程序的下一次构建，通过这些信息从而做出更明确的优化决策。
例如，编译器可能决定更积极的内链那些分析文件显示经常被调用的函数。

在Go中，编译器使用性能分析文件`CPU pprof profiles`作为输入分析文件，例如来自[runtime/pprof](https://pkg.go.dev/runtime/pprof)或是
[net/http/pprof](https://pkg.go.dev/net/http/pprof)的。

从Go1.20开始，一组具有代表性的Go程序的基准测试表明，使用PGO构建可将性能提高约2-4%。
我们预计性能提升通常会随着时间的推移而增加，因为在未来的Go版本中，额外的优化会利用PGO。


## 收集分析文件

Go编译器需要性能分析文件作为PGO的输入。
Go运行时生成的分析文件（例如来自[runtime/pprof](https://pkg.go.dev/runtime/pprof)或是
[net/http/pprof](https://pkg.go.dev/net/http/pprof)的）可以直接用作编译器输入。
也可以使用/转换来自其他分析文件系统的文件。
相关信息参考[附录：替代分析文件来源](#附录替代分析文件来源)

为了最佳的性能提升，分析文件代表应用程序生产环境中的实际行为非常重要。
使用不具代表性的分析文件可能会导致二进制文件在生产环境几乎没有性能提升。
因此，建议直接从生产环境中收集分析文件，这也是Go的PGO设计的主要方法。

典型的工作流程如下：

1. 构建并发布初始二进制文件（无PGO）。
2. 从生产环境收集分析文件。
3. 当需要发布更新的二进制文件时，从最新源构建并提供生产环境的分析文件。
4. 回到第2步。


GO PGO通常很健壮，可以在应用程序的分析文件版本和使用分析文件构建的版本之间倾斜，
以及使用从已经优化的二进制文件收集的分析文件进行构建。
这就是使这个迭代生命周期成为可能的原因。
关于这个工作流程的详细信息参见[AutoFDO](#自动反馈导向优化)。

如果很难或不可能从生产环境中收集（例如，分发给最终用户的命令行工具），
也可以从具有代表性的基准中收集。
注意，构建具有代表性的基准通常非常困难（随着应用程序的发展保持它们具有代表性）。
特别是，微基准测试通常不适合用于PGO分析，因为它们只运行应用程序的一小部分，
在应用于整个程序时产生的收益很小。



## 使用PGO构建

通过`go build -pgo`构建标志控制PGO分析文件选择。
将此标志设置为除了`-pgo=off`以外的任何值可启用PGO优化。

标准方式是将文件名为`default.pgo`的性能分析文件存储在需要配置的二进制文件的根包文件夹中，
并使用`go build -pgo=auto`进行构建，这会自动选择`default.pgo`文件。

建议直接在源仓库中提交分析文件，因为分析文件作为构建的输入，对于可重现（和高性能）构建很重要。
与源文件一起存储简化了构建体验，因为除了获取源文件外不需要其他步骤来获取分析文件。

注意：在Go1.20中，默认值为`-pgo=off`。未来的版本中可能会将默认值更改为`-pgo=auto`以使用带有PGO的`default.pgo`自动构建任何二进制文件。

注意：在Go1.20中，`-pgo=auto`仅适用于单个主包。如果构建多个主包(`go build -pgo=auto ./cmd/foo ./cmd/bar`)将导致构建错误。
参见[https://go.dev/issue/58099](https://go.dev/issue/58099)。

对于更复杂的场景（例如，同一个二进制文件的不同场景对应的不同分析文件，或是不能和源文件一起存储分析文件等），
你可以直接传递分析文件路径来使用（例如，`go build -pgo=/tmp/foo.pprof`）。

注意：传递给`-pgo`的路径会应用到所有主包。例如，`go build -pgo=/tmp/foo.pprof ./cmd/foo ./cmd/bar`
会同时将`foo.pprof`应用到`foo`和`bar`这两个二进制文件上，这通常不是你想要的。
通常不同的二进制文件应该有不同的分析文件，通过单独的`go build`调用传递。



## 注意

### 从生产环境收集代表性分析文件

你的生产环境是你的应用程序的代表性分析文件的最佳来源，就像在[收集分析文件](#收集分析文件)中描述的。

最简单的开始方式是将[net/http/pprof](https://pkg.go.dev/net/http/pprof)添加到您的应用程序中，
然后从您的服务的任意实例中获取`/debug/pprof/profile?seconds=30`。
这是开始实践的好方法，但在某些方面可能并不具有代表性：

- 获取分析文件的这个实例在它被分析时可能没有做任何事，即时它通常都会很忙。
- 交通模式可能会在一天中发生变化，从而使行为在一天中发生变化。
- 实例可能在执行长时间运行的操作（例如，5分钟执行操作A，然后5分钟执行操作B，等等）。一个30S的分析文件可能只涵盖单一操作类型。
- 实例可能不会收到请求的公平分配（某些实例比其他实例收到更多的一种类型的请求）


更稳健的策略是在不同时间从不同实例收集多个分析文件，以限制各个实例分析文件之间差异的影响。
然后可以将多个分析文件合并为一个分析文件给PGO使用。

很多组织运行“持续分析(continuous profiling)”服务，自动执行这种广泛范围内的抽样分析，
可以用作PGO的分析文件来源。


### 合并分析文件

`pprof`工具可以用来合并多个分析文件：  
`$ go tool pprof -proto a.pprof b.pprof > merged.pprof`

这种合并实际上是对输入中样本的直接的加和，无论分析文件的持续时间如何。
因此，在分析应用程序的一小段时间（例如，无限期运行的服务器）时，你可能希望确保所有分析文件具有相同的持续时间（即，所有的分析文件收集30S）。
否则，具有较长持续时间的分析文件将在合并的分析文件中过多出现。


### 自动反馈导向优化

Go PGO旨在支持“[AutoFDO](https://research.google/pubs/pub45290/)”风格的工作流程。

让我们仔细看看[收集分析文件](#收集分析文件)中描述的工作流程：

1. 构建并发布初始二进制文件（无PGO）。
2. 从生产环境收集分析文件。
3. 当需要发布更新的二进制文件时，从最新源构建并提供生产环境的分析文件。
4. 回到第2步。


这看似简单，还是有几个重要的地方要注意的：

- 开发始终在进行，因此分析版本的二进制文件的源代码（第二步）可能与构建的最新源代码（第三步）略有不同。
  Go PGO被设计为对此具有鲁棒性，我们称之为*source stability 源稳定性*。
- 这是一个闭环。也就是说，在第一次迭代后，二进制文件的分析文件版本已经使用前一次迭代的分析文件进行了PGO优化。
  Go PGO也被设计为对此具有鲁棒性，我们称之为*iterative stability 迭代稳定性*。


源稳定性是通过使用启发式方法将分析文件中的样本与编译源相匹配来实现的。
因此，对源代码的许多更改（例如添加新功能）对匹配现有代码没有影响。
当编译器无法匹配更改后的代码时，一些优化就会丢失，但是请注意，这是一种*graceful degradation 优雅降级*。
未能匹配的单个函数可能会失去优化机会，但总体来说PGO收益通常分布在许多函数中。
关于匹配和降级的更多详细信息，参见[源稳定性](#源稳定性与重构)部分。

*迭代稳定性*是为了防止连续PGO构建中的可变性能循环（例如，构建#1很快，构建#2很慢，构建#3很快，等等）。
我们通过性能分析文件来确认热门功能以进行优化。
理论上，PGO可以大大加快常用函数的速度，以至于它在下一个分析文件中不再显得常用且没有得到优化，从而再次变慢。
Go编译器对PGO优化采用保守的方法，我们认为这可以防止显著差异。
如果你的确观察到这种不稳定性，请在[https://go.dev/issue/new](https://go.dev/issue/new)上提交问题。

同时，源代码和迭代稳定性消除了对两阶段构建的需求，其中第一个未优化的构建被描述为金丝雀，然后使用PGO重建用于生产环境（除非需要绝对的峰值性能）。


### 源稳定性与重构

如上所述，Go的PGO尽最大努力继续将样本从旧分析文件匹配到当前源代码。
具体来说，Go在函数内使用行偏移量（例如，调用函数foo的第5行）。

很多常见的更改不会中断匹配，包括：

- 在热函数之外更改文件（在函数上方或者下方添加/更改代码）。
- 将函数移动到同一包中的另一个文件（编译器完全忽略源文件名）。

一些可能会中断匹配的改变：

- 热函数内的更改（可能会影响行偏移）
- 重命名函数（增加或改变方法的类型）（更改了符号名称）
- 将函数移动到另一个包（更改符号名称）


如果分析文件相对较新，那么差异可能只会影响少数热函数，从而限制未能匹配的函数错过优化的影响。
不过，随着时间的推移，退化会慢慢累积，因为代码很少重构回其旧形式，因此定期收集新的分析文件以限制生产环境中的源偏差很重要。

分析文件匹配可能会显著降低的一种情况是大规模重构，它会重命名很多函数或是在包之间移动它们。
这种情况下，在新分析文件显示新结构之前，你可能会受到短期性能影响。

对于机械重命名，理论上可以重写现有分析文件以将旧符号名称更改为新名称。
[github.com/google/pprof/profile](https://github.com/google/pprof/profile)包含以这种方式重写pprof分析文件所需的原语，
但在撰写本文时，不存在现成的工具。


### 新代码的性能

添加新代码或启用带有标志翻转的新代码路径时，该代码不会出现在第一次构建的分析文件中，
因此在收集反映新代码的新分析文件之前不会收到PGO优化。
在评估新代码的推出时请记住，初始版本并不代表其稳态性能。



## FAQ：常见问题

### 是否可以用PGO优化Go标准库？

是的。Go的PGO适用于整个程序。重建所有包以考虑潜在的分析文件引导优化，包括标准库包。


### 是否可以用PGO优化依赖包？

是的。Go的PGO应用于整个程序。重建所有包以考虑潜在的分析文件引导优化，包括依赖项中的包。这意味着您的应用程序使用依赖项的独特方式会影响应用于该依赖项的优化


### 具有非代表性分析文件的PGO是否会导致我的程序比没有PGO还慢？

应该不是。虽然不代表生产环境行为的分析文件会导致应用程序不常用部分的优化，但它不应该使应用程序的常用部分变慢。
如果你遇到PGO导致性能比没有PGO更差的程序，请在[https://go.dev/issue/new](https://go.dev/issue/new)提交问题。


### 对于不同的GOOS/GOARCH构建，我能使用相同的分析文件么？

是的。分析文件的格式在操作系统和架构配置中是等效的，因此它们可以在不同的配置中使用。
例如，从`linux/amd64`二进制文件收集的分析文件可能会在`windows/amd64`构建中使用。

也就是说，[上面](#自动反馈导向优化)讨论的源稳定性警告也适用于此。
在这些配置中存在差异的任何源代码都不会被优化。
对于大部分应用程序而言，绝大部分代码都是平台无关的，因此这种形式的退化是有限的。

举例来说，操作系统包中文件处理的内部结构在Linux和Windows之间是不同的。
如果这些功能在Linux分析文件中很常用，那么Windows等效函数不会获得PGO优化，因为它们和分析文件不匹配。

你可以合并不同GOOS/GOARCH构建的分析文件。参阅下一个问题深入了解这样做的权衡。


### 我应该如何处理单个二进制文件的不同工作负载类型？

这没有明显的选择。
用于不同类型工作负载的同一个二进制文件（例如，在一项服务中以读取密集型方式使用的数据库，
在另一项服务中以写入密集型方式使用）可能具有不同的热组件，它们受益于不同的优化。

有三种选项：

1. 为每个工作负载构建不同版本的二进制文件：使用来自每个工作负载的分析文件来构建特定于此工作负载的二进制文件。
   这可以为每个工作负载提供最佳性能，但可能会增加处理多个二进制文件和分析文件源的操作复杂性。
2. 仅使用来自“最重要”工作负载的分析文件构建单个二进制文件：选择“最重要”的工作负载（占用空间最大，
   对性能最敏感），并仅使用来自该工作负载的分析文件进行构建。
   这为选定的工作负载提供了最佳性能，并且仍然可以对其他工作负载提供适当的性能改进，
   通过优化跨工作负载共享的通用代码。
3. 合并来自不同工作负载的分析文件：从每个工作负载中获取分析文件（按总足迹加权）并将它们合并到一个用于构建单个
   通用分析文件的"fleet-wide"分析文件中，用于构建。这可能会为所有工作负载提供适当的性能改进。


### PGO如何影响构建时间？

启用PGO构建应该会导致包构建时间的增加，可测量，但很小。
可能比单个包构建时间更引人注目的是PGO分析文件适用于二进制文件中的所有包，
这意味着第一次使用分析文件需要重建依赖关系图中的每个包。
这些构建像任何其他构建一样被缓存，因此使用相同分析文件的后续增量构建不需要完全重建。

如果构建时间极度增加，请在[https://go.dev/issue/new](https://go.dev/issue/new)上提交问题。

注意：在Go1.20中，解析分析文件会增加大量开销，特别是对大型分析文件，这会显著增加构建时间。
可以在[https://go.dev/issue/58102](https://go.dev/issue/58102)中追踪，这会在未来的版本中解决。


### PGO如何影响二进制文件大小？

由于额外的函数关联，PGO可能会产生稍大的二进制文件。



### 附录：替代分析文件来源

Go运行时生成的性能分析文件（通过[runtime/pprof](https://pkg.go.dev/runtime/pprof)等）已经采用正确的格式可直接用于PGO的输入。
然而，组织可能有替代的首选工具（例如，Linux perf），或是现有的广泛范围的持续分析系统，它们希望与Go PGO一起使用。

如果转换为pprof格式，其他来源的分析文件可以和Go PGO一起使用，但它们必须遵循以下要求：

- 类型“CPU”和单位“count”的样本索引应该是0
- 样本应该代表样本位置的CPU时间的样本
- 分析文件必须符号化（必须设置[Function.name](https://github.com/google/pprof/blob/76d1ae5aea2b3f738f2058d17533b747a1a5cd01/proto/profile.proto#L208)）
- 样本必须包含内联函数的堆栈帧。如果省略内联函数，Go将无法保持迭代稳定性。
- 必须设置[`Function.start_line`](https://github.com/google/pprof/blob/76d1ae5aea2b3f738f2058d17533b747a1a5cd01/proto/profile.proto#L215)。
  这是函数开始的行号。也就是包含`func`关键字的行。Go编译器使用此字段来计算样本的行偏移量（`Location.Line.line - Function.start_line`）。
  **注意，许多现有的pprof转换器省略了该字段。**

注意：在Go1.20中，DWARF元数据省略了函数起始行（`DW_AT_decl_line`），这可能会使工具难以确定起始行。
可以通过[https://go.dev/issue/57308](https://go.dev/issue/57308)追踪，预计在Go1.21中修复。