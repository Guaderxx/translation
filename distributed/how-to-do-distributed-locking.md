# 如何构建分布式锁

> [原文](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)  
> Martin Kleppmann  --  2016.02.08

作为[我的书](http://dataintensive.net/)的研究的一部分，我在Redis网站上发现了一种名为[Redlock](http://redis.io/topics/distlock)的算法。
该算法声称基于Redis实现容错分布式锁（或者更确切的说，[租约](https://pdfs.semanticscholar.org/a25e/ee836dbd2a5ae680f835309a484c9f39ae4e.pdf)），该页面也希望分布式系统人员提供反馈。
该算法本能的在我的脑海深处敲响了警钟，所以我花了一些时间思考它并写下了这些笔记。

既然已经有[超过10个独立的Redlock实现](https://redis.io/docs/manual/patterns/distributed-locks/)，我们也不知道谁在依赖这个算法，我认为公开分享我的笔记是值的的。
我不会深入探讨Redis的其他方面，其中一些已经在[别处](https://aphyr.com/tags/Redis)受到批评。

在详细介绍Redlock之前，让我先说一下，我非常喜欢Redis，过去我在生产项目中成功的的使用过它。
我认为它非常适合在您想要在服务器之间共享一些瞬态的，近似的，快速变化的数据的情况，以及如果您出于某种原因偶尔丢失这些数据也不是什么大问题的情况。
例如，一个好的用例是维护每个IP地址的请求计数器（用于速率限制目的）和每个用户ID的不同IP地址集（用于滥用检测）。

但是，Redis已经逐渐进入了对一致性和持久性要求较高的数据管理领域--这让我很担心，因为这不是Redis的设计初衷。
可以说，分布式锁就是这些领域之一。让我们深入了解一下。



## 你用那个锁做什么？

锁的目的是确保在可能尝试做同一件事的多个节点中，只有一个节点真正做了（至少一次只有一个）。
这个工作可能是往共享的存储系统中写入一些数据，执行一些计算，调用一些外部API，或是类似的事。
在高层次上，你可能希望在分布式应用程序使用锁的原因有两个：[性能或准确性](https://research.google.com/archive/chubby.html)。
为了区分这些情况，你可以想想如果锁失效了会发生什么：

- **性能：**    使用锁可以避免不会重复相同的工作（例如一些昂贵的计算）。
  如果锁失效，两个节点最终做了同样的工作，结果是成本略有增加（你最终向AWS多支付了5美分，而不是本来要支付的），
  或是一个小的问题（例如，一个用户最终收到两次相同的电子邮件通知）
- **准确性：**  锁可以防止并发进程互相干扰并弄乱系统状态。如果锁失效且两个节点同时处理一块数据，
  则会导致文件损坏，数据丢失，永久不一致，给患者服用错误剂量的药物或是其他一些严重问题。

两者都是想要锁的有效情况，但你需要非常清楚你正在处理的是两者中的哪一个。

我会争辩说，如果你只是为了效率目的使用锁，就没有必要承担Redlock的成本和复杂性，运行5个Redis服务器并检查大多数来获取锁。
你最好只使用单个Redis实例，也许可以异步复制到辅助实例，以防主实例崩溃。

如果你使用单个Redis实例，当你的Redis节点突然断电，或者出现其他问题，当然会损失一些锁。
但是，如果你只是将锁用作效率优化，并且崩溃不会经常发生，那就没有关系。
这种“没什么关系”的场景正是Redis的亮点所在。
至少如果你依赖于单个Redis实例，那么每个查看系统的人都清楚锁是近似的，并且仅用于非关键目的。

另一方面，Redlock算法，具有5个副本和多数表决，乍一看似乎适用于锁对准确性很重要的情况。
我会在下面各节中论证它不适合该目的。
对于本文的其它部分，我们将假设你的锁对于准确性很重要，并且如果两个不同的节点同时认为它们有相同的锁，那会是一个严重的错误。



## 用锁保护资源

让我们暂时搁置Redlock的细节，讨论一般情况下如何使用分布式锁（与所使用的特定锁算法无关）。
重要的是要记住分布式系统中的锁不像多线程应用程序中的互斥锁。
这是一个更复杂的野兽，由于不同节点和网络都可能以各种方式独立失败的问题。

例如，假设你有一个应用程序，其中客户端需要更新共享存储（例如HDFS或S3）中的文件。
客户端首先获取锁，然后读取文件，做一些变更，再将修改的文件写回，最后释放锁。
该锁可防止两个客户端同时执行此读取-修改-写入循环，这回导致更新丢失。
代码可能看起来像这样：

```
// THIS COIDE IS BROKER
function writeData(filename, data) {
    var lock = lockService.acquireLock(filename);
    if (!lock) {
        throw "Failed to acquire lock";
    }

    try {
        var file = storage.readFile(filename);
        var updated = updateContents(file, data);
        storage.writeFile(filename, updated);
    } finally {
        lock.release();
    }
}
```

不幸的是，即使你有一个完美的锁服务，上面的代码也会被破坏。
下图展示了如何以损坏的数据结束：

<img src="https://martin.kleppmann.com/2016/02/unsafe-lock.png" />

在这个例子中，获取锁的客户端在持有锁的同时暂停了很长一段时间 -- 例如，因为GC启动。
锁有超时时间（即它是租约），这总是个好主意，否则崩溃的客户端可能最终会永远持有锁并永远不会释放它。
但是，如果GC暂停持续时间超过租约到期时间，且客户端没有意识到它已经过期，它可能会继续进行一些不安全的更改。

这个错误不是理论上的，HBase曾有有过[这个问题](http://www.slideshare.net/enissoz/hbase-and-hdfs-understanding-filesystem-usage)。
通常，GC暂停特别短暂，但众所周知，“stop-the-world”GC暂停有时会持续[几分钟](https://blog.cloudera.com/blog/2011/02/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-1/) -- 肯定足以让租约到期。
即使是所谓的“并发”垃圾收集器，如HotSpot JVM's CMS，也不能完全与应用程序代码并行运行 -- 甚至它们需要时不时的[停止运行](http://mechanical-sympathy.blogspot.co.uk/2013/07/java-garbage-collection-distilled.html)。

你无法通过在写回存储之前插入对锁过期的检查来解决此问题。
请记住，GC可以在任何时间点暂停正在运行的线程，包括对你来说最不方便的时间点（在最后一次检查和写入操作之间）。

如果你因为编程语言运行时没有长时间的GC暂停而开心，那么还有很多其他原因可能导致你的进程暂停。
也许是你的进程试图读取一个尚未加载到内存中的地址，因此它会出现页面错误而暂停，直到从磁盘加载该页面。
也许你的磁盘实际上是EBS，因此在Amazon拥挤的网络上无意中读取变量变成了同步网络请求。
也许有很多其他进程在争用CPU，而你在[调度程序树](https://twitter.com/aphyr/status/682077908953792512)中遇到了一个黑色节点。
也许有人不小心向进程发送了SIGSTOP。总之，你的进程会暂停。

如果你还是不相信我关于进程暂停的说法，那么请考虑文件写入请求在到达存储服务之前可能会在网络中延迟。
以太网和IP等分组网络可能会任意延迟数据包，它们[确实如此](https://queue.acm.org/detail.cfm?id=2655736)：
在[Github的一个著名事件](https://github.com/blog/1364-downtime-last-saturday)中，数据包在网络中延迟了大约90秒。
这意味着可能一个应用程序进程发送一个写请求，它可能会在一分钟后到达存储服务器，此时租约已经到期。

即使在管理良好的网络，这种事情也可能发生。
你根本无法对事件做出任何假设，这就是为什么无论你使用哪种锁服务，上面的代码从根本上讲都是不安全的。



## 用防护使锁安全

这个问题的修复实际上非常简单：你需要在对存储服务的每个写入请求中包含一个*防护令牌*。
这种情况下，防护令牌只是一个数字，每次客户端获取锁时都会增加（由锁服务增加）。
下图说明了这一点：

<img src="https://martin.kleppmann.com/2016/02/fencing-tokens.png" />

客户端1获取租约并得到令牌33,然后它开始长时间的暂停，租约过期了。
客户端2获取租约并得到令牌34（这个数字总是在增加的），然后将它的写入发送到存储服务，包括令牌34。
之后，客户端1暂停结束并发送它的写入到存储服务，包含它的令牌33.
但是，存储服务记住了它已经处理了一个令牌数更大（34）的写入，所以会拒绝这个令牌数33的写入。

请注意，这需要存储服务器在检查令牌方面发挥积极作用，并拒绝令牌已倒退的任何写入。
但这并不是特别难，一旦你知道了诀窍。
且假设锁服务生成严格单调递增的令牌，这使得锁安全。
例如，如果你用ZooKeeper作为锁服务，你可以使用zxis或znode版本号作为防护令牌，这样就很好。

但是，这导致我们遇到Redlock的第一个大问题：它没有任何生成防护令牌的工具。
该算法不会产生任何保证在每次客户端获取锁时都会增加的数字。
这意味着即时该算法在其他方面是完美的，使用起来也不安全，因为在一个客户端暂停或其数据包延迟的情况下，你无法阻止客户端之间的竞争条件。

而且我不清楚如何更改Redlock算法以开始生成防护令牌。
它使用的随机数不提供所需的单调性。
仅仅在一个Redis节点保留一个计数器是不够的，因为该节点可能挂了。
在多个节点上保留计数器意味着它们会不同步。
你可能需要一个共识算法来生成防护令牌。（如果只[增加一个计数器](https://twitter.com/lindsey/status/575006945213485056)会很简单）。



## 使用时间解决共识

Redlock无法生成防护令牌这一事实应该已经足以成为在正确性取决于锁的情况下不使用它的充分理由。
但还有一些更进一步的问题值得讨论。

在学术文献中，这种算法最实用的系统模型是[具有不可靠故障检测器的异步模型](http://courses.csail.mit.edu/6.852/08/papers/CT96-JACM.pdf)。
通俗的说，这意味着算法不对时间进行假设：进程可能会暂停任意长度的时间，数据包可能会在网络中任意延迟，时钟可能会任意出错；
而算法仍被期望做正确的事。
鉴于我们上面的讨论，这些都是非常合理的假设。

算法可能使用时钟的唯一目的就是产生超时，避免在节点挂掉的情况下一直等下去。
但是超时不一定是准确的： 仅仅因为请求超时，并不意味着另一个节点肯定是宕机的；
也可能是网络有很大的延迟，或者你的本地时钟出问题了。
作为故障检测器使用时，超时只是故障的一种可能性。
（如果可以的话，分布式算法将完全没有时钟，但这样就[不可能达成共识](http://www.cs.princeton.edu/courses/archive/fall07/cos518/papers/flp.pdf)。
获取锁就像比较和设置操作，[需要达成共识](https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf)。）

请注意，Redis使用[gettimeofday](https://github.com/antirez/redis/blob/edd4d555df57dc84265fdfb4ef59a4678832f6da/src/server.c#L390-L404)而不是[单调时钟](http://linux.die.net/man/2/clock_gettime)来确定[键的到期时间](https://github.com/antirez/redis/blob/f0b168e8944af41c4161249040f01ece227cfc0c/src/db.c#L933-L959)。
`gettimeofday`的手册中[明确指出](http://linux.die.net/man/2/gettimeofday)，它所返回的时间会受到系统时间不连续跳动的影响--也就是说，
它可能突然向前跳几分钟，甚至向后跳（例如，如果时钟因为与NTP服务器相差太多而[被NTP踩到](https://www.eecis.udel.edu/~mills/ntp/html/clock.html)，或者如果时钟被管理员手动调整）。
因此，如果系统时钟在做奇怪的事情，很容易发生Redis中的键过期比预期的快的多或慢得多。

对于异步模型中的算法，这不是一个大问题：这些算法通常确保它们的安全属性始终保持不变，而[无需进行任何时序假设](http://www.net.t-labs.tu-berlin.de/~petr/ADC-07/papers/DLS88.pdf)。
只有活性属性取决于超时或其他一些故障检测器。
通俗的说，这意味着，即使系统中的时间安排到处都是（进程暂停，网络延迟，时钟前后跳动），
一个算法的性能可能会陷入地狱，但该算法绝不会做出错误的决定。

但是，Redlock不是这样的。它的安全性取决于很多时序假设：它假定所有Redis节点在过期前将密钥保存大约正确的时间长度；
与到期持续时间相比，网络延迟很小；并且该过程暂停比到期持续时间短得多。



## 用不好的定时器破坏Redlock

让我们看一些例子来证明Redlock对时序假设的依赖。
假设有一个系统，有5个Redis节点（A，B，C，D，E）和两个客户端（1,2）。
如果其中一个Redis节点的时钟向前跳会发生什么？

1. 客户端1在节点A，B，C上获取锁。由于网络问题，D，E无法连接。
2. 节点C的时钟向前跳，导致锁过期。
3. 客户端2在节点C，D，E上获取锁。由于网络问题，A，B无法连接。
4. 客户端1和2现在都相信它们获取了锁。

如果C在将锁保存到硬盘之前崩溃并立即重启，则可能发生类似的问题。
出于这个原因，Redlock文档[建议](http://redis.io/topics/distlock#performance-crash-recovery-and-fsync)将崩溃节点的重启至少延迟到最长活跃锁的持续时间。
但这种重启延迟再次依赖于相当准确的时间测量，如果时钟跳了就会失败。

好吧，也许你认为时钟跳是不现实的，因为你非常有信心正确配置NTP以只转换时钟。
这种情况下，让我们看一个进程暂停如何导致算法失败的示例：

1. 客户端1在节点A，B，C，D，E上请求锁。
2. 当对客户端1的响应正在进行时，客户端1进入了GC暂停。
3. 所有Redis节点上的锁过期了。
4. 客户端2在节点A，B，C，D，E上获取锁。
5. 客户端1完成GC，并收到来自Redis节点的响应，表明它已成功获取锁（进程暂停时它们被保存在客户端1的内核网络缓冲区中）。
6. 客户端1,2现在都相信它们持有锁。

请注意，尽管Redis是由C编写的，因此没有GC，但这对我们没有帮助：任何客户端可能遇到GC暂停的系统都有这个问题。
你只能通过防止客户端1在客户端2获取锁后在锁下执行任何操作来确保安全，例如使用上面的防护方法。

较长的网络延迟会产生与进程暂停等效的效果。
这可能取决于你的TCP用户超时 -- 如果你使超时明显短于Redis TTL，则延迟的网络数据包可能会被忽略，
但我们必须详细查看TCP实现才能确定。
此外，随着超时，我们又回到了时间测量的准确性。



## Redlock的同步性假设

这些示例表明Redlock仅在假设同步系统模型时才能正常工作 -- 
即具有以下属性的系统：

- 有界的网络延迟（你可以保证数据包总在某个保证的最大延迟内到达）
- 有界的进程暂停（换句话说，硬实时约束，你通常只在汽车安全气囊系统和类似的系统中发现）
- 有界的时钟误差（祈祷你不会从一个[糟糕的NTP服务器](http://xenia.media.mit.edu/~nelson/research/ntp-survey99/)上获取时间）

请注意，同步模型并不意味着完全同步的时钟：它意味着你假设网络延迟，
暂停和时钟偏移的[已知固定上限](http://www.net.t-labs.tu-berlin.de/~petr/ADC-07/papers/DLS88.pdf)。
Redlock假设延迟、暂停和漂移相对于锁的生存时间都是很小的；
如果时序问题变得和生存时间一样大，算法就会失败。

在一个表现良好的数据中心环境中，大多数时候都会满足时序假设 -- 这被称为[部分同步系统](http://www.net.t-labs.tu-berlin.de/~petr/ADC-07/papers/DLS88.pdf)。
但这够好么？一旦这些时序假设被打破，Redlock可能会违反其安全属性，假如在另一个客户到期前向一个客户授予租约。
如果你依赖于你的锁的正确性，“大部分时间”是不够的 -- 你需要它总是正确的。

有大量证据表明，为大多数实际系统环境假设同步系统模型是不安全的。
以Github的[90秒的数据包延迟](https://github.com/blog/1364-downtime-last-saturday)不断提醒自己。
Redlock不太可能通过[Jepsen](https://aphyr.com/tags/jepsen)测试。

另一方面，为部分同步系统模型（或带有故障检测器的异步模型）设计的共识算法实际上有可能起作用。
Raft、Viewstamped Replication、Zab和Paxos都属于这一类。
这样的算法必须放弃所有时序假设。
这很难：人们很容易假设网络，进程和时钟比实际情况更可靠。
但是在分布式系统的混乱现实中，你必须非常小心你的假设。



## 总结

我认为Redlock算法是一个糟糕的选择，因为它是“非驴非马”：
对于效率优化的锁来说，它是不必要的重量级且昂贵的，
但对于正确性取决于锁的情况来说，它不够安全。

特别是，该算法对计时和系统时钟做了危险的假设（基本上是假设一个同步系统，
其网络延迟和操作的执行时间是有界的），如果这些假设没有得到满足，
它就会违反安全属性。此外，它还缺乏生成防护令牌的设置（保护系统
免受网络或暂停进程的长时间延迟）。

如果你只是在尽力的基础上需要锁（作为一种效率优化，而不是正确性），
我建议坚持使用Redis的[直接的单节点锁算法](http://redis.io/commands/set)（条件性的set-if-not-exists
来获得锁，原子性的delete-if-value-matches来释放锁），
并在你的代码中非常清楚的记录，锁只是近似的，可能偶尔会失败。
不要费心去建立一个有5个Redis节点的集群。

另一方面，如果你在准确性方面需要锁，请不要用Redlock。
相反，请使用适当的共识系统，例如[ZooKeeper](https://zookeeper.apache.org/)，
可能通过实现一个[Curator recipe](http://curator.apache.org/curator-recipes/index.html)实现锁。
（至少，使用[具有合理事务保证的数据库](http://www.postgresql.org/)。）
并且请在锁下对所有资源访问强制使用防护令牌。

就像我开始说的，如果正确使用，Redis是一个非常优秀的工具。
上述任何一项都不会削弱Redis对其预期用途的实用性。
[Salvatore](http://antirez.com/)多年来一直非常专注于这个项目，它的成功是当之无愧的。
但是每种工具都有局限性，了解它们并相应的进行计划很重要。

如果你想了解更多，我会在[我的书的第8，9章](http://dataintensive.net/)中更详细的解释这个主题，现在可以从O'Reilly获得早期版本。
（上面的图摘自我的书）要学习如果使用ZooKeeper，我推荐[Junqueira和Reed的书](http://shop.oreilly.com/product/0636920028901.do).
为了很好的介绍分布式系统的理论，我推荐[Cachin，Guerraoui和Rodrigues的教科书](http://www.distributedprogramming.net/)。

感谢Kyle Kingsbury、Camille Fournier、Flavio Junqueir和Salvatore Sanfilippo审阅本文的草稿。当然，任何错误都是我的。


**2016.02.09更新：**    Salvatore，Redlock的原作者，发布了对本文的[反驳](http://antirez.com/news/101)
（另请参阅[HN讨论](https://news.ycombinator.com/item?id=11065933)）。
他提出了一些很好的观点，但我坚持我的结论。如果我有时间，我可能会在后续帖子中详细说明，
但请形成你自己的意见 -- 并请参阅下面的参考资料，其中许多已经过严格的学术同行评审（不同于我们的任何一篇博文）。



## 参考

1. [租约：分布式文件缓存一致性的高效容错机制](https://pdfs.semanticscholar.org/a25e/ee836dbd2a5ae680f835309a484c9f39ae4e.pdf)  -- Cary G Gray && David R Cheriton -- 1989.09
2. [用于松耦合分布式系统的Chubby锁服务](https://research.google.com/archive/chubby.html) -- Mike Burrows -- 2006.11
3. [ZooKeeper：分布式进程协调](http://shop.oreilly.com/product/0636920028901.do) -- Flavio P Junqueira && Benjamin Reed -- 2013.11
4. [HBase和HDFC：理解HBase中文件系统的使用](http://www.slideshare.net/enissoz/hbase-and-hdfs-understanding-filesystem-usage) -- Enis Söztutar -- 2013.06
5. [使用MemStore-Local分配缓冲区避免Apache HBase中的Full GC：第一部分](https://blog.cloudera.com/blog/2011/02/avoiding-full-gcs-in-hbase-with-memstore-local-allocation-buffers-part-1/) -- Todd Lipcon -- 2011.02.24
6. [Java GC 提炼](http://mechanical-sympathy.blogspot.co.uk/2013/07/java-garbage-collection-distilled.html) -- Martin Thompson -- 2013.07.16
7. [网络可靠性](https://queue.acm.org/detail.cfm?id=2655736) -- Peter Bailis and Kyle Kingsbury -- 2014.07
8. [github事件](https://github.com/blog/1364-downtime-last-saturday) -- Mark Imbriaco -- 2012.12.26
9. [可靠分布式系统的不可靠故障检测器](http://courses.csail.mit.edu/6.852/08/papers/CT96-JACM.pdf) -- Tushar Deepak Chandra && Sam Toueg -- 1996.03
10. [一个错误进程不可能达成分布式共识](http://www.cs.princeton.edu/courses/archive/fall07/cos518/papers/flp.pdf) -- Michael J Fischer, Nancy Lynch, and Michael S Paterson -- 1985.04
11. [无等待同步](https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf) -- Maurice P Herlihy -- 1991.01
12. [在存在部分同步的情况下达成共识](http://www.net.t-labs.tu-berlin.de/~petr/ADC-07/papers/DLS88.pdf) -- Cynthia Dwork, Nancy Lynch, and Larry Stockmeyer -- 1988.04
13. [可靠和安全的分布式编程简介](http://www.distributedprogramming.net/) -- Christian Cachin, Rachid Guerraoui, and Luís Rodrigues -- 2011.02
