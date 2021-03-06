# Redis分布式锁（译）

原文参考：[http://redis.io/topics/distlock](http://redis.io/topics/distlock) 



在很多不同的进程必须以互斥方式操作共享资源的环境中，分布式锁是一个非常有用的原语。

已经有很多库和博客描述了如何利用Redis来实现一个DLM（分布式锁管理器），但是不同的库都使用了不同的方式，还有很多只使用了与稍微复杂一点的设计相比要简单，但是缺乏安全性保证的方式。本文试图提供一种利用Redis实现分布式锁更加权威的算法。我们推荐一个名为Redlock的算法，它比单实例的简单实现更加安全。我们希望社区可以分析这个算法，提供反馈并且使用这个算法来实现或者提出更好的替代实现。

 ## 已有的实现 



- [Redlock-rb](https://github.com/antirez/redlock-rb) (Ruby implementation).     There is also a [fork of Redlock-rb](https://github.com/leandromoreira/redlock-rb) that     adds a gem for easy distribution and perhaps more.
- [Redlock-py](https://github.com/SPSCommerce/redlock-py) (Python implementation).
- [Redlock-php](https://github.com/ronnylt/redlock-php) (PHP     implementation).
- [PHPRedisMutex](https://github.com/malkusch/lock#phpredismutex) (further     PHP implementation)
- [cheprasov/php-redis-lock](https://github.com/cheprasov/php-redis-lock) (PHP     library for locks)
- [Redsync.go](https://github.com/hjr265/redsync.go) (Go implementation).
- [Redisson](https://github.com/mrniko/redisson) (Java implementation).
- [Redis::DistLock](https://github.com/sbertrang/redis-distlock) (Perl     implementation).
- [Redlock-cpp](https://github.com/jacket-code/redlock-cpp) (C++     implementation).
- [Redlock-cs](https://github.com/kidfashion/redlock-cs) (C#/.NET     implementation).
- [RedLock.net](https://github.com/samcook/RedLock.net) (C#/.NET     implementation). Includes async and lock extension support.
- [ScarletLock](https://github.com/psibernetic/scarletlock) (C#     .NET implementation with configurable datastore)
- [node-redlock](https://github.com/mike-marcacci/node-redlock) (NodeJS     implementation). Includes support for lock extension.

 

 ## 安全性与生命期保证



 保证高效使用分布式锁至少需要三个属性：

1. 安全属性：互斥。在任意给定的时间，只有一个客户端可以持有锁。
2. 生命期属性A：无死锁的。即使锁定了某个资源的客户端发生崩溃或者出现网络分区，其他客户端最终也可以获得一把锁。
3. 生命期属性B：容错的。只要Redis节点中的多数派是可用的，客户端就可以获得/释放锁。

 

## 基于Failover的实现的不足



利用Redis锁定一个资源最简单的方法是在Redis实例中创建一个键。通常情况下，创建这个键时会利用Redis的过期功能设置其有限的存活期，因此该键最终是可以被释放的（生命期属性A）。当客户端需要释放资源时只要删除这个键就可以了。

表面上看这种简单的实现可以很好的工作，但是却存在一个问题：Redis成为架构中的单个失效点，如果这台master节点宕机怎么办？通过增加slave节点并没有什么卵用。

通过增加slave节点并不能解决互斥问题，因为Redis的副本是异步拷贝的。在这个模型中存在明显的竞争条件：

1. 客户端A在master节点中创建一个key获取了锁
2. master节点在被锁定的key被复制到slave节点之前崩溃了
3. slave节点被提升为master节点
4. 客户端B在新的master节点上创建key来获取A已经锁定的资源，这时B的操作是可以成功的，从而违反了锁的互斥性。

 

## 单实例的正确实现

在克服上文描述的单实例限制之前，我们先看看如何正确的实现这种简单的案例。因为单实例方案在很多能接受竞争条件的应用中还是非常有用的，而且锁定单个实例也是分布式算法的基础。获取锁的方法：

```she
SET resource_name my_random_value NX PX 30000
```

这条命令仅在key不存在的情况下（NX选项）可以设置过期值为30000毫秒（PX选项）的key。key的值被设置为一个自定义的随机值，这个值在所有的客户端和所有的加锁请求中必须是唯一的。使用随机值的目的是为了可以通过脚本安全的释放锁：“只有key存在并且存储在key中的值是我期望的值时才删除这个key”。可以通过下面的Lua脚本实现：

 ```lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end 
 ```

这种实现可以避免删除不是当前客户端创建的锁。例如，一个客户端可能持有了一把锁，并且该客户端的后续操作被阻塞了很长时间（超过了锁的生命期）之后才完成，然后这个客户端请求删除这把锁。如果key的值不唯一就会导致该客户端会删除另外一个客户端新建的锁。

随机字符串可以用下列方式实现：

- /dev/urandom
- 微妙精度的Unix时间戳

key的生命期称为"锁有效时间"，这个时间既是锁的自动释放时间，同时也是获取到锁的客户端在另一个客户端可以获取该key上的锁之前可用于执行操作的时间。这个设计仅仅是在获得锁之后的一个时间窗，并不违反锁的互斥性。 

这种设计对于由单个可用的redis实例构成的非分布式系统来说是足够安全的，对于分布式系统则需要扩展这个实现。 



## Redlock算法

分布式版本的算法假设有N个Redis主节点，这些节点之间是互相独立的，因此不使用副本或者其他隐式的协调系统。前面已经描述了在单实例环境下如何安全的获取和释放锁，我们也理所当然的认为(takefor granted)Redlock算法也使用这个方法在单实例中获取和释放锁。在我们的例子中N = 5， 因此需要在不同的计算机或者虚拟机中运行5台redis主节点保证这些节点进入失败状态的原因是彼此独立的。

客户端需要执行下列步骤来获取一把锁资源：

1. 以毫秒格式获取当前的时间
2. 用相同的key和随机值依次在N个实例中获取锁。在第二步中，在每个实例中设置锁时，客户端使用比锁的自动释放时间值小的超时时间来获取锁。例如，如果自动释放时间是10秒，那么超时时间可以是5~50毫秒。这可以防止客户端试图从一个已经宕机的节点获取锁时被阻塞很长时间：如果某个redis实例已经不用，那么客户端必须尽快与下一个实例通信。
3. 客户端通过从当前时间戳减去第一步中获取的时间戳来计算需获取锁花费了多少时间。当且仅当客户端可以从集群中的多数派（至少3个实例）获取到锁，并且获取锁花费的总时间比“锁有效时间”小，那么就认为锁已经成功获取。
4. 如果锁被成功获取，这把锁的实际有效时间等于初始有效时间减去第三步中得到的获取锁所花费的时间。
5. 如果因为某些原因（不能锁定$$N/2 + 1$$ 或者有效时间为负数），客户端的锁操作失败，它将试图解锁所有的实例（包括该客户端不能锁定的实例）。

### 算法是否异步

算法本身基于这样一个假设：进程之间不存在同步的时钟，每个进程的本地时间都以相同的速率精确流动，并且与锁的自动释放时间相比误差也非常小。这个假设与真实世界的计算机非常类似：每个计算机都有一个本地时钟，并且我们通常可以依靠不同计算机来获取很小的时钟漂移。

此刻需要更好的表述互斥规则：只有持有锁的客户端在"锁有效时间"（第三步获得的时间）减去某个时间（只有几毫秒，以消弭不同进程之间的时钟漂移）之内终止它的工作，互斥性才可以得到保证。

需要时钟漂移的相似系统可以参考这篇有趣的论文：[Leases: an efficient fault-tolerant mechanism for distributedfile cache consistency](http://dl.acm.org/citation.cfm?id=74870).

### 出错重试

如果客户端无法获取锁，它必须在一个随机的延迟之后重试，以解除多个同时对相同的资源申请锁（这可能导致出现没有任何人可以赢的脑裂情况）的客户端之间的同步。当然，客户端在多数派中获取锁的速度越快，脑裂条件的窗口越小（重试的也越少），因此理想情况下客户端必须用多路复用同时向N个节点发送SET命令。 

客户端从多数派获取锁的操作失败后尽快释放持有的锁非常重要。因为这样可以让那些已经被持有的锁无需等待锁过期就再次变成可用状态（当然，如果发生了网络分区并且客户端无法与该redis实例通信，就只有等待锁过期了）。 

### 锁释放

 锁的释放很简单，不论客户端是否已经成功的获取到了锁，它只要释放所有实例上的锁即可。

### 安全性

假设一个客户端可以获取多数派的锁。所有的实例将会包含一个同名的key。然而，不同实例上的key是在不同时间被锁定的，因此这些key也会在不同的时间过期。但是在最糟糕的情况下，如果第一个key在T1时间被设置（在于第一个服务器通信之前取样），最后一个key在T2时间被设置（从最后一个服务器获得响应的时间），可以确认过期集合中的第一个key存在的最小时长为：$$MIN\_VALIDITY = TTL - (T2 - T1)- CLOCK\_DRIFT$$。其他所有的key都会在随后过期，因此我们确信key至少可以存在这么长时间。

在多数key都被设置的时间内，另一个客户端将无法获取锁。因为如果已经有$$N/2 + 1$$个key存在了，那么就有$$N/2 + 1$$个SETNX操作是无法成功的。因此，如果一把锁已经被某个客户端持有，同一时刻这把锁是无法再次被获取的。

多个客户端同时获取同一把锁是不能同时都成功的。

如果一个客户端用一个近似或者大于最大有效时间（SET命令使用的TTL时间）的时长锁定了多数派实例，客户端将认为这把锁是无效的然后解锁实例。因此我们只需考虑在最大有效期之内可以成功锁定多数派实例的情况。这种情况与前面描述的一致，在MIN_VALIDITY内，没有客户端可以再次获取到相同的锁。因此只有在锁定多数派节点的时长大于TTL时，多个客户端才能够在同一时间（第二步结束时的时间）锁定N/2+ 1个实例。然而这个时间超过了TTL时间，从而导致锁失效。

### 活性参数

系统活性基于三个主要功能：

1. 锁的自动释放（通过key过期）：最终所有key都可以被再次锁定。
2. 基于未成功获取锁、或者获取锁之后工作完成时，客户端都会协助删除所有锁这样的事实，我们无需强制等待锁过期来再次获取锁资源。
3. 为了最大限度的使得在资源竞争时脑裂条件不成立，当客户端需要重试时，会等待一段比获取多数派节点锁所需的时间少一点的时间。

虽然如此，在网络分区时我们还是需要等待TTL时长。如果持续存在网络分区，可能要需要无限等待了。在客户端获取一把锁并且在删除锁之前被网络分区隔离时，这种情况每次都会出现。

基本上，如果持续存在网络分区，系统可能在长时间内变成不可用状态。

### 性能、灾难恢复和同步刷新

很多使用用Redis作为锁服务器的用户对锁的获取/释放延迟以及每秒钟对锁的获取/释放次数的性能要求很高。为了满足这个需求，采用多路复用（或者“穷人的多路复用”，即认为客户端与所有节点之间的RTT是相似的，采用非阻塞套接字发送命令，随后一起读取所有的命令）降低与N个redis节点通信的延迟。

如果想实现一个可以灾难恢复的系统，还需要考虑持久化的问题。

从基础上来看一下这个问题，我们假设redis被配置为不提供任何持久化能力。一个客户端从5个redis实例中的3个获取了锁。此前可以被客户端获取到锁的实例被重启，此刻会再次存在3个实例可以用于锁定相同的资源，另一个客户端就可以再次锁定该资源，违反了锁的互斥性。

如果打开AOF持久化，会有所改观。例如，我们可以通过发送**SHUTDOWN**命令并重启机器升级一台服务器。因为redis的键过期语义是一致的，因此即使服务器处于关闭状态时间也仍然在流逝，所有的需求都可以满足。只要是安全关闭的，一切都将很美好。如果发生停电，默认配置下redis每秒钟向磁盘刷新一次数据，这可能导致服务器重启之后key出现丢失的情况。理论上，如果要在各种重启状况下都能保证锁的安全性，必须在持久化配置中设置sync=always。这将会导致系统的性能下降到与传统的以安全的方式实现分布式锁的CP系统的级别一致。 

只要redis实例在崩溃重启之后不参与任何现有的活跃锁，那么在实例重启时处于活跃状态的锁集合就都是之前被这个重新加入系统的实例以外的服务器存储的，算法的安全性也就可以得到保证。为了保证这个目标，我们只需使实例在崩溃之后至少比TTL时间长一点的时间段之内是不可用，这段时间可以让实例崩溃时已经存在的锁变成不可用并自动释放。使用延迟重启可以使系统无需任何持久化就可以达到所需的安全性。不过这个可能会导致系统出现可用性问题。例如，如果多数派节点崩溃，整个系统将会在TTL时间内变得全局（此处的全局表示在这段时间内没有资源可以被锁定）不可用。

 ### 使算法更可靠：锁延期

如果客户端的工作由一些小的步骤组成，可以使用小一些的默认“锁有效时间”将算法扩展一个锁延期机制。如果在计算过程中“锁有效时间”是一个比较小的值，若key存在并且key的值与客户端获取锁时分配的随机值相同，可以通过向所有的实例节点发送一个Lua脚本来扩大key的TTL。

如果客户端可以在多数派节点中延长锁的时间，它只需考虑有效期内重新获取锁的问题。但是技术上这个问题不会改变算法本身，因此必须限制锁的最大重获取次数，否则就会违反生命期属性。



### 需要帮助？

如果你正从事分布式系统的工作，你的意见和分析将会非常有益。用其他语言实现参考实现也非常棒。

非常感谢！

### Redlock分析

1. Martin Kleppmann [在这里分析了Redlock](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)。我并不同意这个分析，并且在[这里](http://antirez.com/news/101)对这个分析做了回复。