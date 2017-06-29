# 以太坊黄皮书笔记（四）

## GAS和报酬

为了避免滥用网络和图灵完备性带来的固有问题[^3] 以太坊中所有的可编程计算都与费用挂钩，费用以*gas*为单位。因此任何给定的可编程计算的片段（包括合约创建，消息调用，访问和使用账号存储，以及在虚拟机上执行操作）都有一个达成共识的以*gas*计算的成本。

每笔交易都有一个与之相关的特别的*gas*数量：**gasLimit**，是隐式从发送者账户余额购买的。购买行为以交易中指定的*gasPrice*进行。如果账户余额不足以支撑购买需要消耗的*gas*，交易将会被当作是无效交易。之所以被称为*gasLimit*是由于在交易结束时任何尚未使用的gas将会被退还给发送者的账户中（与购买时的汇兑相同）。**Gas**在交易执行之外是无法存在的，因此对于具有可信的相关代码的账户，可能会被单独设定并保留一个相对较高的*gasLimit*。

通俗的讲，用于未被退回的用于购买*gas*的以太币将会被发送给*受益人*地址，账户地址在矿工的控制之下是一种典型情况。交易者可以随意指定其想要设定的任何*gasPrice*，但是矿工可以选择忽略该笔交易。一个具有更高*gas*价格的交易将会耗费发送者更多以太币并向矿工支付更高的价值，因此更容易被矿工选中。矿工通常会公布其执行交易的最低*gas*价格，交易者可以在决定提供什么样的交易价格时忽略这些公布的价格。由于存在一个（带权重的）最低可接受的*gas*价格区间，交易者需要在更低的*gas*价格和最大化交易被矿工快速选中之间进行权衡。

## 交易的执行

交易的执行是以太坊协议中最复杂的部分：这部分定义了状态转移函数$$\Upsilon$$。假设任何被执行的交易首先已经通过了基本的有效性测试，包括：

1. 交易是结构完好的RLP，没有额外的尾字节；
2. 交易签名是有效的；
3. 交易的*nonce*是有效的（等于发送者账户的当前nonce）；
4. 交易使用的gas限制不比基本gas数$$g_0$$小；
5. 发送者的账户余额包含预先支付的最低成本$$v_0$$。

考虑函数$$\Upsilon$$，交易$$T$$和状态$$\sigma$$，其正式的形式如下：
$$
(57)\qquad \sigma' = \Upsilon(\sigma, T)
$$
$$\sigma'$$就是交易后的状态。我们还定义了$$\Upsilon^g$$用于估算交易执行需要的gas数量，$$\Upsilon^l$$用于评估交易的累积日志元素。

### 子状态

在交易执行过程中，一直在累积紧随交易之后立刻操作的信息，这些信息称为*交易子状态*，用元组$$A$$表示:
$$
(58)\qquad A \equiv (A_s, A_l, A_r)
$$
元组内容包括自销毁的集合$$A_s$$：一个在交易完成之后丢弃的账号集合。$$A_l$$是日志序列：在VM代码执行时一系列归档并可索引的“检查点”，这样合约调用就可以很容易的被以太坊之外（比如去中心化的应用前端）的旁观者追踪。最后是退款余额$$A_r$$，该值通过使用SSTORE指令将合约存储从非零值重置为0来增加。虽然不是立即退款，也仍然允许部分抵扣总的执行成本。

简而言之，空的非自毁的、无日志、0退款余额的子状态$$A^0$$定义如下：
$$
A^0 \equiv (\phi， ()， 0)
$$

### 执行

定义当前交易在执行之前需要支付的基本gas数量 $$g_0$$如下：
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
$$T_i, T_d$$表示根据交易是合约创建还是消息调用，与交易关联的数据和初始化EVM代码的字节序列。

$$G_{txcreate}$$在交易是合约创建时增加，但如果是EVM-code的结果则不添加。$$G$$的完整定义在附录G中。

预支成本$$v_0$$通过下面的公式计算:
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
注意最后一个条件，交易的gas限制$$T_g$$与当前区块之前使用的gas量$$l(B_R)_u$$之和必须不大于当前区块的**gasLimit**。

一笔有效的交易的执行以一个不可撤销的状态变化开始：发送者账号的nonce值$$S(T)$$加一，同时余额减去预支的成本$$T_gT_p$$。可用的用于处理计算的gas值$$g$$定义为$$T_g - g_0$$。不论是合约创建还是消息调用的计算都会产生一个最终状态（可能逻辑上与当前状态等价），对其改变是确定的，并且不会无效：从当前点开始不可能有无效的交易。

检查点的状态$$\sigma_0$$定义为：
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
$$g$$是在扣除交易存在性需要支付的基本gas后的剩余gas：

$$(69)\qquad g \equiv T_g - g_0$$

$$T_o$$是原始交易者，可能与通过EVM代码而非交易直接触发的消息调用或者合约创建的发送者不同。

$$\Theta_3$$ 表示只有函数值的前三个分量会被取用；消息调用的输出价值（一个字节数组）的最终表示，并且在交易计算的上下文中是不使用的。

消息调用或者合约创建被处理之后，最终状态由以原始速率退给发送者的退款数量$$g^*$$决定，其值等于剩余的gas量$$g'$$加上一些退款计数器的补贴：
$$
(70)\qquad g^* \equiv g' + min\{\lfloor \frac{T_g - g'}{2} \rfloor, A_r\}
$$
可退还的总数是剩余gas量$$g'$$加上$$A_r$$， 第二部分为$$A_r$$与已使用的总量$$T_g - g'$$的一半（下除法）的最小值。

gas需要的以太币是给予矿工地址的，矿工的地址被设定为当前区块$$B$$的受益地址。所以我们依据状态$$\sigma$$来定义预终状态$$\sigma^*$$：
$$
(71)\qquad \qquad\quad \sigma^* \equiv \sigma_P \qquad except \\
(72) \ \sigma^*[S(T)]_b \equiv \sigma_P[S(T)]_b + g^*T_p \\
(73) \ \sigma^*[m]_b \equiv \sigma_P[m]_b + (T_g - g^*)T_p \\
(74) \qquad\qquad\qquad\qquad\quad\ \ \   m \equiv B_{Hc}
$$
在删除自毁集合中的所有账号之后达到最终状态$$\sigma'$$：
$$
(75)\qquad\quad \sigma' \equiv \sigma^* \quad except \\
(76)\qquad \forall i \in A_s : \sigma'[i] \equiv \phi
$$
最后设定在这笔交易中使用的gas总数$$\Upsilon^g$$和当前交易创建的日志$$\Upsilon^ l$$：
$$
(77)\qquad \Upsilon^g(\sigma, T) \equiv T_g - g' \\
(78)\qquad\qquad \Upsilon^l(\sigma, T) \equiv A_l
$$

## 合约的创建

在创建一个账户时有很多基本参数：发送者(s)， 原始交易者（o）， 可用的gas（$g$）， gas价格（$$p$$)， 捐款（$$v$$)， 以及一个任意长度的字节数组**i**，EVM初始化代码和消息调用/合约创建的当前栈（$$e$$）深度。创建函数正式定义为$$\Lambda$$, 用于从这些值和状态$$\sigma$$来计算出一个包含新的状态，剩余gas和累积交易子状态的元组（$$\sigma', g', A$$）：
$$
(79)\qquad (\sigma', g', A) \equiv \Lambda(\sigma, s, o, g, p, v, \mathcal{i}, e)
$$
新账户的地址$$a$$定义为包含发送者和nonce的结构的RLP编码的Keccak 散列值的高位160比特：
$$
(80)\qquad a \equiv \mathcal{B}_{96..255}(KEC(RLP((s, \sigma[s]_n - 1))))
$$
其中，**$KEC$**是Keccak 256位散列函数，**RLP**是**RLP**编码函数，$$\mathcal{B}_{a..b}(X)$$计算出包含二进制数据**X**在范围[a,b]内的二进制值，$$\sigma[x]$$是$$x$$或者$$\phi$$（若不存在）的地址状态。注意我们使用的nocne比发送者的nonce小1；需要判断在该调用之前已经为发送者账户增加了nonce，因此使用的是发送者在可响应的交易或者VM操作开始时的nonce值。

账户的nonce初始值为0， 余额为传入的值，存储为空，代码散列值为空字符串的Keccak 256位散列。发送者的余额也是通过传入的值扣除，因此可变的状态变成了$$\sigma^*$$:
$$
(81) \qquad \qquad\qquad\qquad\quad \sigma^* \equiv \sigma \quad except: \\
(82) \ \sigma^*[a] \equiv (0, v+ v', TRIE(\phi), KEC()) \\
(83) \qquad\qquad\qquad\qquad\ \ \  \sigma^*[s]_b \equiv \sigma[s]_b - v
$$
$$v'$$是账户之前已存在的价值，在当前事件中是之前存在的：
$$
(84)\qquad v' \equiv \begin{cases}
0 \quad if \quad \sigma[a] = \phi \\
\sigma[a]_b \quad otherwise
\end{cases}
$$
最后，根据执行模型执行EVM初始化代码**i**对账户完成初始化。代码执行可以影响很多并非执行状态内部的事件：账户的存储可能被改变，新的账户可以被创建以及发起新的消息调用。因此，代码的执行函数$$\Xi$$计算出一个包括结果状态$$\sigma^{**}$$， 可用的gas余额$$g^{**}$$，累积的子状态$A$和账户的实体代码$$o$$。
$$
(85)\quad (\sigma^{**}, g^{**}, A, o) \equiv \Xi(\sigma^*,  g, I)
$$
其中$$I$$包含了定义在*执行模型*中的执行环境参数：
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
$$I_d$$计算的结果是一个空元组，因为对于这个调用没有输入数据。$$I_H$$没有特殊处理由区块链决定。

代码执行会耗尽gas，并且gas不能降到0以下，因此执行可能在代码达到一个自然的停机状态之前就退出。在这种（和其他的）异常情况中，我们说出现了一个gas耗尽（out-of-gas, OOG)异常：计算出来的状态定义为空集$$\phi$$， 并且这个创建操作对状态没有任何影响，只保留以试图创建合约时的即刻状态。

如果初始化代码成功的完成，需要支付最终的合约创建开销，即代码保证金$$c$$，其值与创建的合约代码大小成比例：
$$
(94) \quad c \equiv G_{codedeposit} \times |o|
$$
如果缺乏足够的gas余额来支付这笔开销，比如$$g^{**} < c$$， 那么也会抛出一个*out-of-gas*异常。

在任何出现此类异常的情况下gas余额都会使0，例如，如果合约的创建是由一笔交易的接受主导的，那么这就不会影响合约创建的固有成本的支付，不管怎样都必须支付。但是当出现out-of-gas时，交易的价值并没有转移到异常退出的合约地址上。

如果没有出现这个异常，剩余的gas会被退还给发起人并且当前修改过的状态可以持久化了。因此正式形式，可以将结果状态、gas和子状态设定为$$(\sigma', g', A)$$, 其中：
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
$$\sigma'$$的决定条件中的异常$$o$$是初始化代码执行产生的字节序列设定了新创建的账户最终的主体代码。

需要注意的是，结果要么成功的创建带有额度的合约，要么不存在合约与价值转移。



#### 微妙之处

注意，当初始化代码正在执行时，新创建的地址是存在的只不过没有基本的主体代码。因此在这段时间内该地址收到的任何消息调用都会导致没有代码执行。如果初始化执行以**SELFDESTRUCT**指令结束是没有意义的，因为在交易完成之前这个账户就会被删除。对于一个正常的**STOP**代码，或者代码返回的是一个空集，就会以一个僵尸账户的状态保留，并且所有剩余的余额都会被永久的锁定在该账户中。

