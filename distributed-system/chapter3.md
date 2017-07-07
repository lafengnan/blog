

# Raft算法

@2016.12.8



## Raft算法是什么

其实Raft提出来的根本原因就是Paxos太复杂了，论文读起来晦涩难懂，工程实践也不是很简单易行。这个世界好像就这样，很难好起来了。那些复杂的充满严密逻辑的理论和知识总是掌握在少数人手里，普罗大众只能一脸懵逼的看他们滔滔不绝，洋洋洒洒；而简单易行的理论与知识若不具备美到令人窒息的形态，就总让人怀疑其伟大的内涵。

在Raft的官方网站上赫然写着*Raft is a consensus algorithm that is designed to be easy to understand.*，这货发明出来完全就是为了让人更好的理解一致性算法，而且与Paxos算法在容错性和性能上完全等价。当然也有一些区别，Raft将一致性问题分解成相对独立的子问题，并在实践系统中完美地解决了所有的麻烦。作者希望Raft可以让更多的人了解一致性，从而可以开发出比现在更多的高质量的基于一致性的系统。

Raft算法和已有的一致性算法（最突出的是，Oki和Liskov的基于视图时间戳的复制Viewstamped Replication)有很多相似之处，但是也有很多独有的功能：

* Strong leader：Raft使用了比其他一致性算法更强大的leader形式。例如日志条目只能从leader节点流向其他服务器。这简化了日志复制的管理也使Raft更容易理解了。
* Leader election：Raft使用随机定时器选举leader。这样只需为其他一致性算法需要的心跳增加少量机制，同时还可以简单快速的解决冲突。
* Membership changes：Raft算法用于更改集群中的服务器集合的机制使用了一种新的*joint consensus*方式，这种方式下两个不同配置大部分内容在转换过程中会重叠。这就允许集合在配置更改过程中可以继续正常的运营。

## 基础

### 一致性

**一致性**是容错的分布式系统的基础问题。一致性涉及多台服务器如何对值达成一致。一旦这些服务器对某个值达做出了决议，那么这个决议就是最终决议。典型的一致性算法在大多数服务器处于可用状态时可以起作用，例如一个由5台服务器组成的集群，若有2台服务器宕机也可以持续工作；但是一旦有更多的服务器宕机，就会停止工作了（也不会反悔不正确的结果）。其实一致性问题就像美国总统选举，只要选举人团中的多数人都没有*休眠*，可以正常投票，那么即使多数人都给特朗普投了赞成票，那么特朗普就是铁打的新总统，这就是一个无法更改的最终决议；可是若选举人团中大多数人都嗑了金三胖的神药，个个都在*休眠*而无法投票，只剩下2个人可以投票，那么此刻无论投给谁，这结果都不能算数，也就是选举制度失效了，需要推迟大选，这样的结果也挺好，因为怎么着都不会选出一个自大狂或者重度面瘫患者来当总统的。

典型情况下，一致性问题都会在**复制的状态机**上下文中出现，状态机复制是一种构建容错系统的通用方法。每一台服务器都有一个状态机和一份日志。状态机是我们希望具备容错特性的模块，例如哈希表。即使集群中的少部分服务器都宕机了，它也让能客户端们以为自己是在和一个独立可靠的状态机交互。每个状态机从它的日志中获取输入命令，在哈希表的例子中，日志可能包括类似*set x = 3*这样的命令。一致性算法用于对服务器日志的命令达成一致。一致性算法必须保证如果有任何状态机以第$$n$$个顺序采用*set x = 3*这条命令，那么其他状态机的第$$n$$条命就不能与这条命令不同。结果就是每个状态机都处理相同的命令序列，从而产生相同的结果序列而达到相同的状态序列。



### 复制的状态机

一致性问题在典型情况下会在所谓*复制的状态机*上下文中出现，这种方式下，一组服务器集合上的状态机对相同的状态使用相同的副本，即使其中某些服务器宕机了也可以继续工作。

![replicated-state-machine结构](/resources/replicated-state-machine.png)

图-1

这种机制主要用于分布式系统中大量的容错问题。例如带有单个leader节点的大规模系统，像GFS，HDFS以及RAMCloud使用，都很典型的使用了一个独立的状态机来管理Leader选举和存储在Leader崩溃时可以幸免于难配置信息。复制的状态机示例有Google的Chubby和Zookeeper。

典型的复制状态机都是像图-1一样，用一个复制日志实现的。每台服务器存储一个包括一系列命令的日志，这些命令会被状态机依次执行。每个日志都以相同的顺序包括相同的命令，因此每个状态机都会用相同的顺序处理命令。由于状态机是确定的，每个状态机都会运算出相同的状态和相同的输出序列。

那么用什么来保证那些复制的日志的一致性呢？对，就是一致性算法。一致性算法就是用来保证日志的一致性的，因为状态机依赖日志，如果日志内容不一致那么不同的状态机是不可能计算出一致的状态来的。服务器上的一致性模块从客户端接收命令并将命令添加到其日志中。不同服务器上的一致性模块都与其他服务器的一致性模块通信以确保每一份日志最终都会包含顺序相同的相同请求，即使其中某些服务器宕机了也要满足这个要求。一旦命令被正确的复制了，每台服务器的状态机就可以用日志中的顺序执行这些命令了，然后讲结果返回给客户端。这个过程对客户端来说是透明的，从客户端的角度来看这些服务器就好像一台单独的高度可靠的状态机一样。

实践系统中的一致性算法具备下列典型的属性：

*  在任何[非拜占庭条件](https://lafengnan.gitbooks.io/blog/content/distributed-system/chapter2.html)下，包括网络延迟、分区，以及分组丢失、重复帧和乱序，都可以保证安全性（不返回错误的结果）。
* 只要多数服务器是可操作的，并且服务器之间以及服务器与客户端之间的通信是正常的情况下，算法是全功能的（可用的）。因此，一个典型的5节点集群在任意两台服务器失效的情况下是可以容错的。停机可以看成是服务器失效，重启后也可以从稳定的存储器中加载状态重新加入集群。
* 算法不依赖时间来保证日志的一致性：因为错误的时钟，以及最坏情况下极端的消息延迟都会导致可用性问题。
* 通常情况下，只要集群中的多数节点已经在一轮中向远程调用响应了结果，命令就可以被认为是完成了；少部分比较慢的节点不应该影响系统的整体性能。



## Raft算法

### 基本概念

#### 节点状态

Raft算法中定义了三种状态，分布式系统中的每个节点在任意一个给定的时间总是处于三种状态之一，而这些状态可以看作是一个节点的角色：

* Leader
* Follower
* Candidate

在正常操作中，集群中只有一个leader节点，其他节点都是follower。Follower节点都是*被动*节点，它们自己从不主动发送请求，相反，它们只是简单的响应leader和candidate节点的请求。Leader节点处理所有的客户端请求（如果一个客户端连接到的是一个follower节点，follower节点会把请求转发给leader节点）。Candidate节点用于选举新的leader节点，这个在后面解释。下图展示了不同状态之间的转换

​		![state-trans](/resources/state-trans.png)



#### 任期Terms

Raft算法将时间分隔为任意长度的不同任期term（如下图所示）。每个任期都用一个整数标识，每个任期都从一次*选举*开始，所谓*选举*就是一个或多个candidate节点试图成为新的leader节点的过程。如果某个节点成为leader节点，那么在接下来的周期中该节点就会一直扮演leader角色。有时候，选举可能会产生分裂的投票结果，这时候term就会以无leader为由而中止，随后很快一个新的任期伴随着新一轮选举而产生。Raft可以确保在一个给定任期中最多只有一个leader节点。

不同的服务器在不同的时间可能会观察到不同任期的转换，有些情况下一台服务器可能连一轮选举都观察不到，甚至是整个任期都看不到。实际上，任期可以看作是Raft算法中的逻辑时钟，它使得服务器可以感知过期的信息，比如已经失效的leader节点信息。每台服务器都会存储一个随时间单调递增当前的任期号。每当服务器互相通信时，都会交换各自的当前任期号。如果某台服务器的当前任期号比其它任启浩小，那么该服务器就会将其任期号更新为较大的号。如果一个candidate节点或者leader节点发现其自身的任期号已经过期了，它会立刻转换成follower状态。服务器会拒绝带有过期任期号的请求。

其实Terms就像美国总统的任期，就像现在的美国总统[特朗普](https://zh.wikipedia.org/wiki/%E5%94%90%E7%B4%8D%C2%B7%E5%B7%9D%E6%99%AE)是美国第45任总统，“45”就是特朗普作为美国总统的当前任期号。每一任美国总统的任期号都随着时间单调递增，任期号越大离当前时间点越近，任期号越小说明该总统越老旧。每一个总统任期都是由一轮总统选举产生的，每个任期内只有一个美国总统。这些和算法中的Terms概念完全相同，只是形式上稍微有点差异而已。

![terms](/resources/terms.png)

总结起来Terms具备如下特征：

* 每个Term由一轮选举产生
* 每个Term内至多存在一个leader节点
* Term号随时间单调递增
* 每个服务器本地维护一个当前任期号：currentTerm
* 若选举产生脑裂导致选举失败，则当前Term终止
* 节点与节点之间，以及节点与客户端之间通信时交换Term号，随时更新失效的Term号
* 服务器拒绝所有带有过期Term号的请求

#### RPC远程过程调用

Raft服务器通过远程过程调用（RPC）互相通信，基本的一致性算法只需**两种**类型的RPC：

* RequestVote RPC，由candidate节点在选举过程中发起
* AppendEntries RPC，由leader节点发起用于复制日志条目以及提供一种心跳形式

除了上述2种基本的RPC调用之外，还有一种RPC调用用于服务器之间传输快照。

如果在一个有限的时间内没有收到回执响应，服务器会发起RPC重试，而且为了提高性能服务器会并行发起RPC调用。



#### 过程

Raft算法将一致性问题分解成了三个相对独立的子问题分别处理，这三个问题分别是：

*  Leader节点选举：当一个已有的leader节点失效时，必须选择一个新的leader节点接替。
* 日志复制：Leader节点必须接受来自客户端的日志条目并在集群中复制，强迫其他服务器的日志接受自己的日志条目。
* 安全性：Raft算法安全性的关键属性是下面列出的状态机安全性属性。如果一旦任何服务器在其状态机中已经使用了一个特殊的日志条目，其他服务器就不能对相同的日志号使用不同的命令。
  + 选举安全性：在一个给定的任期内，最多只能选举出一个leader
  + Leader只追加日志：Leader节点不会覆写或者删除其日志中的条目，它只会追加新的条目
  + 日志匹配：如果两个日志包含一个具有相同的索引和任期的条目，那么在所有日志条目中这两条就是等价的
  + Leader完全性：如果一个日志条目在一个给定的任期内提交，那么该日志会在所有具有更高任期号leader节点的日志中出现
  + 状态机安全性：如果一台服务器已经在其状态机上使用了给定索引号的日志条目，对于该日志索引，其他服务器就不能使用不同的日志条目

Raft算法的简单流程是：

1. 选举一个唯一的leader节点，赋予其完全的责任来管理复制的日志
2. leader节点从客户端接收日志条目，并将条目复制到集群中的其他服务器上，通知这些服务器合适可以安全的将这些日志条目使用到他们的状态机中

### Leader选举

Raft算法通过一种心跳机制触发Leader选举。当节点启动时首先成为follower节点，只要节点能收到leader节点或者candidate节点的有效RPC调用就会一直处于follower状态。Leader节点周期性的向所有follower节点发送心跳数据（不包含日志条目的AppendEntries RPC调用）来维持其leader身份。如果Follower节点在一定时间（electiontimeout，选举超时）之内没能收到leader节点的信息，就认为系统中此时不存在有效的leader节点，随之发起一次选举选择一个新的leader节点。

为了发起一轮选举，follower节点首先将其currentTerm增1，并转换成candidate状态。然后给自己投票，同时并行向集群中的每个节点发起*RequestVotePRC*调用请求为自己投票。candidate节点在满足下列三种条件中的任何一个之前会一直处于candidate状态：

1. 该候选节点赢得选举
2. 另一个节点成为leader节点
3. 任期结束时仍没有节点胜出

如果某个candidate节点收到了全部集群中多数节点对相同任期（term）的投票，那么该candidate节点就赢得了大选。按照*先来先服务*的原则，在一个给定的周期内，最多只能投票给一个候选节点。*多数节点同意原则*确保在一个特定任期内最多只有一个候选节点可以赢得大选。一旦某个候选节点赢得了选举，就会成为leader节点，随后向集群中其他节点发送心跳消息宣布其身份以阻止新的选举。

在等待选票的过程中，候选节点可能会收到其他节点发来的宣称其Leader身份的*AppendEntriesRPC*消息，候选节点收到消息后有3种选择：

1. 如果PRC消息中携带的leader节点的任期号大于等于候选节点的currentTerm，该候选节点会承认新leader的合法地位并进入follower状态。
2. 反之，如果PRC消息中携带的term值比候选节点的currentTerm小，该候选节点必须拒绝该RPC消息并保持其候选节点状态。
3. 出现选票分裂。如果多个follower节点同时成为候选节点，选票可能会被分裂导致没有一个候选节点可以赢得多数节点的选票。这种情况下每个候选节点都会超时并通过增加其任期值开始一轮新的投票。（但是如果缺乏额外的措施，可能会陷入无穷的选票分离状态）

Raft使用随机的*选举超时时间*来保证选票分裂现象属于小概率事件，即使出现选也可以迅速解决。为了在首次投票时就杜绝选票分裂现象，选举超时值会从固定的时间间隔中随机选择（例如，150-300毫秒）。随机值在所有的节点中平均分布，因此在多数情况下只有一个节点会超时，超时节点会赢得选举并在其他节点超时之前发送心跳消息。这个机制也用于处理选票分裂的情况，每个候选节点在开始一轮选举时重启其随机的*选举超时时间*，并在发起新的选举之前等待该定时器超时。这也就避免了在新的选举任期中出现新的选票分裂情况。

Raft算法在设计时还考虑过采用排名系统来选举：每个candidate节点都分配一个唯一的排名，这个排名可以用于在互相竞争的candidate节点之间做出选择。如果一个candidate节点发现另一个candidate的排名更高，它就会退回follower状态，这样具有更高排名的candidate节点就可以很容易的赢得下一轮大选。不过作者发现这种方式会导致可用性问题（在具有较高排名值的服务器失效时，具有较低排名值的服务器可能需要等待超时从而再次成为candidate节点，但是如果太快，为了成为leader节点，它可以重置这个过程）。也就是说低排名的节点可能不停的重置选举过程而导致选举无法成功。

虽然作者对算法调整了很多次，但是每次调整都会出现新的错误情况，最终他们放弃了排名算法而改用随机定时器来实现选举。



### 日志复制

在Leader节点选举成功之后，就开始为客户端请求服务。每个客户端请求都包含一个由复制状态机执行的命令。Leader节点将命令作为新的条目追加于其日志尾部，然后向所有剩余节点并行发起*AppendEntries RPC*调用复制该条目。当条目安全的复制之后，leader节点将该条目应用到其自身的状态机中并向客户端返回执行结果。如果follower节点崩溃或者运行缓慢，或者出现网络丢包，leader节点会无限重试*AppendEntries PRC*调用，直至所有follower节点最终都存储了全部日志条目。

日志的组织形式如下图所示，每个日志条目都存储一个状态机命令，每条命令都带有leader节点收到命令时的任期号。日志条目中的任期号用于检测日志间的不一致性，并用以确保满足安全性条件。每条日志条目还有一个标识其在日志中位置的整数索引值。

![log](/resources/log-entry.png)

Leader节点决定何时可以安全地向状态机应用日志条目，被使用的日志条目称作*已提交的*条目。Raft算法保证已提交的日志条目是持久化的，且最终都会被所有可用的状态机执行。Leader节点在成功的向集群中的多数节点复制日志条目之后立即提交该条目，该操作同时也会提交包括旧leader节点创建的条目在内的所有之前未提交的条目。Leader节点会跟踪其所知的已提交条目的最大索引号，并在其发起的*AppendEntries* RPC调用消息中（包括心跳消息）包含该索引号以通知其他Follower节点，Follower节点在得知条目已经被提交之后即在其本地状态机中按顺序使用条目。

Raft算法的日志机制在高层次上维护了不同节点的日志之间的一致性，这种机制不仅简化了系统行为，使其更易预测，同时也是保证算法安全性的重要组件。Raft算法维护下列属性来构成下图所示的安全性：

- 如果不同日志中的两个条目具有相同的索引号与周期值，那么这两个条目中存储的是相同的命令
- 如果不同日志中的两个条目具有相同的索引号与周期值，那么日志中之前所有的条目都是相同的

第一个属性遵循leader节点在一个给定的周期内最多创建一个给定索引号的条目，并且日志中的每个条目都不会改变其位置的事实。

第二个属性由AppendEntries执行的简单的一致性检查来保证：

当Leader节点发送AppendEntriesRPC调用时，消息中会携带与新条目紧邻的旧条目在日志中的索引号与其term值，如果Follower节点在其本地日志中没有发现该索引号与term之前，该Follower节点会拒绝新的日志条目的写入。一致性检查的执行类似一个归纳步骤：

1. 最初的空白的日志状态机满足日志匹配属性
2. 每当扩展日志时，一致性检查都维持日志匹配属性
3. 每当AppendEntries调用返回成功时，leader节点便知道Follower节点的日志与其包含最新条目的本地日志相同



#### 异常情况

在正常情况下leader节点与Follower节点之间的的日志是保持一致的，*AppendEntries*的一致性检查并不会失败，不过万一Leader节点突然崩溃，使得其日志中的条目尚未被其他Follower节点完全同步，从而导致不同的节点之间的日志不同步，这种不同步可能会由于一些列的leader节点和Follower节点的崩溃而恶化。下图展示了Follower节点与一个新的leader节点之间的日志不同步现象，Follower节点可能缺失了某些在新的leader节点中存在的日志条目，也可能包含新的leader节点日志中不存在的日志条目，也可能同时存在这两种情况。

![log-exception](/resources/log-exception.png)

当最上面的leader节点恢复供电时，在follower节点中可能出现a-f的任意一种情况。图中每个格子代表一个日志条目，其中的数字是任期号。

* Follower节点可能丢失日志（a-b两种情况）
* Follower节点可能有额外的尚未提交的日志（c-d两种情况）
* Follower节点可能既有缺失的日志，也可能还有尚未提交的日志（e-f）

例如情况（f），如果该服务器在任期2时是一个leader节点，向其日志中添加了一些条目，但是在提交之前崩溃了然后迅速重启在任期3成为新的leader节点，又向日志中增加了一些条目，在任期2和任期3中的条目被提交之前再次崩溃，并持续几个任期。这时f就会出现。

出现这种日志不同步的情况时，Raft算法通过强制要求所有的Follower节点复制新leader节点的日志来解决节点之间的日志不同步问题。这也意味着，一旦新的leader节点被选举出来，Follower节点中与新leader节点日志冲突的条目都要被leader节点的日志条目覆盖。

同步的过程如下：

1. Leader节点在其本地日志中寻找与该Follower节点一致的最新的日志条目$$E_0$$
2. 删除Follower节点日志中$$E_0$$之后的所有条目
3. 向Follower节点发送其本地日志中$$E_0$$条目之后的所有条目

上述3步操作都是在*AppendEntries* RPC中执行的一致性检查的响应中发生。即如果*AppendEntries* RPC调用的一致性检查失败，Leader节点将执行上述三步来强制Follower节点与其保持同步。

Leader节点为每个Follower节点维护一个nextIndex值，该值通知Follower节点leader节点即将发送给每个Follower节点的日志条目的索引值。当一个Leader节点首次启动时，它将所有的nextIndex值都初始化为其本地日志最后一个条目的下一个值。如果某个Follower节点与该Leader节点的日志不同步，下一个AppendEntries RPC调用的一致性检查将会失败，Leader节点在收到该Follower节点拒绝添加条目的响应之后会递减其nextIndex值并重新发送AppendEntries RPC调用，直到与该Follower节点达成一致。此时该AppendEntries调用的一致性检查将会成功，从而删除Follower节点中与Leader节点不一致的所有条目，并添加Leader节点中存在而Follower节点中不存在的条目，此后该Follower节点的日志将与Leader节点保持同步了。

#### 同步优化 

Leader节点一旦发现某个Follower节点与其不同步，它将反复发送AppendEntries调用，直至Follower节点的一致性检查成功。这个过程可以通过优化来减少Leader节点发起AppendEntries RPC调用的次数、提高性能。

Follower节点在拒绝Leader节点的AppendEntries调用时可以在返回值中添加冲突条目的term值以及其本地日志中存储的该term的首个日志条目的索引号。通过这个信息，Leader节点可以通过忽略该term中的所有条目来递减其nextIndex值，这样每次重试AppendEntries调用时就不是一次一个条目，而是一次一个term，从而减少重试次数提高性能。



### 安全性



#### 选举限制

前面说的Leader节点选举与日志同步并不足以保证每个节点中的状态机都以相同的顺序执行所有的命令。例如，当leader节点提交多个日志条目的同时，某个Follower节点处于不可用状态，随后又被选举为新的Leader节点，并用新的条目代替原leader节点提交的条目，这样就导致不同的节点执行了不同的命令序列。可以通过增加选举限制来完善Raft算法的安全性。

限制：

> Raft要求参与选举的follower节点必须包括所有的日志条目才能赢得选举。

按照选举规则，一个候选节点要想赢得选举，它必须要赢得当前集群中的多数节点的选票才能成功。由于Leader节点只有在收到集群中的某个多数派节点的日志复制成功的响应之后才会提交日志条目，因此，当前系统中一定至少有一个节点保存了之前的Leader节点所有提交的所有日志条目。如果某个候选节点的日志条目与多数派节点中的任何日志都同样新，那么该候选节点将会持有所有的已提交的条目。基于这个原则，对RequestVote RPC调用引入如下限制：

* RPC消息中需要包含候选节点的日志信息，
* 并且如果某个投票节点的日志比该候选节点的日志新，该投票节点将会拒绝为该候选节点投票。Raft算法通过比较不同日志中最新条目的索引号与term值来判断哪个日志更新：
  1. 如果term不同，则拥有更大term值的日志较新
  2. 如果term相同，则拥有更大索引值的日志较新



#### 提交前任的日志条目

根据Raft算法的日志提交策略，当集群中的多数节点成功存储了日志条目之后，leader节点就会提交当前任期的日志。如果Leader节点在提交条目之前崩溃，新的Leader节点将会试着完成该条目的复制。但是新的Leader节点并不能立刻判断被多数节点成功存储的旧周期日志条目是否已经提交，如下图所示的情形描述了这种特殊状况（任期为2的条目是否已经成功提交）。 

1. S1是Leader节点，并且已经向部分节点成功的复制了索引号为2的条目
2. S1突然崩溃，S5在周期3从S3，S4和其自身赢得投票成为新的Leader节点，并在日志的索引号2处接受了一个不同的条目
3. S5突然崩溃，同时S1重新启动并赢得选举（周期4），并继续复制日志条目。此时，周期2的条目已经成功复制到了集群中的多数派节点，但是尚未提交。
4. 如果S1突然崩溃，S5可以通过或得S2，S3，S4的选票赢得选举成为新的Leader节点，然后用其自身的日志条目（周期3）来覆盖其他节点的日志条目（删除S1的条目4，用条目3覆盖其他节点的条目2）
5. 但是，如果S1在崩溃之前已经将其当期周期（4）的条目都成功的复制到了集群中的多数派节点，节点S5将无法赢得选举（因为其周期3比集群中剩余可用节点S2，S3，S4，S5中的S2，S3节点的周期值小，将无法获得S2和S3的选票，S5最多只能获得S4和自身的选票而无法成为多数派节点），只有S2或者S3节点可以成为新的Leader节点，从而继续完成日志拷贝从而将周期4之前的所有条目都提交到集群中的全部节点。

![commit-previous](/resources/commit-previous.png)

Raft算法采用如下策略来解决上述问题：

Raft通过统计副本数而不提交旧任期的日志条目，通过统计副本数只提交Leader节点当前周期的日志条目。一旦当前任期的日志条目已经提交，该条目之前的所有条目都会因为需遵循日志匹配属性而间接提交。



### Follower节点与候选节点崩溃

Follower节点与候选节点的崩溃处理不像leader节点，处理比较简单。如果一个follower节点或者candidate节点崩溃了，那么后续发送到该节点的*RequestVote* 和*AppendEntries* RPC调用就会失败。Raft通过无限重试来处理这些失败，如果崩溃的节点重启了，那么RPC调用会成功的完成。如果一个服务器在完成一个RPC调用之后但是在返回响应之前崩溃，那么重启之后它会再次收到相同的RPC调用。Raft RPC调用都是幂等的，所以重复执行相同的RPC并不会有什么不妥。

### 定时与可用性

Raft算法需要满足一个需求，那就是安全性不能依赖于定时系统：不能因为系统中某些事件发生的速度快于或者慢于预期速度而导致出现不正确的结果。 定时机制在Raft算法的选举过程中最关键，只要系统能够满足下面的计时需求，Raft算法就能够选举和维护一个稳定不变的Leader节点：

$$ broadcastTime \le electionTimeout \le MTBF$$

其中，

* broadcastTime是节点向向其他节点并行发送RPC并接受响应的平均时间，通常为0.5ms-20ms

- electionTimeout是前文描述的选举超时时长，通常为10ms-500ms
- MTBF是节点的平均无故障时间

## 集群成员变更

为了使系统保持可用性，集群的成员发生变更导致的集群配置的变化必须要避免引入人工重启集群这样的操作。为了保证集群配置的变更能够安全生效，集群的配置管理需要引入两阶段机制。

* 阶段1： 联合一致性(jointconsensus)
* 阶段2： 传输新配置

联合一致性同时包含新老配置：

- 日志条目以两种配置复制到所有节点
- 不同配置中的节点都可能成为Leader节点提供服务
- 不同的配置下选举都需要互相独立的多数派节点达成一致

联合一致性使的节点不同时刻在不同的配置中无需考虑安全性，同时也保证集群可以持续提供可用性。集群的配置在日志中以特殊条目存储通信，下图模拟了集群配置的变更过程。

![membership-change](/resources/membership-change.png)

## 日志压缩

在正常处理更多的客户端请求时Raft的日志会增长，但是在实际系统中，日志不能无限制增长。因为随着日志越来越长，就会占用更多的空间，也会花费更长的时间去重放。如果不采取措施丢弃日志中的累积的过时信息，最终肯定会导致可用性问题。

快照是最简单的压缩方式。在快照中，当前系统的完整状态会被写入到一个存储在稳定设备上的*快照*中，然后所有截止到快照点的所有日志全部丢弃。快照技术在Chubby和Zookeeper中都有使用。

用递增的方式实现压缩，例如*日志清除*和*日志结构合并树*都是可行的方法。这些方法每次在一段数据上操作，因此压缩负载随着时间会均匀分布。首先选择一段已经累积了很多已经被删除和覆写了的对象的数据区域，然后将该区域的活跃数据更加紧凑的重写一遍，随后释放这段区域。这种方式与快照技术相比明显需要引入额外的机制和复杂度，而快照技术只需简单的比较完整的数据集。

*日志清除*可能需要修改Raft算法，状态机可以使用和快照技术相同的接口实现**LSM**树。

下图展示了Raft中的快照技术的基本思想。每台服务器都独立的执行快照，只覆盖其自身日志中已经提交的条目。大部分工作都由状态机向快照中写入其当前状态组成。Raft同时会在快照中包含少量的元数据：快照替换的日志中最后一条日志的索引*last included index*和该条目的任期号*last  included term*。这些元数据保存下来用于支持**AppendEntries**紧随快照的第一条日志的一致性检查，因为这条日志需要一个前导日志索引号与任期号。为了开启集群成员变更功能，快照中还包括与日志中最后一条日志的索引相同的最新的配置信息。一旦一个台服务器完成了一份快照写工作，可能会删除直到包含的最后一条日志条目截止的所有日志条目。

虽然服务器通常情况下独立的执行快照，但是leader节点偶尔也必须向落后的follower节点发送快照。这在leader节点已经丢弃了下一条需要发送给follower节点的日志时会发生。幸运的是，这种情况并不是一个正常的操作：follower节点通常都与leader节点拥有的条目保持一致。虽然如此，一个异常缓慢的follower节点或者一个新加入集群的服务器可能并不会如此。让这样的follower节点跟上leader节点的方式是在整个网络中发送一份快照。

Leader节点使用一个新的名为*InstallSnapshot*的RPC向落后太多的follower节点发送快照。参见下图，当follower节点收到*InstallSnapshot*RPC调用时，它必须决定如何处理其已有的日志条目。通常情况下，快照会包含接收者尚未收到的新信息，这些条目会被快照完全取代，也可能存在于快照中的条目冲突的尚未提交的条目。相反，如果follower节点收到的快照描述了一个follow而节点已经包含的日志前缀(由于重传或者错误导致)，那么该快照覆盖的日志条目将会被删除，但是快照后面的日志仍然有效并且会被保留。

快照技术和Raft的强leader原则是分离的，因为follower节点可以无需知道leader节点就拍摄快照。但是我们认为这样分离是合理的。在leader节点帮助我们在达成一致性的时候避免了冲突决议时，当拍摄快照时一致性已经得到满足，因此，并不存在决议冲突。数据仍然是从leader节点流向follower节点，只是follower节点现在不能重现组织它们的数据而已。

其实，按照Raft论文的描述，作者也曾经考虑过另外一种基于leader，只由leader节点拍摄快照，然后将快照发送给follower节点的方法。但是这种方法有两个缺点：

1. 首先向每个follower节点发送快照会浪费网络带宽，也会拉低快照过程
2. Leader节点的实现会更加复杂，例如，leader节点在向follower节点复制日志时还需要向follower节点并行发送快照

影响快照性能的因素有很多：

1. 首先，服务器必须决定何时拍摄快照，如果过于频繁则会浪费磁盘带宽和能量；反之，则存在浪费存储容量的风险，也增加了重启过程中日志重放所需的时间。一个简单的策略是当日志达到一个固定的大小时拍摄快照。
2. 其次，写一份快照会耗费一定量的时间，但是我们又不希望这个操作导致正常的操作延迟。采用的技术是写时复制copy-on-write技术来规避，这样新的更新可以在不影响快照的情况下写入。