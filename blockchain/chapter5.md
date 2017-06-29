# 以太坊黄皮书笔记（五）

## 消息调用

在执行消息调用的情况下需要这些参数：发送者（s），交易创建者（o），接收者（r），执行代码的账户（c，通常与接收者相同），可用的gas（g），价值（v）和gas价格（p）以及任意长度的输入数据字节数组（d）和消息调用/合约创建的当前栈深度（e）。

除了计算新的状态和交易的子状态，消息调用还有一个额外的分量—用字节数组$o$表示的输出数据。这个数据在执行交易时是被忽略的，但是消息调用可以通过执行VM代码初始化，在这种情况下这个数据就要被使用了。
$$
(97)\qquad (\sigma', g', A, o) \equiv \Theta(\sigma, s, o, r, c, g, p, v, \tilde{v}, d, e)
$$
**注意**，需要区分传输的数据$$v$$和执行上下文中出现的用于**DELEGATECALL**的价值$$\tilde{v}$$。

第一个转移状态$$\sigma_1$$作为原始状态，但是带有从发送者转移给接收者的价值(除非$$s = r$$)：
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
账户的关联代码（标记为Keccak散列为$$\sigma[c]_c$$的片段）依照执行模型（后文）执行。只是与合约创建相同，如果执行异常终止（比如，因为耗尽了gas，栈下溢，无效的跳转目的地或者无效的指令），那么不就会向调用者退回gas，且状态会被撤回到在余额转移（比如，$\sigma$）之前的时间点。
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
假设客户端会在某个时间点预先将$$KEC((I_b), I_b)$$对存储下来，以让条件$$I_b$$可用。

正如我们所见，通用执行框架$$\Xi$$在计算消息调用时有**4**个异常：这是4个所谓的“预编译”合约，表示随后可能成为*原生扩展*的架构的初始部分。这四个合约在地址$$1,2,3, 4$$执行椭圆曲线公钥恢复函数，SHA2 256位散列算法，RIPEMD 160位散列以及对应的身份函数。

## 执行模型

执行模型表述了给定一系列字节码指令和一个很小的环境数据元组，系统状态是被如何改变的。这是通过一个正式的称为以太坊虚拟机（EVM）的虚拟状态机模型规范的。EVM是一个准图灵完备的机器；这里用“准”修订词是因为计算与$gas$参数固有的绑定造成的，因为这限制了可完成的总计算量。

### 基础

**EMV**是一个简单的基于栈的体系结构。虚拟机的字长（也就是栈成员的大小）是256比特。选择这个长度是为了方便使用Keccak-256散列算法和椭圆曲线计算。其内存模型是一个简单的字寻址字节数组，栈的最大长度是1024。虚拟机还有一个独立的存储模型，与内存类似但不是一个字节数组(byte array)，而是一个字寻址的字数组(word array)。与内存不同， 存储区域不是瞬变的，而且是作为系统状态的一部分维护的。所有的存储和内存区域都被初始化为0.

虚拟机并不遵循标准的冯诺依曼体系结构，程序代码并不是存储在通用可访问的内存或者存储区域中的，而是存储在一个隔离的可通过一个特殊指令交互的虚拟**ROM**中。

虚拟机可能因为一些原因异常执行，包括栈下溢和无效的指令。和*out-of-gas*异常类似，这些异常并不能使状态保持完好，实际上，虚拟机会立刻停机并向会单独处理异常的异常代理（交易处理器或者递归创建的执行环境）报告。

### 费用概览

费用（以gas为单位结算）在3种不同的情况下会被扣除，这三种情况都是一个操作能够执行的先决条件。

首先，最常见的是操作计算的固有费用。

其次，gas可能会被用于支付一个次级消息调用或者合约创建，这构成了用于CREATE, CALL和CALLCODE的报酬。

最后，随着内存使用的增加而支付gas。

一个账户的整个执行中，内存消耗的可支付总费用是与包含所有内存偏移（无论读还是写）的32字节的最小倍数成正比的。这是在“及时”基础上的一种支付。正因如此，引用一段至少比前面索引的内存大32字节的内存都会产生一笔额外的使用费。因为这笔费用，非常不推荐寻址超过32比特上限。也就是说，实现必须可以管理这种可能情况。

存储费用有一种轻巧细微的行为—为了奖励最小化的使用存储（与存在于所有节点上的一个很大的状态数据库直接相关），在存储区域清除一个条目的操作所需要的执行费用不仅会被豁免而且还会给予一定的回馈。实际上，这笔回馈在前端就已经被支付了，因为一个存储位置使用的成本通常都比普通消耗多。

### 执行环境

除了系统状态$$\sigma$$，用于计算的gas余量g，还有一些重要的信息在执行代理必须提供的执行环境中使用，这些信息包含在元组$$I$$中：

- $$I_a$$ 拥有正在执行的代码的账户的地址。
- $$I_o$$ 发起当前执行的交易的发送者地址。
- $$I_p$$ 发起当前执行的交易的$gas$价格。
- $$I_d$$ 当前执行的输入数据字节数组，如果执行代理是一笔交易，就是交易数据。
- $$I_s$$ 触发代码执行的账户的地址；如果执行代理是一笔交易，就是交易发送者。
- $$I_v$$ 以**Wei**计算的价值，作执行与执行相同过程的一部分传给这个账户；如果执行代理是一笔交易，就是交易价值。
- $$I_b$$ 执行的虚拟机代码的字节数组。
- $$I_H$$ 当前区块的区块头部。
- $$I_e$$ 当前消息调用或者合约创建的深度（比如，当前正在执行的CALLL或者CREATE的数量）

执行模型定义了函数$$\Xi$$,可以计算结果状态$$\sigma'$$， gas余量$$g'$$, 累积的子状态$$A$$和结果输出$$o$$，给定这些信息的定义，对于当前上下文我们可以定义其为：
$$
(114)\qquad (\sigma', g', A, o) \equiv \Xi(\sigma, g, I)
$$
累积的子状态$A$定义为自毁集合$s$， 日志序列$l$ 和退款$r$组成的元组：$$(115)\qquad A \equiv (s, l, r)$$.

### 执行概览

现在必须定义$$\Xi$$函数了。在大部分实际的实现中，这是被作为包括全部系统状态$$\sigma$$和虚拟机状态$$\mu$$对的一个可迭代的过程。正式情况下，用函数$$X$$递归定义。使用一个迭代器函数$$O$$（定义了状态机单个周期的结果）与判断当前状态是否是虚拟机的一个异常停机状态函数簇$$Z$$， 以及仅在当前状态是虚拟机正常的停机状态时设置的指令的输出数据$$H$$。

用（）表示的空序列不等价于空集合$$\phi$$，这在解释输出$$H$$时非常重要。当执行会继续执行时结果为$$\phi$$，当执行会中止时是一个序列（隐式为空序列）。
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
需要注意的是，当计算$$\Xi$$时， 我们丢弃了第4个元素$$I'$$并且从状态机的结果状态$$\mu'$$获取了gas余量$$\mu'_g$$。

因此$$X$$是循环的（此处是递归，但是实现通常倾向于使用一个简单的迭代循环）直到$$Z$$变成真，表示当前状态是异常的，虚拟机必须终止，而且所有改动都要丢弃；或者$H$变成一个序列（不是空集合），表示虚拟机进入了一个受控的终止状态。

### （虚拟）机器状态

虚拟机状态$$\mu$$通过元组$$(g, pc, m, i, s)$$定义，分别表示可用的gas， 程序计数器$$pc \in \mathbb{P_{256}}$$， 内存内容，内存中活跃的字数（从0开始连续计算），以及栈内容。内存内容$$\mu_m$$是一个大小为$$2^{256}$$的0序列。

为了便于读取，以短大写字母书写的指令助记词必须转译成其对应的等价数字。

为了定义$Z, H$和$O$， 定义$w$为当前要被执行的操作：
$$
(126) w \equiv \begin{cases}
I_b[\mu_{pc}] \quad if \quad \mu_{pc} < \parallel I_b \parallel \\
STOP \quad otherwise
\end{cases}
$$
同时假设正在执行的指令删除和增加的栈成员的固定数量为$$\delta$$和$$\alpha$$，这两个值都可以作为指令与指令成本函数$$C$$的下标用于以gas为计量单位来计算全部成本。



#### 异常停机

异常停机函数$$Z$$定义为：
$$
(127)\qquad Z(\sigma, \mu, I) \equiv \mu_g < C(\sigma, \mu, I) \vee \\
\delta_w = \phi \vee \\
\parallel \mu_s \parallel < \delta_w \vee \\
(w \in \{JUMP, JUMPI\} \vee \\
\mu_s[0] \notin D(I_b)) \vee \\
\parallel \mu_s \parallel - \delta_w + \alpha_w > 1024
$$
如果没有足够的gas， 或者指令是无效的（因此其$$\delta$$下标也是未定义的）， 或者没有足够的栈元素，或者JUMP/JUMPI的目的地不存在， 或者新的栈大小超过了1024，执行就会进入一个异常停机的状态。（实际上没有命令可以通过其执行来触发一个异常停机）。



#### 跳转目的地址校验

之前用$$D$$作为函数来判定正在执行的给定代码的有效的跳转目的地集合。我们代码中任意被JUMPDEST指令占用的位置来定义这个函数。

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
$$N$$是代码中下一个忽略PUSH指令的数据后，下一个指令的位置，如果任意：
$$
(130) \\
N(i, w) \equiv \begin{cases}
i + w - PUSH1 + 2 \quad if \quad w \in [PUSH1, PUSH32] \\
i+ 1 \qquad \qquad \qquad \qquad otherwise
\end{cases}
$$

#### 普通停机

正常停机函数$$H$$定义为：
![halt](/resources/blockchain/halt.png)
停机操作的返回 数据RETURN有一个设定的函数$$H_{RETURN}$$。

### 执行周期

从序列最左侧低索引的位置向栈中添加或者删除元素，所有其他元素保持不变：
$$
(132)\qquad O((\sigma, \mu, A, I)) \equiv (\sigma', \mu', A', I) \\
(133)\qquad \qquad \qquad \qquad \quad \Delta \equiv \alpha_w - \delta_w \\
(134)\qquad \qquad \qquad \ \ \parallel \mu'_s \parallel \equiv \parallel \mu_s \parallel + \Delta \\
(135) \forall x \in [\alpha_w, \parallel \mu'_s \parallel) : \mu'_s[x] \equiv \mu_s[x + \Delta]
$$
gas根据指令的gas成本扣除，对于大部分指令来说，程序计数器在每个周期递增1，对于3种异常，带有2个指令中的某个指令作为下标的函数$$J$$用于计算相应的值：
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
但是，典型的指令确实会修改其中一个或者几个值。被修改的值按照指令列表和$$\alpha$$和$$\delta$$以及gas需求的正式描述一起放在附录H中。



### 区块树到区块链

权威的区块链是一个从根节点到叶子节点贯穿整个区块树的路径。为了保持区块链的一致性，我们根据路径上完成的最大的计算量来标识区块链，也就是说那个*最重*的路径。有一个很清楚的因子来帮助我们判定路径权重，那就是叶子节点的区块号，区块号等价于路径中不计非挖矿产生的创世区块的区块数量。路径越长，为了到达叶子节点所耗费的总的已完成的挖矿工作量就越大。这个机制与已经存在的机制相同，比如比特币协议驱动的机制。

由于一个区块投部包括了难度，头部本身就已经足够可以用于校验计算是否已经完成。所有区块都向一条区块链的总计算量或者*总难度*贡献力量。

因此我们以区块$B$递归定义了总难度：
$$
(142)\qquad B_t \equiv B'_t + B_d \\
(143)\qquad \ \  B' \equiv P(B_H)
$$
对于给定的区块$B$， $$B_t$$是它的总难度，$$B'$$是其父亲区块，$$B_d$$是其难度。



### 区块完成

完成一个区块的过程包括4个阶段:

1. 验证（或者，如果是挖矿，就是决定）ommer；
2. 验证（或者，如果是挖矿，就是决定）交易；
3. 发放奖励；
4. 检验（或者，如果是挖矿，就是计算一个有效的）状态和nonce。

#### 叔区块验证

叔区块头部的验证是指验证每个叔区块头部既是一个有效的头部，也满足第N代成员与当前块的关系，其中$$N \le 6$$。叔区块头部最大为2，正式定义为：
$$
(144)\qquad \parallel B_U \parallel \le 2 \bigwedge V(U) \wedge k(U, P(B_H)_h, 6) \\
U \in B_U \quad
$$
$$k$$表示"is-kin"属性。
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
$$B(H)$$是头部$H$相应的的区块。



#### 交易验证

给定的**gasUsed**必须与所列出的交易相对应：$$B_{H_g}$$表示当前块中使用的总gas数，必须和最终的交易累积使用的gas相等：
$$
(147)\qquad B_{H_g} \equiv \mathcal{l(R)_u}
$$

#### 激励兑现

对一个区块的激励会增加该区块与每个叔区块受益者地址对应的账户的余额。假设受益账号为$$R_b$$，对于每个叔区块，我们增加一个占当前区块奖励$$\frac{1}{32}$$的额外奖励，并且叔区块的受益人获取的激励依赖于区块号。正式地定义为函数$$\Omega$$:
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

现在可以定义函数$$\Gamma$$，映射一个区块到其初始状态：
$$
(152) \\
\Gamma(B) \equiv \begin{cases} 
\sigma_0 \qquad \qquad \qquad \qquad \qquad \qquad \quad  if \quad P(B_H) = \phi \\
\sigma_i : TRIE(L_S(\sigma_i)) = P(B_H)_{H_r} \quad otherwise
\end{cases}
$$
这里$$TRIE(L_S(\sigma_i))$$表示字典树的根节点的状态$$\sigma_i$$的散列；假设协议的实现会将其细致高效的存储在状态数据库中，因为字典树本质上就是一个不可变的数据结构。

最后定义区块转移函数$$\Phi$$，映射一个尚未完成的区块$$B$$到一个完成的区块$$B'$$：
$$
(153)\qquad \Phi(B) \equiv B': B' = B^* \quad except: \\
(154)\qquad \qquad\qquad\quad\  B'_n = n : x \le \frac{2^{256}}{H_d} \\
(155)\ B'_m = m \quad with (x, m) = PoW(B^{*}_{n'}, n, d) \\
(156)\ B^* \equiv B \quad except: \quad B^*_r = r(\Pi(\Gamma(B), B))
$$
正如开头出设定的，$$\Pi$$是以区块完成函数$$\Omega$$和交易计算函数$$\Upsilon$$定义的状态转移函数。

$$R[n]_\sigma, R[n]_l$$和$$R[n]_u$$是每一笔交易之后第n个响应的状态，日志以及累积使用的的gas量（$$R[n]_b$$是元组中的第四个元素，已经以日志的形式定义了）。前者简单的定义为对前一个交易的状态（对于首笔交易来说是区块的初始状态）应用相关的交易后得到的状态：
$$
(157)\qquad R[n]_\sigma = \begin{cases}
\Gamma(B) \qquad\qquad \qquad \quad \ \  if \quad n < 0 \\
\Upsilon(R[n-1]_\sigma, B_T[n]) \quad otherwise
\end{cases}
$$
对于$$B_R[n]_u$$，我们用相似的方法将每个元素定义为在计算响应的交易时使用的gas与前一个元素的和（如果它是第一个，就是0）：
$$
(158)\qquad R[n]_u = \begin{cases}
0 \qquad \qquad \qquad \qquad if \quad n < 0 \\
\Upsilon^g(R[n-1]_\sigma, B_T[n]) \\
\quad + R[n-1]_u \qquad otherwise
\end{cases}
$$
对于$$R[n]_l$$，利用在交易执行函数中定义的函数$$\Upsilon^l$$来定义：
$$
(159) \qquad R[n]_l = \Upsilon^l(R[n-1]_\sigma, B_T[n])
$$
最后定义函数$$\Pi$$作为作用于最终交易状态$$\mathcal{l}(B_R)_\sigma$$的区块激励函数$$\Omega$$的新状态：
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

工作量证明函数更正式的形式为$$PoW$$:
$$
(161)\qquad m = H_m \wedge n \le \frac{2^256}{H_d} \quad with \quad (m,n) = PoW(H_{n'}, H_n, d)
$$
$$H_{n'}$$是新的区块头部，不过缺少了随机数和混合散列分量；

$$H_n$$是头部的随机数；$$d$$是一个计算混合散列需要的巨型数据集；$$H_d$$是新的区块的难度值（比如区块难度从段10开始）。$$PoW$$是用于计算出一个首元素为混合散列mixHash，第二个元素为一个加密依赖于$$H$$和**d**的伪随机数的数组的工作量证明函数。算法为$$Ethash$$.

