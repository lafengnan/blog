# 以太坊黄皮书笔记（二）

## 区块链



整体上*以太坊*可以看作是一个基于*交易*的状态机：从创世状态开始*不断*执行交易，将状态转换成某个*最终状态*。这个*最终状态*在以太坊世界作为一个权威的”版本"而被接受。状态可以包含这些信息：

- 账户余额(Balances)
- 账户信誉(Reputations)
- 托管（Trust arrangements)
- 与物理世界信息的数据关联

简单来说，就是当前可以用计算机表示的任何数据都是可以接受的。

**交易**是两个状态之间的一个**有效**的弧，这里强调**有效**很重要，因为无效的状态变化远比有效的状态变化多的多。无效的状态变化可能是：单方减少账户余额却不存在对等的余额增加。

**有效**的状态变化来自交易的触发，正式的形式：
$$
（1)\qquad \sigma_{t+1} \equiv \Upsilon(\sigma, T)
$$
$$\Upsilon$$ 是以太坊的状态转移函数，在以太坊中$$\Upsilon$$与$$\sigma$$一起配合比任何已有的对比系统都强大。

$$\Upsilon$$允许参数执行任意计算，同时$$\sigma$$则允许参数在交易之间存储任意的状态。

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

$$\Omega$$是区块最终的状态转移函数（向指定节点提供奖励的函数）；

B是当前区块，其中包括存在于其他模块中的一系列交易；

$$\Pi$$是区块级的状态转移函数

这些是区块链模式的基础，不仅是以太坊的基石，也是迄今为止其他所有去中心化的一致性交易系统的基石。

以太坊区块链基本结构图：

![区块链基本结构](/resources/blockchain/pow_chain.png)

## 价值

以太坊的货币称为以太币，**ETH**，有时也以古英语字符$D$表示。以太币的最小面值是**Wei**， 1 ETH等于$$10^{18}$$ Wei， 除了Wei还有其他的面值：

| 乘数          | 名称     |
| ----------- | ------ |
| $$10^0$$    | Wei    |
| $$10^{12}$$ | Szabo  |
| $$10^{15}$$ | Finney |
| $$10^{18}$$ | Ether  |

当前以太坊中任何涉及*价值*的上下文中，包括余额、现金或者付款都必须以*Wei*来计算。

## 约定

2个顶级结构用粗体希腊字母表示：全局状态**$$\sigma$$**和虚拟机状态**$$\mu$$**。

操作高级结构的函数用大写的希腊字母表示，例如以太坊状态转移函数$$\Upsilon$$.

其他大部分函数用大写字母表示，例如通用成本函数$$C$$。这些函数可能用下标表示特定的变体函数，比如$$SSTORE$$操作的成本函数$$C_{SSTORE}$$。对于特殊都和和外部定义的函数，以打印排版格式书写，比如Keccak-256散列函数（作为胜利者战胜了SHA-3）表示为$KEC$（常常作为Keccak的简写使用），当然，$KEC512$就表示Keccak 512 散列。

元组用一个大写字母表示，比如$$T$$表示一笔以太坊交易。这个标记可能会根据需要以下标去表示一个单独的分量，比如$$T_n$$表示当前这笔交易的随机数。下标的形式用于表示其类型，比如大写字母的下标表示特定下标分量的元组。

标量和固定长度的字节序列（或者同义词数组）用给一个普通的小写字母表示，比如$n$用于表示一个交易随机数（nonce）。那些有特殊含义的值用希腊字母表示，比如$$\delta$$表示给定操作在栈上需要的元素数量。

任意长度的序列用一个粗体小写字母表示，比如**o**表示消息调用输出数据的字节序列。对于特定的重要值，可能会使用一个粗体大写字母。

我们假定标量都是正整数，因此他们都属于集合$$\mathbb{P}$$。所有的字节序列集合$$\mathbb{B}$$在下面定义。如果这样的序列集合被限定了长度，则用下标表示。因此，所有长度为32的字节序列被命名为$$\mathbb{B}_{32}$$，所有小于$$2^{256}$$的正整数命名为$$\mathbb{P}_{256}$$。正式定义在下文给出。

中括号用于索引或者引用独立的分量或者一串序列的子串，比如$$\mu_s[0]$$表示虚拟机栈中的第一个元素。对于子序列，点号用于指定范围，包括左右限制元素，比如$$\mu_m[0..31]$$表示虚拟机内存中的钱32个元素。

对于表示一个账户序列的全局状态$$\sigma$$，中括号用于引用一个单独的账户。

对于已存在的值的变体，遵循定义的给定范围的规则，加入用一个占位符$$\square$$表示未修改的输入值，那么被修改过的可用值就用$$\square'$$表示，中间值用$$\square^*, \square^{**}$$表示。对于特别情况，为了最大化可读性，只有在含义不清楚的时候，采用阿尔法数字作为中间值的下标，尤其是那些特殊的情况。

对于已有的函数使用，给定函数$f$，用$$f^*$$表示一个相似的，元素级的函数映射在序列之间代替。

贯穿全文定义了很多有用的函数，其中最常见的是$$\mathcal{l}$$，用于计算给定序列中的最后一个元素：
$$
(5)\qquad \mathcal{l}(x) \equiv x[\parallel x \parallel - 1]
$$

## 术语

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



## 递归表达式

这里定义了一系列用于编码结构化数据（字节数组）的方法。可用的结构$$\mathbb{T}$$定义如下：
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

- 如果字节数组只包含一个单独的字节，并且这个字节小于128字节，那么输入和输出严格相等。
- 如果字节数组少于56个字节，那么输出等于输入序列加上前缀后长度等于数组长度加上128 的字节的序列
- 否则，输出等于在输入加上在翻译为一个大端整数时等于输入字节数组长度的最短字节数组前缀，然后在通过该字节数组的长度加上183作为最前面的前缀。

正式的定义为$$R_b$$:
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
因此，$$BE$$就是将一个正整数值扩展成一个具有最小长度的大端字节数组的函数，点号操作用于连接序列。

如果准备序列化的值不是一个字节数组，而是一个其他元素的序列，那么RLP序列化采用下面两种形式中的一种：

- 如果包含的元素拼接后的序列长度小于56字节，那么其输出等于拼接后的序列加上一个长度等于该字节数组的字节加上192
- 否则，输出等于凭借后的序列加上翻译为大端整数时长度等于拼接后的字节数组长度的前缀，然后通过该字节数的长度加上247 作为最前面的前缀

正式的定义为$$R_l$$:
$$
(169)\qquad R_l(X) \equiv \begin{cases}
(192 + \parallel s(X) \parallel) \cdot s(X) \qquad \qquad \qquad \qquad \qquad \quad \ if \quad \parallel s(X) \parallel < 56 \\
(247 + \parallel BE( \parallel s(X) \parallel ) \parallel ) \cdot BE(\parallel s(X) \parallel ) \cdot s(X) \quad otherwise
\end{cases}
\\
(170)\qquad \qquad \qquad \qquad \qquad \qquad \qquad \qquad \qquad \qquad \qquad s(X) \equiv RLP(X_0) \cdot RLP(X_1)...
$$
如果RLP用于对标量编码，之定义为一个正整数（$$\mathbb{P} $$ 或者任意$$x \in \mathbb{P}_x$$），必须设定为其大端表达式与其相等最短的字节数组。因此对正整数$$i$$的RLP定义为：
$$
(171)\qquad RLP(i: i \in \mathbb{P}) \equiv RLP(BE(i))
$$
当翻译RLP数据是，如果一个期望的片段被解码成一个标量，并且在字节序列中首部为0，客户端需要认为这不是权威的数据，并以对待无效RLP数据的方式——完全忽略，来处理。

