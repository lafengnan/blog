# 以太坊黄皮书笔记

## 区块链

*Any technology worth adopting is often adopted first by criminals.*

​                                                                                        \- [Kathryn Haun ](https://www.youtube.com/watch?v=507wn9VcSAE)



## The Golden Eight: Blockchain Transormations Of Financial Services

1. Authenticate & Attest to Value
2. Transfer Value
3. Store Value
4. Lend Value
5. Exchange Value
6. Fund & Invest
7. Insure Value & Manage Risk
8. Account for & Audit Value



## 智能合约

**Smart contracts** are computer protocols that facilitate, verify, or enforce the negotiation or performance of a *contract*, or make a contractual clause unnecessary. *Smart contracts* often emulate the logic of contractual clauses. Proponents of smart contracts claim that many kinds contractual caluses may thus be made partially for fully self-executing, self-enforcing, or both. Smart contracts aim to provide security superior to tradtional contract law and to reduce other *transaction costs* associated with contracting.[^1]

[^ 1 ][Smart contract](https://en.wikipedia.org/wiki/Smart_contract)



**Smart contracts** help you exchange money, property, shares, or anything of value in a transparent, conflict-free way, while avoiding the services of middleman.[^2]

[^ 2 ][Smart Contracts: The Blockchain Technology That Will Replace Lawyers](https://blockgeeks.com/guides/smart-contracts/)



*Cotcha:* 其实智能合约就像一个触发器，在预设条件成立时自动执行从而免除了中间方的参与，也使单方违约很难成立。目前比特币支持的脚本语言不是*图灵完备*的，因而只能执行比较简单的功能，*以太坊* 则可以实现想要的全部功能。



## 侧链

 **Sidechain(侧链)**是用来连接不同区块链网络的技术。



**Insight of sidechain:**

> What if you could send Bitcoins not only to individuals, addresses and centralized services but to **other blockchains**?

Imagine there is a Bitcoin-like system out there that you’d like to use. Perhaps it’s litecoin or ethereum or perhaps it’s something brand new.   Maybe it has a faster block confirmation interval and a richer scripting language. It doesn’t matter.   The point is: you’d like to use it but would rather not have to go through the risk and effort of buying the native tokens for that platform. You have Bitcoins already. Why can’t you use them?

The sidechain ideas is this:

+ 将你持有的一定数量的比特币发送到一个特殊的地址上，这个地址经过特别设计，使得其拥有的比特币不受任何人控制，包括你自己和任何其他人（也就是说这些比特币变得不可移动了，直到有人证明这些比特币不在其他地方使用了才能对其解锁）
+ 一旦这些被冻结的比特币得到足够的确认，你再向你准备使用的区块链发送一则消息，消息内容包括这些比特币已经发送到了比特币网络中的某个指定的地址，并且这些比特币现在处于不可用状态，操作也是你完成的。
+ 如果第二个区块链同意成为比特币网络的侧链，它就会创建与比特币数量相同的令牌由你控制
+ 如果你从比特币网络向第二个区块链转移了比特币，并不是说你创建或者销毁了比特币，你只是在比特币网络中冻结了这些比特币，将他们“转移”到了第二个区块链网络中
+ 现在你就可以在第二个区块链网络中依照他们的规则进行交易
+ 第二个区块链上面的块创建速度可能更快，脚本语言可能是“图灵完备”的，等等。第二个区块链只关心一个规则：侧链已经同意只要你证明已经将一定数量的比特币移出了比特币网络，它就会在侧链中创建相同数量的令牌。
+ 最后，相反的过程也很聪明，是一个对称的过程。在任何时间，任意一个在侧链中持有这些转移过来的令牌的人都可以通过将侧链中这些令牌冻结，从而将他们回送到比特币网络中。这样这些比特币将会从侧链中消失，而在比特币网络中重新出现，其主人正是在侧链中最后持有那些令牌的人。




## 以太坊黄皮书



### 区块链

整体上*以太坊*可以看作是一个基于*交易*的状态机：从创世状态开始*不断*执行交易，将状态转换成某个*最终状态*。这个*最终状态*在以太坊世界作为一个权威的”版本"而被接受。状态可以包含这些信息：

* 账户余额(Balances)
* 账户信誉(Reputations)
* 托管（Trust arrangements)
* 与物理世界信息的数据关联

简单来说，就是当前可以用计算机表示的任何数据都是可以接受的。

**交易**是两个状态之间的一个**有效**的弧，这里强调**有效**很重要，因为无效的状态变化远比有效的状态变化多的多。无效的状态变化可能是：单方减少账户余额却不存在对等的余额增加。

**有效**的状态变化来自交易的触发，正式的形式：
$$
（1)\qquad \sigma_{t+1} \equiv \Upsilon(\sigma, T)
$$
$\Upsilon$ 是以太坊的状态转移函数，在以太坊中$\Upsilon$与$\sigma$一起配合比任何已有的对比系统都强大。

$\Upsilon$允许参数执行任意计算，同时$\sigma$则允许参数在交易之间存储任意的状态。

交易在验证之后会进入**区块**，不同的**区块**用加密的哈希值作为引用一起串成链条。区块以日志的形式将一系列交易、前一个区块的信息，以及最终状态（虽然并不存储最终状态本身—体积太大了）的标识符一起记录下来。对交易序列区块会为挖矿节点标识奖励信息。奖励以一个状态转移函数的形式向指定的账号中增加*价值*。 

挖矿是一个通过工作量证明机制来竞争向区块中写入交易序列权力的机制，其正式形式为：
$$
(2)\qquad \qquad \quad \quad  \sigma_{t+1} \equiv \Pi(\sigma_t, B)
$$

$$
(3) \qquad \quad  B \equiv (...,(T_{0},T_{1},...))
$$

$$
(4) \ \Pi(\sigma, B) \equiv \Omega(B, \Upsilon(\Upsilon(\sigma,T_0), T_1)...)
$$

其中，

$\Omega$是区块最终的状态转移函数（向指定节点提供奖励的函数）；

B是当前区块，其中包括存在于其他模块中的一系列交易；

$\Pi$是区块级的状态转移函数

这些是区块链模式的基础，不仅是以太坊的基石，也是迄今为止其他所有去中心化的一致性交易系统的基石。

以太坊区块链基本结构图：

![区块链基本结构](/resources/blockchain/pow_chain.png)



### 价值

以太坊的货币称为以太币，**ETH**，有时也以古英语字符$D$表示。以太币的最小面值是**Wei**， 1 ETH等于$10^{18}$ Wei， 除了Wei还有其他的面值：

| 乘数        | 名称     |
| --------- | ------ |
| $10^0$    | Wei    |
| $10^{12}$ | Szabo  |
| $10^{15}$ | Finney |
| $10^{18}$ | Ether  |

当前以太坊中任何涉及*价值*的上下文中，包括余额、现金或者付款都必须以*Wei*来计算。

### 约定

2个顶级结构用粗体希腊字母表示：全局状态**$\sigma$**和虚拟机状态**$\mu$**。

操作高级结构的函数用大写的希腊字母表示，例如以太坊状态转移函数$\Upsilon$.

其他大部分函数用大写字母表示，例如通用成本函数$C$。这些函数可能用下标表示特定的变体函数，比如$SSTORE$操作的成本函数$C_{SSTORE}$。对于特殊都和和外部定义的函数，以打印排版格式书写，比如Keccak-256散列函数（作为胜利者战胜了SHA-3）表示为$KEC$（常常作为Keccak的简写使用），当然，$KEC512$就表示Keccak 512 散列。

元组用一个大写字母表示，比如$T$表示一笔以太坊交易。这个标记可能会根据需要以下标去表示一个单独的分量，比如$T_n$表示当前这笔交易的随机数。下标的形式用于表示其类型，比如大写字母的下标表示特定下标分量的元组。

标量和固定长度的字节序列（或者同义词数组）用给一个普通的小写字母表示，比如$n$用于表示一个交易随机数（nonce）。那些有特殊含义的值用希腊字母表示，比如$\delta$表示给定操作在栈上需要的元素数量。

任意长度的序列用一个粗体小写字母表示，比如**o**表示消息调用输出数据的字节序列。对于特定的重要值，可能会使用一个粗体大写字母。

我们假定标量都是正整数，因此他们都属于集合$\mathbb{P}$。所有的字节序列集合$\mathbb{B}$在下面定义。如果这样的序列集合被限定了长度，则用下标表示。因此，所有长度为32的字节序列被命名为$\mathbb{B}_{32}$，所有小于$2^{256}$的正整数命名为$\mathbb{P}_{256}$。正式定义在下文给出。

中括号用于索引或者引用独立的分量或者一串序列的子串，比如$\mu_s[0]$表示虚拟机栈中的第一个元素。对于子序列，点号用于指定范围，包括左右限制元素，比如$\mu_m[0..31]$表示虚拟机内存中的钱32个元素。

对于表示一个账户序列的全局状态$\sigma$，中括号用于引用一个单独的账户。

对于已存在的值的变体，遵循定义的给定范围的规则，加入用一个占位符$\square$表示未修改的输入值，那么被修改过的可用值就用$\square'$表示，中间值用$\square^*, \square^{**}$表示。对于特别情况，为了最大化可读性，只有在含义不清楚的时候，采用阿尔法数字作为中间值的下标，尤其是那些特殊的情况。

对于已有的函数使用，给定函数$f$，用$f^*$表示一个相似的，元素级的函数映射在序列之间代替。

贯穿全文定义了很多有用的函数，其中最常见的是$\mathcal{l}$，用于计算给定序列中的最后一个元素：
$$
(5)\qquad \mathcal{l}(x) \equiv x[\parallel x \parallel - 1]
$$

### 术语

**External Actor** 外部操作者：以太坊之外可以与以太坊交节点互的人或者其他实体。可以通过签名的交易存款，检查区块链及其关联的状态。有一个（或者多个）固有账号。

**Address** 地址：一个用于标识账户的160比特编码。

**Account** 账户：账户中有一个固定的月和作为以太坊状态部分维护的交易计数器。账户中还有一些（可能为空）EVM代码和一个（可能为空）与代码关联的存储状态。虽然账号之间是类似的，但是还是有必要区分两种不同的账号： 带有空EVM代码（因此账户的余额是受控的，如果可能都是由外部实体控制的）和带有非空EVM代码的账号（因此账户代表一个自主对象）。每个账号都有一个单独的地址标识。

**Transaction** 交易：由一个外部操作者签名的数据。标识一个消息或者一个新的自主对象。交易被记录在区块链中的一个区块中。

**Autonomous Object** 自主对象：仅在以太坊的抽象状态中存在。有一个固定的地址和关联账户；账户没有非空的祥光的EVM代码。只作为一个账户的存储状态。

**Storage State** 存储状态：给定账户的关联EVM代码运行时维护的特殊信息。

**Message** 消息：通过一个自主对象的确定性操作或者交易的安全加密签名，在账户之间传递的数据（以字节集合）和价值（以太币），

**Message call** 消息调用：从一个账户到另一个账户之间传递消息的动作。如果目标账户与非空EVM代码关联，则虚拟机将以所说的对象的状态启动，并在骑上操作消息。如果消息发送者是一个自主对象，那么调用将传递VM操作返回的任意数据。

**Gas** 汽油：网络开销的基本单位。只能用可以与Gas自由兑换的以太币（截止PoC-4）支付。Gas在内部以太坊计算引擎之外并不存在，其价格由交易设定，而且矿工可以忽略Gas价格过低的交易。

**Contract** 合约：用于表示一些可能与一个账户或者一个自主对象关联的EVM代码的非正式说法。

**Object** 对象：自住对象的同义词。

**App** 应用：一个终端用户可见的位于以太坊浏览器上的应用。

**Ethereum Browser** 以太坊浏览器：（又称以太坊参考客户端）是一个跨平台的GUI，接口与简单的浏览器（Chrome

）很相似，可以运行那些后端是存储的基于鸡腿饭协议的沙盒应用。

**Ethereum Virtual Machine** 以太坊虚拟机：（又称EVM）对于一个账户的EVM代码，以太坊虚拟机组成了执行模型的重要部分。

**Ethereum Runtime Environment** 以太坊运行时环境：（又称ERE)在EVM中供一个自主对象执行的环境。包括EVM以及对特定的I/O指令，包括CALL和CREATE，EVM依赖的全局状态结构。

**EVM Code** 以太坊虚拟机代码：EVM可以原生执行的字节码。用于正式表示发送给一个账户的消息的含义与衍生信息。

**EVM Assembly** 以太坊虚拟机汇编：人可读的EVM代码形式。

**LLL** 类Lisp 的底层语言，一种人类可写的语言，用于授权简单的合约，和用于编译的通用高低层语言工具包。



### 递归表达式

这里定义了一系列用于编码结构化数据（字节数组）的方法。可用的结构$\mathbb{T}$定义如下：
$$
(162)\qquad \qquad \qquad \qquad \qquad \qquad \qquad \quad \mathbb{T} \equiv \mathbb{L} \cup \mathbb{B} \\
(163)\ \mathbb{L} \equiv \{t: t = (t[0], t[1], ...) \wedge \forall_{n < \parallel t \parallel } t[n] \in \mathbb{T}\} \\
(164)\ \mathbb{B} \equiv \{b: b = (b[0],b[1],...) \wedge \forall_{n < \parallel b \parallel} b[n] \in \mathbb{O}\}
$$
$\mathbb{O}$是字节集合，因此$\mathbb{B}$就是所有字节序列的集合（字节数组，如果看作树就是一片叶子）。$\mathbb{L}$是所有不是只有一片叶子的（树的一个的分支节点）类树（子）结构的集合，$\mathbb{T}$是所有字节数组和这类结构化序列的集合。

通过两个子函数定义RLP函数，第一个处理当值是一个字节数组时的实例，第二个处理值是一系列更高级的值：
$$
(165)\qquad RLP(X) \equiv \begin{cases} 
R_b(X) \quad if \quad X \in \mathbb{B} \\
R_l(X) \quad otherwise
\end{cases}
$$
如果准备序列化的值是一个字节数组，RLP序列化采用下面三种形式中的一种：

* 如果字节数组只包含一个单独的字节，并且这个字节小于128字节，那么输入和输出严格相等。
* 如果字节数组少于56个字节，那么输出等于输入序列加上前缀后长度等于数组长度加上128 的字节的序列
* 否则，输出等于在输入加上在翻译为一个大端整数时等于输入字节数组长度的最短字节数组前缀，然后在通过该字节数组的长度加上183作为最前面的前缀。

正式的定义为$R_b$:
$$
(166)\qquad R_b(X) \equiv \begin{cases}
X \qquad \qquad \qquad \qquad if \quad \parallel x \parallel = 1 \wedge X[0] < 128 \\
(128 + \parallel X \parallel ) \cdot X \qquad else \quad  if \parallel X \parallel < 56 \\
(183 + \parallel BE(\parallel X \parallel) \parallel ) \cdot BE(\parallel X \parallel) \cdot X \quad otherwise 
\end{cases}
\\
(167)\qquad \qquad \quad  BE(x) \equiv (b_0, b_1, ...)  : b_0 \ne 0 \wedge x = \sum^{n < \parallel b \parallel }_{n = 0} b_n \cdot 256^{\parallel b \parallel -1 -n} \\ 
(168) \qquad \qquad \qquad \qquad \qquad \qquad \qquad  \qquad  (a) \cdot (b, c) \cdot (d, e) = (a,b,c,d,e)
$$


因此，$BE$就是将一个正整数值扩展成一个具有最小长度的大端字节数组的函数，点号操作用于连接序列。

如果准备序列化的值不是一个字节数组，而是一个其他元素的序列，那么RLP序列化采用下面两种形式中的一种：

* 如果包含的元素拼接后的序列长度小于56字节，那么其输出等于拼接后的序列加上一个长度等于该字节数组的字节加上192
* 否则，输出等于凭借后的序列加上翻译为大端整数时长度等于拼接后的字节数组长度的前缀，然后通过该字节数的长度加上247 作为最前面的前缀

正式的定义为$R_l$:
$$
(169)\qquad R_l(X) \equiv \begin{cases}
(192 + \parallel s(X) \parallel) \cdot s(X) \qquad \qquad \qquad \qquad \qquad \quad \ if \quad \parallel s(X) \parallel < 56 \\
(247 + \parallel BE( \parallel s(X) \parallel ) \parallel ) \cdot BE(\parallel s(X) \parallel ) \cdot s(X) \quad otherwise
\end{cases}
\\
(170)\qquad \qquad \qquad \qquad \qquad \qquad \qquad \qquad \qquad \qquad \qquad s(X) \equiv RLP(X_0) \cdot RLP(X_1)...
$$


如果RLP用于对标量编码，之定义为一个正整数（$\mathbb{P} $ 或者任意$x \in \mathbb{P}_x$），必须设定为其大端表达式与其相等最短的字节数组。因此对正整数$i$的RLP定义为：
$$
(171)\qquad RLP(i: i \in \mathbb{P}) \equiv RLP(BE(i))
$$
当翻译RLP数据是，如果一个期望的片段被解码成一个标量，并且在字节序列中首部为0，客户端需要认为这不是权威的数据，并以对待无效RLP数据的方式——完全忽略，来处理。



### 区块、状态以及交易



#### 全局状态

*全局状态*（状态）是一个地址集（160位标识符）与账号状态集（序列化为RLP的数据结构）之间的映射。虽然这个映射并不存储在区块链上，我们仍然假设其具体实现会在一个修改过的默克尔树（trie，字典树）中维护这个映射关系。字典树需要一个简单的后台数据库维护字节数组到字节数组之间的映射关系，这个数据库称为*状态数据库*。这样做有很多好处：

1. 这个数据结构的根节点是加密依赖其全部的内部数据，因此它的散列值可以作为整个系统的安全标识符使用
2. 作为一个*不可变*的数据结构，它允许通过简单的切换根散列值来回溯到任意一个前续状态（根散列是已知晓的）

因为在区块链中存储了所有的根散列值，所以能够很简单的回退到旧的状态。

账号状态包括下列4个字段：

* nonce：标量，等于从当前地址发出的交易数量。对于带有关联代码的账户，等于当前账号创建的合约数量。对处于状态$\sigma$的账号*a*，该值用$\sigma[a]_n$表示
* balance：标量，与当前账户持有*Wei*的数量相同，用$\sigma[a]_b$表示
* storageRoot：一棵对账户的存储内容（256比特整数值之间的一个映射）进行编码的默克尔树的根节点的256比特散列值。字典树的内容是一个256比特整数的Keccak 256比特散列与RLP编码的256比特值之间映射关系的编码。这个散列值用$\sigma[a]_s$表示
* codeHash：当前账号的**EVM**代码的散列值，这个代码只有接收到一个消息调用时才执行；该字段是*不可变*的，因此和其他字段不同的是在创建以后无法修改。所有的这些代码段都包含与他们的散列值对应的状态数据库中，以用于后续检索。该散列值用**b**表示，且已知$KEC(b) = \sigma[a]_c$

*为了引用存储在字典树底层的键值对集合而非引用字典树的根散列值，定义了下面的等式*：
$$
(6)\qquad TRIE(L^*_I(\sigma[a]_s)) \equiv \sigma[a]_s
$$
其中，$L^*_I$是字典树中键/值对的塌陷函数（collapse function）是以基础函数$L_I$在元素级的变体定义的，其中$L_I$定义如下：
$$
(7) \qquad L_I((k,v)) \equiv (KEC(k), RLP(v)) 
$$

$$
(8) \qquad k \in \mathbb{B_{32}} \wedge v \in \mathbb{P}
$$

$\sigma[a]_s$并不是账号的“物理”成员，也不会为其随后的序列化贡献任何力量。

如果**codeHash**字段是空字符串的Keccak-256散列，例如，$\sigma[a]_c = KEC(())$, 那么这个节点就表示一个*简单账号*，有时也称作“非合约”账号。因此我们可以定义一个全局状态塌陷函数$L_S$:
$$
(9) \qquad L_S(\sigma) \equiv \{p(a) : \sigma[a] \ne \phi\}
$$
其中，
$$
(10) \qquad p(a) \equiv (KEC(a), RLP((\sigma[a]_n, \sigma[a]_b, \sigma[a]_s, \sigma[a]_c)))
$$
函数$L_S$和字典函数一起为全局状态提供一个短标识符（散列），假设：
$$
(11) \qquad \forall a : \sigma[a] = \phi \vee  (a \in \mathbb{B_{20} \wedge v(\sigma[a])})
$$
其中$v$是账号的验证函数：
$$
(12) \qquad v(x) \equiv x_n \in \mathbb{P_{256}} \wedge x_b \in \mathbb{P_{256}} \wedge x_s \in \mathbb{B_{32}} \wedge x_c \in \mathbb{B_{32}}
$$

#### 交易

一笔交易是对以太坊来说一个外部操作者发起的一条简单的加密签名指令。外部操作者是一个自然人，软件工具可以用于其创建和传播。目前存在**2**种交易：

* 产生消息调用的交易
* 产生带有关联代码的新账户的创建（以非正式名称“合约创建”而熟知）的交易

这2种交易都指定了一些通用字段：

* nonce：标量，与发送者发送的交易数相同，用$T_n$表示。
* gasPrice: 标量，等于执行当前交易所消耗的计算量以每单位*gas*需要支付的*Wei*数，用$T_p$表示。
* gasLimit：标量，等于执行当前交易所要使用的*gas*上限。这个值是在任何计算完成之前预支的，随后可能并不增加，用$T_g$表示。
* to：消息调用接收者的160比特地址，对于*合约创建*交易，此处用$\phi$表示仅有的$\mathbb{B_0}$的成员，用$T_t$表示。
* value：标量，与准备转给消息调用的接收者的*Wei*数一致，对于*合约创建*来说作为新创建的账号的资金，用$T_v$表示。
* v, r, s：与交易的签名有关的值，被用于判定交易的发送者，用$T_w, T_r, T_s$表示。

此外，一笔*合约创建*交易包括：

* init：一个没有大小限制的字节数组，设定账户初始化过程的*EVM-code*， 用$T_i$表示。

**init**是一个**EVM-code**片段，其返回的是在每次收到消息调用（通过交易或者内部代码执行）时需要执行的第二个代码段 — **body**。**init**只在账号创建时执行一次，随后立即被丢弃。

相反，一个消息调用交易包括：

* data：一个没有大小限制的字节数组，用于指定消息调用的输入数据，用$T_d$表示。

$$
（13）\\
L_T(T) \equiv \begin{cases}
(T_n, T_p, T_g, T_t, T_v, T_i, T_w, T_r, T_s) & if  T_t = \phi \\
(T_n, T_p, T_g, T_t, T_v, T_d, T_w, T_r, Ts) & otherwise

\end{cases}
$$

这里假设除了任意长度的字节数组$T_i$和$T_d$，所有的模块都用RLP解释为整数。
$$
(14) \\
T_n \in \mathbb{P_{256}} \wedge T_v \in \mathbb{P_{256}} \wedge T_p \in \mathbb{P_{256}} \wedge \\
T_g \in \mathbb{P_{256}} \wedge T_w \in \mathbb{P_{5}} \wedge T_r \in \mathbb{P_{256}} \wedge \\
T_s \in \mathbb{P_{256}} \wedge T_d \in \mathbb{B} \wedge T_i \in \mathbb{B}
$$
其中，(15)  $P_n = \{ P: P \in \mathbb{P} \wedge P \lt 2^n\}$

地址散列$T_t$有稍许不同：要么是以个20字节的地址散列或者，要么对于*合约创建*交易（因此形式上等价于$\phi$）来说它是RLP空字节序列，因此也是$\mathbb{B_0}$的成员：
$$
(16) \\
T_t \in \begin{cases} 
\mathbb{B_{20}} &  if & T_t \ne \phi \\
\mathbb{B_0} & otherwise
\end{cases}
$$

#### 区块

![区块头部](/resources/blockchain/block_header.png)

在以太坊中*区块*是由相关信息的头部**H**， 包含的交易信息**T**，以及其他与当前区块的父节点具有相同父节点的区块头（即叔节点的区块头）集合**U**组成。

区块头部包含下列信息：

* parentHash：父区块头完整的Keccak 256位散列，用$H_p$表示
* ommersHash：叔区块列表的Keccak 256位散列，用$H_o$表示。
* beneficiary：为成功采集到当前区块所手机的有费用的160位转移地址。
* stateRoot：在全部交易执行完成之后，状态字典树根节点的Keccak 256位散列，用$H_r$表示。
* transactionsRoot：字典树结构的根节点，携带当前区块交易列表中每一笔交易的Keccak 256位散列，用$H_t$表示。
* receiptsRoot：字典树结构的根节点，携带当前区块交易列表中每一笔交易收据的Keccak 256位散列，用$H_e$表示。
* logsBloom：由交易列表中的每一笔交易收据中所包含的日志条目的可索引信息（日志地址和日志主题）组成的布隆过滤器，用$H_b$表示。
* difficulty：与当前区块的难度级别相关的标量。可以通过前驱区块的难度级别与时间戳计算得出，用$H_d$表示。
* number：当前区块的所有前驱区块的数量，创世区块的number为0，用$H_i$表示。
* gasLimit：标量，与目前每个区块的gas配额限制相同，用$H_l$表示。
* gasUsed：标量，与当前块中所有交易所耗费的gas总数相同，用$H_g$表示。
* timestamp：标量，与当前块诞生时的Unix系统time()合法输出相同，用$H_s$表示。
* extraData：一个包含当前区块相关数据的字节数组，必须小于等于32字节，用$H_x$表示。
* mixHash：一个256位的散列值，与*nonce*一起用于证明在当前区块上已经执行了足够的计算，用$H_m$表示。
* nonce：一个64位散列值，与*mixHash*一起用于证明在当前区块上已经执行了足够的计算，用$H_n$表示。

区块的另外两个模块比较简单，只是叔区块的头部列表（格式与前面描述的头部格式相同）和当前块中包含的交易序列。形式上说我们可以定义区块*B*为：
$$
(17)\\
B \equiv (B_H, B_T, B_U)
$$

##### 交易收据

为了将一笔交易中对构成一个*零知识证明*有用的信息进行编码，或者进行索引与搜索，我们编码了每一笔交易的收据，其中包含交易执行的信息。对于第$i$笔交易，其收据用$B_R[i]$表示，并放置于一个键索引的字典树中，树的根节点在区块头部中以$H_e$记录。

交易收据是一个带有4个成员的元组，其成员包括：

* *交易后状态*$R_\sigma$,
* 截止当前交易发生时刻，包含当前交易的区块中累积使用的gas量$R_u$,
* 当前交易执行过程中创建的日志集合$R_l$,
* 从日志信息中选择的布隆过滤器$R_b$

即，一个交易收据可以表示为：
$$
(18) \\

R \equiv (R_\sigma, R_u, R_b, R_l)
$$
函数$L_R$用于为正在转换成一个RLP序列化的字节数组准备一个交易收据：
$$
(19) \\
L_R(R) \equiv (TRIE(L_S(R_\sigma)), R_u, R_b, R_l)
$$
因此，交易后状态$R_\sigma$会被编码进一个字典树结构，其根节点为第一个成员。

需要声明累积使用的gas量$R_u$是一个正整数，并且日志布隆过滤器$R_b$是一个2048比特（256字节）的散列：
$$
(20) \\
R_u \in \mathbb{P} \wedge R_b \in \mathbb{B_{256}}
$$
日志集合$R_l$是一些列日志条目，例如$(O_0, O_1, …)$。一个日志条目$O$是一个由日志记录器地址$O_a$， 一系列32字节的日志主题$O_t$和数个字节的数据$O_d$构成的元组：
$$
(22) \qquad \quad \quad \ \ O \equiv (O_a, (O_{t0}, O_{t1}, ...), O_d) \\
(23) \quad O_a \in \mathbb{B_{20}} \wedge \forall_{t \in O_t} : t\in \mathbb{B32} \wedge O_d \in \mathbb{B}
$$
布隆过滤器函数$M$用于将一个日志条目缩减成一个256字节的散列值：
$$
(23) \\
M(O) \equiv \bigvee M_{3:2048}(t)) \\
                    t \in \{O_a\} \cup O_t
$$
其中，$M_{3:2048}$是一个特殊的布隆过滤器，它对给定的任意一个字节序列从2048个比特中设置3个比特。其实现是通过对字节序列的Keccak-256散列的前3对字节的最低11比特进行操作：
$$
(24) M_{3:2048}(x:x \in \mathbb{B}) \equiv y: y \in \mathbb{B_{256}}\quad where: \\
(25)\quad\qquad\qquad\qquad y = (0, 0, ..., 0) \quad except: \\
(26)\quad\qquad\qquad\quad \forall_i \in \{0,2,4\}   :   \mathcal{B_{m(x, i)}}(y) = 1 \\
(27)\quad\quad m(X, i) \equiv KEC(X)[i, i + 1] \ mod\ 2048
$$

其中，$\mathcal{B}$是比特引用函数，例如$B_j(x)$等于字节数组$X$中下标为$j$的比特（下标从0开始）。



##### 整体有效性

当且仅当区块满足下列条件时开可以说它是有效的：其必须和叔区块以及交易区块散列保持内部一致性，并且给定的交易集$B_T$在基础状态$\sigma$（继承自父区块的完成状态）的基础上按顺序执行，从而生成一个新实体$H_r$的状态：
$$
(28) \\
H_r \equiv TRIE(L_S(\Pi(\sigma, B)))\qquad\qquad\qquad\qquad\qquad\qquad \wedge \\
H_o \equiv KEC(RLP(L^*_H(B_\cup))) \qquad\qquad\qquad\qquad\qquad\quad \wedge \\
H_t \equiv TIRE(\{\forall _i < \parallel B_T \parallel, i \in \mathbb{P} : p(i, L_T(B_T[i]))\})\qquad \wedge \\
H_e \equiv TIRE(\{\forall _i < \parallel B_R \parallel, i \in \mathbb{P} : p(i, L_R(B_R[i]))\})\qquad \wedge \\
H_b \equiv \bigvee_{r \in B_R(rb)} \qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad\quad
$$
其中$p(k,v)$是一对简单的RLP变换，这里第一个元素是交易在区块中的下标，第二个元素是交易收据：
$$
(29)\qquad\qquad p(k,v) \equiv (RLP(k), RLP(v))
$$
进一步说：
$$
(30)\qquad\qquad TRIE(L_S(\sigma)) = P(B_H)_{H_r}
$$
因此$TRIE(L_S(\sigma))$即包含状态$\sigma$与RLP编码的值的键值对的默克尔树的根节点的散列，$P(B_H)$是区块$B$的父区块。

源于交易计算的结果集，确切的说是交易收据$B_R$以及交易状态累积函数$\Pi$在下文列出。



##### 序列化

函数$L_B$和$L_H$是区块及其区块头部的准备函数。与交易收据准备函数$L_R$类似，当需要RLP变换的时候，需要判断类型和结构顺序：
$$
(31)\qquad L_H(H) \equiv (H_p, H_o, H_c, H_r, H_t, H_e, H_b, H_d, \\
H_i, H_l, H_g, H_s, H_x, H_m, H_n) \\
(32)\quad\quad\quad\ \  \ L_B(B) \equiv (L_H(B_H), L^*_T(B_T), L^*_H(B_U))
$$
$L^*_T$和$L^*_H$都是元素级别的序列转换函数，因此：
$$
(33)\ f^*((x_0, x_1, ...)) \equiv (f(x_0), f(x_1), ...) \qquad for\ any\ function\ f
$$
分量类型定义如下：
$$
(34) \\
H_p \in \mathbb{B_{32}} \wedge H_o \in \mathbb{B_{32}} \wedge H_c \in \mathbb{B_{20}} \qquad \wedge \\
H_r \in \mathbb{B_{32}} \wedge H_t \in \mathbb{B_{32}} \wedge H_e \in \mathbb{B_{32}} \qquad \wedge \\
H_b \in \mathbb{B_{256}} \wedge H_d \in \mathbb{P} \wedge H_i \in \mathbb{P} \qquad\quad\  \wedge \\
H_l \in \mathbb{P} \wedge H_g \in \mathbb{P} \wedge H_s \in \mathbb{P_{256}} \qquad\quad\ \ \  \wedge \\
$$
其中，$(35)\quad \mathbb{B_n} = \{B : B \in \mathbb{B} \wedge \parallel B \parallel = n\}$。关于如何构建一个正式的区块结构现在有了严格的规范，**RLP**

函数则提供了权威的方法将区块结构变换成一个可用于传输的字节序列。



##### 区块头部校验

定义$P(B_H)$为区块$B$的父亲区块，形式如下：
$$
(36)\qquad P(H) \equiv B' : KEC(RLP(B'_H)) = H_p
$$
区块号通过父区块号递增1获得：
$$
(37)\quad H_i \equiv P(H)_{H_i} + 1
$$
一个区块的头部$H$的难度定义为$D(H)$:
$$
(38) \\
D(H) \equiv \begin{cases} 
D_0\qquad\qquad\qquad\qquad\qquad\qquad\qquad if\quad H_i = 0 \\
max(D_0, P(H)_{H_d} + x \times \varsigma_2 + \epsilon) \qquad otherwise
\end{cases}
$$
其中：

$(39)\qquad D_0 \equiv 131072$

$(40)\qquad x \equiv \lfloor \frac{P(H)_{H_d}}{2048} \rfloor$  

$(41)\qquad \varsigma_2 \equiv max (1 - \lfloor \frac{H_s - P(H)_{H_s}}{10} \rfloor, -99)​$

$(42)\qquad \epsilon \equiv \lfloor 2^ {\lfloor H_i \div 10000 \rfloor - 2} \rfloor$

区块头部$H$的gas限制$H_l$必须满足关系：

$(43)\qquad H_l < P(H)_{H_l} + \lfloor \frac {P(H)_{H_l}}{1024} \rfloor \qquad \wedge$

$(44)\qquad H_l > P(H)_{H_l} - \lfloor \frac{P(H)_{H_l}}{1024} \rfloor \qquad \wedge$

$(45)\qquad H_l \ge 125000$

$H_s$是区块$H$的时间戳，且需满足关系：

$(46)\qquad H_s > P(H)_{H_s}$

这个机制迫使区块之间以时间来保持内部平衡，最新两个区块之间一个更小的时间窗会导致难度级别的增加，从而需要额外的计算来延长下一个时间窗。相反，如果时间窗太大则难度，即下一个区块的期望时间会被缩小：

$(47)\qquad n \le \frac {2^{256}}{H_d} \wedge m = H_m, \qquad (n, m) = PoW(H_{n'}, H_n, d)$

$H_{n'}$是新区块的头部，但是*没有*nonce和mix-hash分量，**d**是当前的**DAG** — 一个计算mix-hash需要的庞大的数据集，**PoW**是工作量函数：这个计算得出一个数组，第一个元素是mix-hash用于证明使用了一个正确的**DAG**；第二个元素是一个密码学上依赖于*H*和**d**的随机数。对于区间[0, $2^{64}$)上近似于均态的分布，找到结果的期望时间与难度$H_d$成正比。这是区块链的安全基础，也是为什么恶意节点无法将新创建的可以重写历史记录的区块进行传播的根本原因。由于*nonce*需要满足这个需求，且因为其满足的情况依赖于区块的内容，从而反过来随着时间推移，区块中包含的交易用于创建新的有效区块会非常困难，几乎需要矿工中全部可信节点的计算力才能完成。

因此我们可以定义头部校验函数$V(H)$:
$$
(48)\qquad V(H) \equiv n  \le \frac{2^{256}}{H_d} \wedge m = H_m \wedge \\
(49)\qquad\qquad\qquad\qquad\quad\ \  H_d = D(H) \wedge \\
(50)\qquad\qquad\qquad\qquad\quad \ \ \ \ \ \ \ H_g \le H_l \wedge \\
(51)\qquad\ \ \  H_l < P(H)_{H_l} + \lfloor\frac{P(H)_{H_l}}{1024}\rfloor \wedge \\
(52)\qquad\ \ \ H_l > P(H)_{H_l} - \lfloor\frac{P(H)_{H_l}}{1024}\rfloor \wedge \\
(53)\qquad\qquad\qquad\qquad\ \ \  H_l \ge 125000 \wedge \\
(54)\qquad\qquad\qquad\qquad\ \ \ H_s > P(H)_{H_s} \wedge \\
(55)\qquad\qquad\qquad\quad  H_i = P(H)_{H_i} + 1 \wedge \\
(56)\qquad\qquad\qquad\qquad\qquad \parallel H_x \parallel \le 32
$$
其中，$(n, m) = PoW(H_{n'}, H_n, d)$，**extraData**最大不超过32字节。



### GAS和报酬

为了避免滥用网络和图灵完备性带来的固有问题[^3] 以太坊中所有的可编程计算都与费用挂钩，费用以*gas*为单位。因此任何给定的可编程计算的片段（包括合约创建，消息调用，访问和使用账号存储，以及在虚拟机上执行操作）都有一个达成共识的以*gas*计算的成本。

每笔交易都有一个与之相关的特别的*gas*数量：**gasLimit**，是隐式从发送者账户余额购买的。购买行为以交易中指定的*gasPrice*进行。如果账户余额不足以支撑购买需要消耗的*gas*，交易将会被当作是无效交易。之所以被称为*gasLimit*是由于在交易结束时任何尚未使用的gas将会被退还给发送者的账户中（与购买时的汇兑相同）。**Gas**在交易执行之外是无法存在的，因此对于具有可信的相关代码的账户，可能会被单独设定并保留一个相对较高的*gasLimit*。

通俗的讲，用于未被退回的用于购买*gas*的以太币将会被发送给*受益人*地址，账户地址在矿工的控制之下是一种典型情况。交易者可以随意指定其想要设定的任何*gasPrice*，但是矿工可以选择忽略该笔交易。一个具有更高*gas*价格的交易将会耗费发送者更多以太币并向矿工支付更高的价值，因此更容易被矿工选中。矿工通常会公布其执行交易的最低*gas*价格，交易者可以在决定提供什么样的交易价格时忽略这些公布的价格。由于存在一个（带权重的）最低可接受的*gas*价格区间，交易者需要在更低的*gas*价格和最大化交易被矿工快速选中之间进行权衡。

### 交易的执行

交易的执行是以太坊协议中最复杂的部分：这部分定义了状态转移函数$\Upsilon$。假设任何被执行的交易首先已经通过了基本的有效性测试，包括：

1. 交易是结构完好的RLP，没有额外的尾字节；
2. 交易签名是有效的；
3. 交易的*nonce*是有效的（等于发送者账户的当前nonce）；
4. 交易使用的gas限制不比基本gas数$g_0$小；
5. 发送者的账户余额包含预先支付的最低成本$v_0$。

考虑函数$\Upsilon$，交易$T$和状态$\sigma$，其正式的形式如下：


$$
(57)\qquad \sigma' = \Upsilon(\sigma, T)
$$

$\sigma'$就是交易后的状态。我们还定义了$\Upsilon^g$用于估算交易执行需要的gas数量，$\Upsilon^l$用于评估交易的累积日志元素。



#### 子状态

在交易执行过程中，一直在累积紧随交易之后立刻操作的信息，这些信息称为*交易子状态*，用元组$A$表示:
$$
(58)\qquad A \equiv (A_s, A_l, A_r)
$$
元组内容包括自销毁的集合$A_s$：一个在交易完成之后丢弃的账号集合。$A_l$是日志序列：在VM代码执行时一系列归档并可索引的“检查点”，这样合约调用就可以很容易的被以太坊之外（比如去中心化的应用前端）的旁观者追踪。最后是退款余额$A_r$，该值通过使用SSTORE指令将合约存储从非零值重置为0来增加。虽然不是立即退款，也仍然允许部分抵扣总的执行成本。

简而言之，空的非自毁的、无日志、0退款余额的子状态$A^0$定义如下：
$$
A^0 \equiv (\phi， ()， 0)
$$

#### 执行

定义当前交易在执行之前需要支付的基本gas数量 $g_0$如下：
$$
(60)\qquad \sum_{i \in T_i, T_d} \begin{cases}
G_{txdatazero} \qquad\quad if\ \ i = 0 \\
G_{txdatanonzero} \qquad otherwise
\end{cases} 
\\
(61)\qquad\qquad \ \ \  +\begin{cases}
G_{txcreate} \qquad if\quad T_t = \phi \\
0 \qquad\qquad \quad otherwise
\end{cases} 
\\
(62) +G_{transaction}
$$
$T_i, T_d$表示根据交易是合约创建还是消息调用，与交易关联的数据和初始化EVM代码的字节序列。

$G_{txcreate}$在交易是合约创建时增加，但如果是EVM-code的结果则不添加。$G$的完整定义在附录G中。

预支成本$v_0$通过下面的公式计算:
$$
(63)\qquad v_0 \equiv T_gT_p + T_v
$$
有效性由下列条件决定：
$$
(64) \\
S(T) \ne \phi \wedge \\
\sigma[S(T)] \ne \phi \wedge \\
T_n = \sigma[S(T)]_n \wedge \\
g_0 \le T_g \wedge \\
v_0 \le \sigma[S(T)]_b \wedge \\
T_g \le B_{Hl} - \mathcal {l}(B_R)_u
$$
注意最后一个条件，交易的gas限制$T_g$与当前区块之前使用的gas量$l(B_R)_u$之和必须不大于当前区块的**gasLimit**。

一笔有效的交易的执行以一个不可撤销的状态变化开始：发送者账号的nonce值$S(T)$加一，同时余额减去预支的成本$T_gT_p$。可用的用于处理计算的gas值$g$定义为$T_g - g_0$。不论是合约创建还是消息调用的计算都会产生一个最终状态（可能逻辑上与当前状态等价），对其改变是确定的，并且不会无效：从当前点开始不可能有无效的交易。

检查点的状态$\sigma_0$定义为：
$$
(65)\qquad\qquad\quad\ \  \sigma_0 \equiv \sigma \quad except: \\
(66)\  \sigma_0[S(T)]_b \equiv \sigma[S(T)]_b - T_gT_p \\
(67)\quad\ \  \sigma_0[S(T)]_n \equiv \sigma[S(T)]_n + 1
$$
从$\sigma_0$计算$\sigma_P$由交易类型决定，对于合约创建和消息调用，定义由执行之后的临时状态$\sigma_P$， gas余量 $g'$和子状态$组成的元组如下:
$$
(68) \\
(\sigma_P, g', A) \equiv \begin{cases}
\bigwedge(\sigma_0, S(T), T_o, \\
\qquad g, T_p, T_v, T_i, 0) \qquad\qquad \qquad if \ T_t = \phi \\
\Theta_3(\sigma_0, S(T), T_o,  \\
\qquad T_t, T_t, g, T_p, T_v, T_v, T_d, 0) \quad otherwise
\end{cases}
$$
$g$是在扣除交易存在性需要支付的基本gas后的剩余gas：

$(69)\qquad g \equiv T_g - g_0$

$T_o$是原始交易者，可能与通过EVM代码而非交易直接触发的消息调用或者合约创建的发送者不同。

$\Theta_3$ 表示只有函数值的前三个分量会被取用；消息调用的输出价值（一个字节数组）的最终表示，并且在交易计算的上下文中是不使用的。

消息调用或者合约创建被处理之后，最终状态由以原始速率退给发送者的退款数量$g^*$决定，其值等于剩余的gas量$g'$加上一些退款计数器的补贴：
$$
(70)\qquad g^* \equiv g' + min\{\lfloor \frac{T_g - g'}{2} \rfloor, A_r\}
$$
可退还的总数是剩余gas量$g'$加上$A_r$， 第二部分为$A_r$与已使用的总量$T_g - g'$的一半（下除法）的最小值。

gas需要的以太币是给予矿工地址的，矿工的地址被设定为当前区块$B$的受益地址。所以我们依据状态$\sigma$来定义预终状态$\sigma^*$：
$$
(71)\qquad \qquad\quad \sigma^* \equiv \sigma_P \qquad except \\
(72) \ \sigma^*[S(T)]_b \equiv \sigma_P[S(T)]_b + g^*T_p \\
(73) \ \sigma^*[m]_b \equiv \sigma_P[m]_b + (T_g - g^*)T_p \\
(74) \qquad\qquad\qquad\qquad\quad\ \ \   m \equiv B_{Hc}
$$
在删除自毁集合中的所有账号之后达到最终状态$\sigma'$：
$$
(75)\qquad\quad \sigma' \equiv \sigma^* \quad except \\
(76)\qquad \forall i \in A_s : \sigma'[i] \equiv \phi
$$
最后设定在这笔交易中使用的gas总数$\Upsilon^g$和当前交易创建的日志$\Upsilon^ l$：
$$
(77)\qquad \Upsilon^g(\sigma, T) \equiv T_g - g' \\
(78)\qquad\qquad \Upsilon^l(\sigma, T) \equiv A_l
$$

### 合约的创建

在创建一个账户时有很多基本参数：发送者(s)， 原始交易者（o）， 可用的gas（$g$）， gas价格（$p$)， 捐款（$v$)， 以及一个任意长度的字节数组**i**，EVM初始化代码和消息调用/合约创建的当前栈（$e$）深度。创建函数正式定义为$\Lambda$, 用于从这些值和状态$\sigma$来计算出一个包含新的状态，剩余gas和累积交易子状态的元组（$\sigma', g', A$）：
$$
(79)\qquad (\sigma', g', A) \equiv \Lambda(\sigma, s, o, g, p, v, \mathcal{i}, e)
$$
新账户的地址$a$定义为包含发送者和nonce的结构的RLP编码的Keccak 散列值的高位160比特：
$$
(80)\qquad a \equiv \mathcal{B}_{96..255}(KEC(RLP((s, \sigma[s]_n - 1))))
$$
其中，**$KEC$**是Keccak 256位散列函数，**RLP**是**RLP**编码函数，$\mathcal{B}_{a..b}(X)$计算出包含二进制数据**X**在范围[a,b]内的二进制值，$\sigma[x]$是$x$或者$\phi$（若不存在）的地址状态。注意我们使用的nocne比发送者的nonce小1；需要判断在该调用之前已经为发送者账户增加了nonce，因此使用的是发送者在可响应的交易或者VM操作开始时的nonce值。

账户的nonce初始值为0， 余额为传入的值，存储为空，代码散列值为空字符串的Keccak 256位散列。发送者的余额也是通过传入的值扣除，因此可变的状态变成了$\sigma^*$:
$$
(81) \qquad \qquad\qquad\qquad\quad \sigma^* \equiv \sigma \quad except: \\
(82) \ \sigma^*[a] \equiv (0, v+ v', TRIE(\phi), KEC()) \\
(83) \qquad\qquad\qquad\qquad\ \ \  \sigma^*[s]_b \equiv \sigma[s]_b - v
$$
$v'$是账户之前已存在的价值，在当前事件中是之前存在的：
$$
(84)\qquad v' \equiv \begin{cases}
0 \quad if \quad \sigma[a] = \phi \\
\sigma[a]_b \quad otherwise
\end{cases}
$$
最后，根据执行模型执行EVM初始化代码**i**对账户完成初始化。代码执行可以影响很多并非执行状态内部的事件：账户的存储可能被改变，新的账户可以被创建以及发起新的消息调用。因此，代码的执行函数$\Xi$计算出一个包括结果状态$\sigma^{**}$， 可用的gas余额$g^{**}$，累积的子状态$A$和账户的实体代码$o$。
$$
(85)\quad (\sigma^{**}, g^{**}, A, o) \equiv \Xi(\sigma^*,  g, I)
$$
其中$I$包含了定义在*执行模型*中的执行环境参数：
$$
(86)\qquad I_a \equiv a \\
(87)\qquad I_o \equiv o \\
(88)\qquad I_p \equiv p \\
(89)\qquad I_d \equiv () \\
(90)\qquad I_s \equiv s \\
(91)\qquad I_v \equiv v \\
(92)\qquad I_b \equiv i \\
(93)\qquad I_e \equiv e
$$
$I_d$计算的结果是一个空元组，因为对于这个调用没有输入数据。$I_H$没有特殊处理由区块链决定。

代码执行会耗尽gas，并且gas不能降到0以下，因此执行可能在代码达到一个自然的停机状态之前就退出。在这种（和其他的）异常情况中，我们说出现了一个gas耗尽（out-of-gas, OOG)异常：计算出来的状态定义为空集$\phi$， 并且这个创建操作对状态没有任何影响，只保留以试图创建合约时的即刻状态。

如果初始化代码成功的完成，需要支付最终的合约创建开销，即代码保证金$c$，其值与创建的合约代码大小成比例：
$$
(94) \quad c \equiv G_{codedeposit} \times |o|
$$
如果缺乏足够的gas余额来支付这笔开销，比如$g^{**} < c$， 那么也会抛出一个*out-of-gas*异常。

在任何出现此类异常的情况下gas余额都会使0，例如，如果合约的创建是由一笔交易的接受主导的，那么这就不会影响合约创建的固有成本的支付，不管怎样都必须支付。但是当出现out-of-gas时，交易的价值并没有转移到异常退出的合约地址上。

如果没有出现这个异常，剩余的gas会被退还给发起人并且当前修改过的状态可以持久化了。因此正式形式，可以将结果状态、gas和子状态设定为$(\sigma', g', A)$, 其中：
$$
(95)\qquad g' \equiv \begin{cases}
0 \quad if \quad \sigma^{**} = \phi \\
g^{**} - c \quad otherwise 
\end{cases} 
\\
(96)\ \sigma' \equiv \begin{cases}
\sigma \qquad \qquad\quad\qquad \quad if \quad  \sigma^{**} = \phi \\
\sigma^{**} \quad except: \\
\sigma'[a]_c = KEC(o) \quad otherwise
\end{cases}
$$
$\sigma'$的决定条件中的异常$o$是初始化代码执行产生的字节序列设定了新创建的账户最终的主体代码。

需要注意的是，结果要么成功的创建带有额度的合约，要么不存在合约与价值转移。



#### 微妙之处

注意，当初始化代码正在执行时，新创建的地址是存在的只不过没有基本的主体代码。因此在这段时间内该地址收到的任何消息调用都会导致没有代码执行。如果初始化执行以**SELFDESTRUCT**指令结束是没有意义的，因为在交易完成之前这个账户就会被删除。对于一个正常的**STOP**代码，或者代码返回的是一个空集，就会以一个僵尸账户的状态保留，并且所有剩余的余额都会被永久的锁定在该账户中。




### 消息调用

在执行消息调用的情况下需要这些参数：发送者（s），交易创建者（o），接收者（r），执行代码的账户（c，通常与接收者相同），可用的gas（g），价值（v）和gas价格（p）以及任意长度的输入数据字节数组（d）和消息调用/合约创建的当前栈深度（e）。

除了计算新的状态和交易的子状态，消息调用还有一个额外的分量—用字节数组$o$表示的输出数据。这个数据在执行交易时是被忽略的，但是消息调用可以通过执行VM代码初始化，在这种情况下这个数据就要被使用了。
$$
(97)\qquad (\sigma', g', A, o) \equiv \Theta(\sigma, s, o, r, c, g, p, v, \tilde{v}, d, e)
$$
**注意**，需要区分传输的数据$v$和执行上下文中出现的用于**DELEGATECALL**的价值$\tilde{v}$。

第一个转移状态$\sigma_1 $作为原始状态，但是带有从发送者转移给接收者的价值(除非$s = r$)：
$$
(98)\qquad \sigma_1[r]_b \equiv \sigma[r]_b + v \wedge \sigma_1[s]_b \equiv \sigma[s]_b -v
$$
在当前工作中，假设如果$\sigma_1[r]$最初是未定义的，其会被作为一个缺乏代码或者缺乏状态、0余额，0 nonce的账号创建出来。因此之前的等式必须表示：
$$
(99)\qquad \sigma_1 \equiv \sigma'_1 \quad except: \\
(100)\qquad \sigma_1[s]_b \equiv \sigma'_1[s]_b - v \\
(101)\quad \ \  and\ \sigma'_1 \equiv \sigma \ except: \\
(102)\quad \begin{cases}
\sigma'_1[r] \equiv (v, 0, KEC(()), TRIE(\phi)) \quad  if \ \sigma[r] = \phi \\
\sigma'_1[r]_b \equiv \sigma[r]_b + v \qquad \qquad \qquad \qquad  otherwise
\end{cases}
$$
账户的关联代码（标记为Keccak散列为$\sigma[c]_c$的片段）依照执行模型（后文）执行。只是与合约创建相同，如果执行异常终止（比如，因为耗尽了gas，栈下溢，无效的跳转目的地或者无效的指令），那么不就会向调用者退回gas，且状态会被撤回到在余额转移（比如，$\sigma$）之前的时间点。
$$
(103)\qquad \sigma' \equiv \begin{cases}
\sigma \quad if \quad \sigma^{**} = \phi \\
\sigma^{**} \ otherwise
\end{cases}
\\
(104)\qquad g' \equiv \begin{cases}
0 \quad if \quad \sigma^{**} = \phi \\
g^{**} \ otherwise
\end{cases}
\\
(105)\qquad (\sigma^{**}, g^{**}, A, o) \equiv \begin{cases}
\Xi_{ECREC}(\sigma_1, g, I) \quad if \quad r = 1 \\
\Xi_{SHA256}(\sigma_1, g, I) \quad if \quad r = 2 \\
\Xi_{RIP160}(\sigma_1, g, I) \quad if \quad r = 3 \\
\Xi_{ID}(\sigma_1, g, I) \quad \quad \ \ if \quad r = 4 \\
\Xi(\sigma_1, g, I) \qquad \quad \ \  otherwise
\end{cases}
\\
(106)\qquad I_a \equiv r \\
(107)\qquad I_o \equiv o \\
(108)\qquad I_p \equiv p \\
(109)\qquad I_d \equiv d \\
(110)\qquad I_s \equiv s \\
(111)\qquad I_v \equiv \tilde{v} \\
(112)\qquad I_e \equiv e \\
(113)\qquad Let KEC(I_b) = \sigma[c]_c
$$
假设客户端会在某个时间点预先将$KEC((I_b), I_b)$对存储下来，以让条件$I_b$可用。

正如我们所见，通用执行框架$\Xi$在计算消息调用时有**4**个异常：这是4个所谓的“预编译”合约，表示随后可能成为*原生扩展*的架构的初始部分。这四个合约在地址$1,2,3, 4$执行椭圆曲线公钥恢复函数，SHA2 256位散列算法，RIPEMD 160位散列以及对应的身份函数。



### 执行模型

执行模型表述了给定一系列字节码指令和一个很小的环境数据元组，系统状态是被如何改变的。这是通过一个正式的称为以太坊虚拟机（EVM）的虚拟状态机模型规范的。EVM是一个准图灵完备的机器；这里用“准”修订词是因为计算与$gas$参数固有的绑定造成的，因为这限制了可完成的总计算量。



#### 基础

**EMV**是一个简单的基于栈的体系结构。虚拟机的字长（也就是栈成员的大小）是256比特。选择这个长度是为了方便使用Keccak-256散列算法和椭圆曲线计算。其内存模型是一个简单的字寻址字节数组，栈的最大长度是1024。虚拟机还有一个独立的存储模型，与内存类似但不是一个字节数组(byte array)，而是一个字寻址的字数组(word array)。与内存不同， 存储区域不是瞬变的，而且是作为系统状态的一部分维护的。所有的存储和内存区域都被初始化为0.

虚拟机并不遵循标准的冯诺依曼体系结构，程序代码并不是存储在通用可访问的内存或者存储区域中的，而是存储在一个隔离的可通过一个特殊指令交互的虚拟**ROM**中。

虚拟机可能因为一些原因异常执行，包括栈下溢和无效的指令。和*out-of-gas*异常类似，这些异常并不能使状态保持完好，实际上，虚拟机会立刻停机并向会单独处理异常的异常代理（交易处理器或者递归创建的执行环境）报告。


#### 费用概览

费用（以gas为单位结算）在3种不同的情况下会被扣除，这三种情况都是一个操作能够执行的先决条件。

首先，最常见的是操作计算的固有费用。

其次，gas可能会被用于支付一个次级消息调用或者合约创建，这构成了用于CREATE, CALL和CALLCODE的报酬。

最后，随着内存使用的增加而支付gas。

一个账户的整个执行中，内存消耗的可支付总费用是与包含所有内存偏移（无论读还是写）的32字节的最小倍数成正比的。这是在“及时”基础上的一种支付。正因如此，引用一段至少比前面索引的内存大32字节的内存都会产生一笔额外的使用费。因为这笔费用，非常不推荐寻址超过32比特上限。也就是说，实现必须可以管理这种可能情况。

存储费用有一种轻巧细微的行为—为了奖励最小化的使用存储（与存在于所有节点上的一个很大的状态数据库直接相关），在存储区域清除一个条目的操作所需要的执行费用不仅会被豁免而且还会给予一定的回馈。实际上，这笔回馈在前端就已经被支付了，因为一个存储位置使用的成本通常都比普通消耗多。


#### 执行环境

除了系统状态$\sigma$，用于计算的gas余量g，还有一些重要的信息在执行代理必须提供的执行环境中使用，这些信息包含在元组$I$中：

* $I_a$ 拥有正在执行的代码的账户的地址。
* $I_o$ 发起当前执行的交易的发送者地址。
* $I_p$ 发起当前执行的交易的$gas$价格。
* $I_d$ 当前执行的输入数据字节数组，如果执行代理是一笔交易，就是交易数据。
* $I_s$ 触发代码执行的账户的地址；如果执行代理是一笔交易，就是交易发送者。
* $I_v$ 以**Wei**计算的价值，作执行与执行相同过程的一部分传给这个账户；如果执行代理是一笔交易，就是交易价值。
* $I_b$ 执行的虚拟机代码的字节数组。
* $I_H$ 当前区块的区块头部。
* $I_e$ 当前消息调用或者合约创建的深度（比如，当前正在执行的CALLL或者CREATE的数量）

执行模型定义了函数$\Xi$,可以计算结果状态$\sigma'$， gas余量$g'$, 累积的子状态$A$和结果输出$o$，给定这些信息的定义，对于当前上下文我们可以定义其为：
$$
(114)\qquad (\sigma', g', A, o) \equiv \Xi(\sigma, g, I)
$$
累积的子状态$A$定义为自毁集合$s$， 日志序列$l$ 和退款$r$组成的元组：$(115)\qquad A \equiv (s, l, r)$.


#### 执行概览

现在必须定义$\Xi$函数了。在大部分实际的实现中，这是被作为包括全部系统状态$\sigma$和虚拟机状态$\mu$对的一个可迭代的过程。正式情况下，用函数$X$递归定义。使用一个迭代器函数$O$（定义了状态机单个周期的结果）与判断当前状态是否是虚拟机的一个异常停机状态函数簇$Z$， 以及仅在当前状态是虚拟机正常的停机状态时设置的指令的输出数据$H$。

用（）表示的空序列不等价于空集合$\phi$，这在解释输出$H$时非常重要。当执行会继续执行时结果为$\phi$，当执行会中止时是一个序列（隐式为空序列）。
$$
(116)\qquad\qquad\qquad \Xi(\sigma, g, I) \equiv (\sigma', \mu'_g, A, o) \\
(117)\qquad (\sigma, \mu', A, ..., o) \equiv X((\sigma, \mu, A^0, I)) \\
(118)\qquad \qquad\qquad \mu_g \equiv g \\
(119)\qquad \qquad \quad \ \ \mu_{pc} \equiv 0 \\
(120)\qquad \mu_m \equiv (0, 0, ...) \\
(121)\qquad \qquad \qquad \mu_i \equiv 0 \\
(122)\qquad \qquad \quad \ \  \mu_s \equiv () \\ 
(123) \qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad\\
X((\sigma, \mu, A, I)) \equiv \begin{cases} 
(\phi, \mu, A^0, I, ()) \quad if \quad Z(\sigma, \mu, I)  \\
O(\sigma, \mu, A, I) \cdot o \quad if\quad o \ne \phi \\
X(O(\sigma, \mu, A, I)) \quad otherwise
\end{cases}
\\
其中，\\
(124)\qquad\qquad\qquad\qquad\ \   o \equiv H(\mu, I) \\
(125)\qquad (a,b,c,d) \cdot e \equiv (a,b,c,d,e)
$$
需要注意的是，当计算$\Xi$时， 我们丢弃了第4个元素$I'$并且从状态机的结果状态$\mu'$获取了gas余量$\mu'_g$。

因此$X$是循环的（此处是递归，但是实现通常倾向于使用一个简单的迭代循环）直到$Z$变成真，表示当前状态是异常的，虚拟机必须终止，而且所有改动都要丢弃；或者$H$变成一个序列（不是空集合），表示虚拟机进入了一个受控的终止状态。



#### （虚拟）机器状态

虚拟机状态$\mu$通过元组$(g, pc, m, i, s)$定义，分别表示可用的gas， 程序计数器$pc \in \mathbb{P_{256}}$， 内存内容，内存中活跃的字数（从0开始连续计算），以及栈内容。内存内容$\mu_m$是一个大小为$2^{256}$的0序列。

为了便于读取，以短大写字母书写的指令助记词必须转译成其对应的等价数字。

为了定义$Z, H$和$O$， 定义$w$为当前要被执行的操作：
$$
(126) w \equiv \begin{cases}
I_b[\mu_{pc}] \quad if \quad \mu_{pc} < \parallel I_b \parallel \\
STOP \quad otherwise
\end{cases}
$$

同时假设正在执行的指令删除和增加的栈成员的固定数量为$\delta$和$\alpha$，这两个值都可以作为指令与指令成本函数$C$的下标用于以gas为计量单位来计算全部成本。



#### 异常停机

异常停机函数$Z$定义为：
$$
(127)\qquad Z(\sigma, \mu, I) \equiv \mu_g < C(\sigma, \mu, I) \vee \\
\delta_w = \phi \vee \\
\parallel \mu_s \parallel < \delta_w \vee \\
(w \in \{JUMP, JUMPI\} \vee \\
\mu_s[0] \notin D(I_b)) \vee \\
\parallel \mu_s \parallel - \delta_w + \alpha_w > 1024
$$
如果没有足够的gas， 或者指令是无效的（因此其$\delta$下标也是未定义的）， 或者没有足够的栈元素，或者JUMP/JUMPI的目的地不存在， 或者新的栈大小超过了1024，执行就会进入一个异常停机的状态。（实际上没有命令可以通过其执行来触发一个异常停机）。




#### 跳转目的地址校验

之前用$D$作为函数来判定正在执行的给定代码的有效的跳转目的地集合。我们代码中任意被JUMPDEST指令占用的位置来定义这个函数。

所有这些位置必须在一个有效的指令范围内，而不在PUSH操作的数据部分。必须在显式定义的代码部分出现（而非隐式定义在STOP操作中），正式形式：
$$
(128)\qquad D(c) \equiv D_J(c, 0)
$$
其中，
$$
(129) \\
D_J(c,i) \equiv \begin{cases} 
\{\} \qquad \qquad\quad\qquad \qquad \ if \quad i \ge |c| \\
\{i\} \cup D_J(c, N(i, c[i])) \quad if \quad c[i] = JUMPDEST \\
D_J(c, N(i, c[i])) \qquad \quad \ \ otherwise
\end{cases}
$$
$N$是代码中下一个忽略PUSH指令的数据后，下一个指令的位置，如果任意：
$$
(130) \\
N(i, w) \equiv \begin{cases}
i + w - PUSH1 + 2 \quad if \quad w \in [PUSH1, PUSH32] \\
i+ 1 \qquad \qquad \qquad \qquad otherwise
\end{cases}
$$

#### 普通停机

正常停机函数$H$定义为：
![halt] (/resources/blockchain/halt.png)
停机操作的返回 数据RETURN有一个设定的函数$H_{RETURN}$。




### 执行周期

从序列最左侧低索引的位置向栈中添加或者删除元素，所有其他元素保持不变：
$$
(132)\qquad O((\sigma, \mu, A, I)) \equiv (\sigma', \mu', A', I) \\
(133)\qquad \qquad \qquad \qquad \quad \Delta \equiv \alpha_w - \delta_w \\
(134)\qquad \qquad \qquad \ \ \parallel \mu'_s \parallel \equiv \parallel \mu_s \parallel + \Delta \\
(135) \forall x \in [\alpha_w, \parallel \mu'_s \parallel) : \mu'_s[x] \equiv \mu_s[x + \Delta]
$$
gas根据指令的gas成本扣除，对于大部分指令来说，程序计数器在每个周期递增1，对于3种异常，带有2个指令中的某个指令作为下标的函数$J$用于计算相应的值：
$$
(136)\qquad \mu'_g \equiv \mu_g - C(\sigma, \mu, I) \\
(137)\ \mu'_{pc} \equiv \begin{cases}
J_{JUMP}(\mu) \quad if \quad w = JUMP \\
J_{JUMPI}(\mu) \quad if \quad w = JUMPI \\
N(\mu_{pc}, w) \quad otherwise
\end{cases}
$$
通常，我们假设内存，自毁集合和系统状态并不改变：
$$
(138)\qquad \mu'_m \equiv \mu_m \\
(139)\qquad \mu'_i \equiv \mu_i \\
(140)\qquad A' \equiv A \\
(141)\qquad \sigma' \equiv \sigma
$$
但是，典型的指令确实会修改其中一个或者几个值。被修改的值按照指令列表和$\alpha$和$\delta$以及gas需求的正式描述一起放在附录H中。




### 区块树到区块链

权威的区块链是一个从根节点到叶子节点贯穿整个区块树的路径。为了保持区块链的一致性，我们根据路径上完成的最大的计算量来标识区块链，也就是说那个*最重*的路径。有一个很清楚的因子来帮助我们判定路径权重，那就是叶子节点的区块号，区块号等价于路径中不计非挖矿产生的创世区块的区块数量。路径越长，为了到达叶子节点所耗费的总的已完成的挖矿工作量就越大。这个机制与已经存在的机制相同，比如比特币协议驱动的机制。

由于一个区块投部包括了难度，头部本身就已经足够可以用于校验计算是否已经完成。所有区块都向一条区块链的总计算量或者*总难度*贡献力量。

因此我们以区块$B$递归定义了总难度：
$$
(142)\qquad B_t \equiv B'_t + B_d \\
(143)\qquad \ \  B' \equiv P(B_H)
$$
对于给定的区块$B$， $B_t$是它的总难度，$B'$是其父亲区块，$B_d$是其难度。




### 区块完成

完成一个区块的过程包括4个阶段:

1. 验证（或者，如果是挖矿，就是决定）ommer；
2. 验证（或者，如果是挖矿，就是决定）交易；
3. 发放奖励；
4. 检验（或者，如果是挖矿，就是计算一个有效的）状态和nonce。


#### 叔区块验证

叔区块头部的验证是指验证每个叔区块头部既是一个有效的头部，也满足第N代成员与当前块的关系，其中$N \le 6$。叔区块头部最大为2，正式定义为：
$$
(144)\qquad \parallel B_U \parallel \le 2 \bigwedge V(U) \wedge k(U, P(B_H)_h, 6) \\
U \in B_U \quad
$$
$k$表示"is-kin"属性。
$$
(145)\\

k(U,H,n) \equiv \begin{cases} 
false \qquad \qquad \qquad \quad  if \quad n = 0 \\
s(U, H) \\
\vee k(U, p(H)_H, n - 1) \ otherwise 
\end{cases}
$$
$s$表示“is-sibling"属性。
$$
(146)\qquad s(U, H) \equiv (P(H) = P(U) \wedge H \ne U \wedge U \notin B(H)_U)
$$
$B(H)$是头部$H$相应的的区块。



#### 交易验证

给定的**gasUsed**必须与所列出的交易相对应：$B_{H_g}$表示当前块中使用的总gas数，必须和最终的交易累积使用的gas相等：
$$
(147)\qquad B_{H_g} \equiv \mathcal{l(R)_u}
$$

#### 激励兑现

对一个区块的激励会增加该区块与每个叔区块受益者地址对应的账户的余额。假设受益账号为$R_b$，对于每个叔区块，我们增加一个占当前区块奖励$\frac{1}{32}$的额外奖励，并且叔区块的受益人获取的激励依赖于区块号。正式地定义为函数$\Omega$:
$$
(148)\qquad \qquad \Omega(B, \sigma) \equiv \sigma' : \sigma' = \sigma \quad except: \\
(149)\qquad \sigma'[B_{H_c}]_b = \sigma[B_{H_c}]_b + (1 + \frac{\parallel B_U \parallel}{32})R_b \\
(150)\ \forall_{U \in B_U}: \\
\sigma'[U_c]_b = \sigma[U_c]_b + (1 + \frac{1}{8}(U_i - B_{H_i}))R_b
$$
如果在叔区块和区块之间受益人地址有冲突（比如， 两个叔区块具有相同的受益人地址或者一个叔区块与当前区块具有相同的受益人地址），就需要额外的操作。

区块的激励定义为5个以太币：
$$
(151)\qquad R_b= 5 \times 10^{18}
$$

#### 状态与Nonce验证

现在可以定义函数$\Gamma$，映射一个区块到其初始状态：
$$
(152) \\
\Gamma(B) \equiv \begin{cases} 
\sigma_0 \qquad \qquad \qquad \qquad \qquad \qquad \quad  if \quad P(B_H) = \phi \\
\sigma_i : TRIE(L_S(\sigma_i)) = P(B_H)_{H_r} \quad otherwise
\end{cases}
$$
这里$TRIE(L_S(\sigma_i))$表示字典树的根节点的状态$\sigma_i$的散列；假设协议的实现会将其细致高效的存储在状态数据库中，因为字典树本质上就是一个不可变的数据结构。

最后定义区块转移函数$\Phi$，映射一个尚未完成的区块$B$到一个完成的区块$B'$：
$$
(153)\qquad \Phi(B) \equiv B': B' = B^* \quad except: \\
(154)\qquad \qquad\qquad\quad\  B'_n = n : x \le \frac{2^{256}}{H_d} \\
(155)\ B'_m = m \quad with (x, m) = PoW(B^{*}_{n'}, n, d) \\
(156)\ B^* \equiv B \quad except: \quad B^*_r = r(\Pi(\Gamma(B), B))
$$
正如开头出设定的，$\Pi$是以区块完成函数$\Omega$和交易计算函数$\Upsilon$定义的状态转移函数。

$R[n]_\sigma, R[n]_l$和$R[n]_u$是每一笔交易之后第n个响应的状态，日志以及累积使用的的gas量（$R[n]_b$是元组中的第四个元素，已经以日志的形式定义了）。前者简单的定义为对前一个交易的状态（对于首笔交易来说是区块的初始状态）应用相关的交易后得到的状态：
$$
(157)\qquad R[n]_\sigma = \begin{cases}
\Gamma(B) \qquad\qquad \qquad \quad \ \  if \quad n < 0 \\
\Upsilon(R[n-1]_\sigma, B_T[n]) \quad otherwise
\end{cases}
$$
对于$B_R[n]_u$，我们用相似的方法将每个元素定义为在计算响应的交易时使用的gas与前一个元素的和（如果它是第一个，就是0）：
$$
(158)\qquad R[n]_u = \begin{cases}
0 \qquad \qquad \qquad \qquad if \quad n < 0 \\
\Upsilon^g(R[n-1]_\sigma, B_T[n]) \\
\quad + R[n-1]_u \qquad otherwise
\end{cases}
$$
对于$R[n]_l$，利用在交易执行函数中定义的函数$\Upsilon^l$来定义：
$$
(159) \qquad R[n]_l = \Upsilon^l(R[n-1]_\sigma, B_T[n])
$$
最后定义函数$\Pi$作为作用于最终交易状态$\mathcal{l}(B_R)_\sigma$的区块激励函数$\Omega$的新状态：
$$
(160)\qquad \Pi(\sigma, B) \equiv \Omega(B, \mathcal{l}(R)_\sigma)
$$
完整的区块转移机制，工作量证明函数$PoW$就定义好了。



#### 工作量证明挖矿

工作量证明（PoW）的挖掘是以一个加密的安全随机数存在的，证明已经花费了一定的计算量获得了某个令牌值$n$这样有意的猜测。通过对难度（通过扩展的总难度）加入均值和信用，PoW被用于增强区块链的安全性。不止如此，由于挖掘新的区块会获得一笔奖励，工作量证明不止是一种保证区块链在将来仍然权威的安全可信的方法函数，而且也是一种财富分配机制。

由于这两个原因，工作量证明函数有两个重要的目标：

首先，必须可以被所有人访问。对用于获得奖励的特殊的、非通用的硬件的需求必须最小化。这使得分发模型尽可能的开放，并且理想情况下，使得全世界的任何人用电力交换以太币的挖矿活动可以以大致相同的速度进行。

其次，必须不能创造超线性利润，尤其不能带有很高的初始 门槛。这样的机制使得一个装备精良的节点可以获得一个网络中全部挖矿能中能够造成麻烦的计算力，并获得一个超线性的奖励（因此有利于他们控制分配）从而降低网络的安全性。

在比特币世界中有一个麻烦就是**ASIC**。这些定制的计算硬件存在的目的就是做一个简单的任务。在比特币的情况中，这个任务就是SHA256散列函数。当**ASIC**被用于工作量证明函数，这两个目的就都处于危险之中了。正因如此，具备抗**ASIC**能力的工作量证明函数（比如难度或者经济上无效的特殊计算硬件的实现）被认为是工人的银弹。

对于抗**ASIC**能力有两个方向：

第一个方向是使其内存串行化，比如使获得随机数的函数需要大量的内存和带宽，内存不能同时并行的用于发现多个随机数。

第二个是使计算类型实现通用目的。对一个通用目的任务集合而言，“定制化的硬件”本质上是通用目的硬件，比如商用桌面计算机就很接近与这类任务的“定制化的硬件”。以太坊1.0采用了第一种方案。

工作量证明函数更正式的形式为$PoW$:
$$
(161)\qquad m = H_m \wedge n \le \frac{2^256}{H_d} \quad with \quad (m,n) = PoW(H_{n'}, H_n, d)
$$
$H_{n'}$是新的区块头部，不过缺少了随机数和混合散列分量；

$H_n$是头部的随机数；$d$是一个计算混合散列需要的巨型数据集；$H_d$是新的区块的难度值（比如区块难度从段10开始）。$PoW$是用于计算出一个首元素为混合散列mixHash，第二个元素为一个加密依赖于$H$和**d**的伪随机数的数组的工作量证明函数。算法为$Ethash$.

