# 以太坊黄皮书（三）

## 区块、状态以及交易



#### 全局状态

*全局状态*（状态）是一个地址集（160位标识符）与账号状态集（序列化为RLP的数据结构）之间的映射。虽然这个映射并不存储在区块链上，我们仍然假设其具体实现会在一个修改过的默克尔树（trie，字典树）中维护这个映射关系。字典树需要一个简单的后台数据库维护字节数组到字节数组之间的映射关系，这个数据库称为*状态数据库*。这样做有很多好处：

1. 这个数据结构的根节点是加密依赖其全部的内部数据，因此它的散列值可以作为整个系统的安全标识符使用
2. 作为一个*不可变*的数据结构，它允许通过简单的切换根散列值来回溯到任意一个前续状态（根散列是已知晓的）

因为在区块链中存储了所有的根散列值，所以能够很简单的回退到旧的状态。

账号状态包括下列4个字段：

- nonce：标量，等于从当前地址发出的交易数量。对于带有关联代码的账户，等于当前账号创建的合约数量。对处于状态$$\sigma$$的账号*a*，该值用$$\sigma[a]_n$$表示
- balance：标量，与当前账户持有*Wei*的数量相同，用$$\sigma[a]_b$$表示
- storageRoot：一棵对账户的存储内容（256比特整数值之间的一个映射）进行编码的默克尔树的根节点的256比特散列值。字典树的内容是一个256比特整数的Keccak 256比特散列与RLP编码的256比特值之间映射关系的编码。这个散列值用$$\sigma[a]_s$$表示
- codeHash：当前账号的**EVM**代码的散列值，这个代码只有接收到一个消息调用时才执行；该字段是*不可变*的，因此和其他字段不同的是在创建以后无法修改。所有的这些代码段都包含与他们的散列值对应的状态数据库中，以用于后续检索。该散列值用**b**表示，且已知$$KEC(b) = \sigma[a]_c$$

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

$$\sigma[a]_s$$并不是账号的“物理”成员，也不会为其随后的序列化贡献任何力量。

如果**codeHash**字段是空字符串的Keccak-256散列，例如，$$\sigma[a]_c = KEC(())$$, 那么这个节点就表示一个*简单账号*，有时也称作“非合约”账号。因此我们可以定义一个全局状态塌陷函数$$L_S$$:
$$
(9) \qquad L_S(\sigma) \equiv \{p(a) : \sigma[a] \ne \phi\}
$$
其中，
$$
(10) \qquad p(a) \equiv (KEC(a), RLP((\sigma[a]_n, \sigma[a]_b, \sigma[a]_s, \sigma[a]_c)))
$$
函数$$L_S$$和字典函数一起为全局状态提供一个短标识符（散列），假设：
$$
(11) \qquad \forall a : \sigma[a] = \phi \vee  (a \in \mathbb{B_{20} \wedge v(\sigma[a])})
$$
其中$$v$$是账号的验证函数：
$$
(12) \qquad v(x) \equiv x_n \in \mathbb{P_{256}} \wedge x_b \in \mathbb{P_{256}} \wedge x_s \in \mathbb{B_{32}} \wedge x_c \in \mathbb{B_{32}}
$$

#### 交易

一笔交易是对以太坊来说一个外部操作者发起的一条简单的加密签名指令。外部操作者是一个自然人，软件工具可以用于其创建和传播。目前存在**2**种交易：

- 产生消息调用的交易
- 产生带有关联代码的新账户的创建（以非正式名称“合约创建”而熟知）的交易

这2种交易都指定了一些通用字段：

- nonce：标量，与发送者发送的交易数相同，用$$T_n$$表示。
- gasPrice: 标量，等于执行当前交易所消耗的计算量以每单位*gas*需要支付的*Wei*数，用$$T_p$$表示。
- gasLimit：标量，等于执行当前交易所要使用的*gas*上限。这个值是在任何计算完成之前预支的，随后可能并不增加，用$$T_g$$表示。
- to：消息调用接收者的160比特地址，对于*合约创建*交易，此处用$$\phi$$表示仅有的$\mathbb{B_0}$的成员，用$$T_t$$表示。
- value：标量，与准备转给消息调用的接收者的*Wei*数一致，对于*合约创建*来说作为新创建的账号的资金，用$$T_v$$表示。
- v, r, s：与交易的签名有关的值，被用于判定交易的发送者，用$$T_w, T_r, T_s$$表示。

此外，一笔*合约创建*交易包括：

- init：一个没有大小限制的字节数组，设定账户初始化过程的*EVM-code*， 用$$T_i$$表示。

**init**是一个**EVM-code**片段，其返回的是在每次收到消息调用（通过交易或者内部代码执行）时需要执行的第二个代码段 — **body**。**init**只在账号创建时执行一次，随后立即被丢弃。

相反，一个消息调用交易包括：

- data：一个没有大小限制的字节数组，用于指定消息调用的输入数据，用$$T_d$$表示。

$$
（13）\\
L_T(T) \equiv \begin{cases}
(T_n, T_p, T_g, T_t, T_v, T_i, T_w, T_r, T_s) & if  T_t = \phi \\
(T_n, T_p, T_g, T_t, T_v, T_d, T_w, T_r, Ts) & otherwise

\end{cases}
$$

这里假设除了任意长度的字节数组$$T_i$$和$$T_d$$，所有的模块都用RLP解释为整数。
$$
(14) \\
T_n \in \mathbb{P_{256}} \wedge T_v \in \mathbb{P_{256}} \wedge T_p \in \mathbb{P_{256}} \wedge \\
T_g \in \mathbb{P_{256}} \wedge T_w \in \mathbb{P_{5}} \wedge T_r \in \mathbb{P_{256}} \wedge \\
T_s \in \mathbb{P_{256}} \wedge T_d \in \mathbb{B} \wedge T_i \in \mathbb{B}
$$
其中，(15)  $$P_n = \{ P: P \in \mathbb{P} \wedge P \lt 2^n\}$$

地址散列$$T_t$$有稍许不同：要么是以个20字节的地址散列或者，要么对于*合约创建*交易（因此形式上等价于$$\phi$$）来说它是RLP空字节序列，因此也是$$\mathbb{B_0}$$的成员：
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

- parentHash：父区块头完整的Keccak 256位散列，用$$H_p$$表示
- ommersHash：叔区块列表的Keccak 256位散列，用$$H_o$$表示。
- beneficiary：为成功采集到当前区块所手机的有费用的160位转移地址。
- stateRoot：在全部交易执行完成之后，状态字典树根节点的Keccak 256位散列，用$$H_r$$表示。
- transactionsRoot：字典树结构的根节点，携带当前区块交易列表中每一笔交易的Keccak 256位散列，用$$H_t$$表示。
- receiptsRoot：字典树结构的根节点，携带当前区块交易列表中每一笔交易收据的Keccak 256位散列，用$$H_e$$表示。
- logsBloom：由交易列表中的每一笔交易收据中所包含的日志条目的可索引信息（日志地址和日志主题）组成的布隆过滤器，用$$H_b$$表示。
- difficulty：与当前区块的难度级别相关的标量。可以通过前驱区块的难度级别与时间戳计算得出，用$$H_d$$表示。
- number：当前区块的所有前驱区块的数量，创世区块的number为0，用$$H_i$$表示。
- gasLimit：标量，与目前每个区块的gas配额限制相同，用$$H_l$$表示。
- gasUsed：标量，与当前块中所有交易所耗费的gas总数相同，用$$H_g$$表示。
- timestamp：标量，与当前块诞生时的Unix系统time()合法输出相同，用$$H_s$$表示。
- extraData：一个包含当前区块相关数据的字节数组，必须小于等于32字节，用$$H_x$$表示。
- mixHash：一个256位的散列值，与*nonce*一起用于证明在当前区块上已经执行了足够的计算，用$$H_m$$表示。
- nonce：一个64位散列值，与*mixHash*一起用于证明在当前区块上已经执行了足够的计算，用$$H_n$$表示。

区块的另外两个模块比较简单，只是叔区块的头部列表（格式与前面描述的头部格式相同）和当前块中包含的交易序列。形式上说我们可以定义区块*B*为：
$$
(17)\\
B \equiv (B_H, B_T, B_U)
$$

##### 交易收据

为了将一笔交易中对构成一个*零知识证明*有用的信息进行编码，或者进行索引与搜索，我们编码了每一笔交易的收据，其中包含交易执行的信息。对于第$$i$$笔交易，其收据用$$B_R[i]$$表示，并放置于一个键索引的字典树中，树的根节点在区块头部中以$$H_e$$记录。

交易收据是一个带有4个成员的元组，其成员包括：

- *交易后状态*$$R_\sigma$$,
- 截止当前交易发生时刻，包含当前交易的区块中累积使用的gas量$$R_u$$,
- 当前交易执行过程中创建的日志集合$$R_l$$,
- 从日志信息中选择的布隆过滤器$$R_b$$

即，一个交易收据可以表示为：
$$
(18) \\

R \equiv (R_\sigma, R_u, R_b, R_l)
$$
函数$$L_R$$用于为正在转换成一个RLP序列化的字节数组准备一个交易收据：
$$
(19) \\
L_R(R) \equiv (TRIE(L_S(R_\sigma)), R_u, R_b, R_l)
$$
因此，交易后状态$$R_\sigma$$会被编码进一个字典树结构，其根节点为第一个成员。

需要声明累积使用的gas量$$R_u$$是一个正整数，并且日志布隆过滤器$$R_b$$是一个2048比特（256字节）的散列：
$$
(20) \\
R_u \in \mathbb{P} \wedge R_b \in \mathbb{B_{256}}
$$
日志集合$$R_l$$是一些列日志条目，例如$$(O_0, O_1, …)$$。一个日志条目$$O$$是一个由日志记录器地址$$O_a$$， 一系列32字节的日志主题$$O_t$$和数个字节的数据$$O_d$$构成的元组：
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
其中，$$M_{3:2048}$$是一个特殊的布隆过滤器，它对给定的任意一个字节序列从2048个比特中设置3个比特。其实现是通过对字节序列的Keccak-256散列的前3对字节的最低11比特进行操作：
$$
(24) M_{3:2048}(x:x \in \mathbb{B}) \equiv y: y \in \mathbb{B_{256}}\quad where: \\
(25)\quad\qquad\qquad\qquad y = (0, 0, ..., 0) \quad except: \\
(26)\quad\qquad\qquad\quad \forall_i \in \{0,2,4\}   :   \mathcal{B_{m(x, i)}}(y) = 1 \\
(27)\quad\quad m(X, i) \equiv KEC(X)[i, i + 1] \ mod\ 2048
$$
其中，$$\mathcal{B}$$是比特引用函数，例如$$B_j(x)$$等于字节数组$$X$$中下标为$$j$$的比特（下标从0开始）。



##### 整体有效性

当且仅当区块满足下列条件时开可以说它是有效的：其必须和叔区块以及交易区块散列保持内部一致性，并且给定的交易集$$B_T$$在基础状态$$\sigma$$（继承自父区块的完成状态）的基础上按顺序执行，从而生成一个新实体$$H_r$$的状态：
$$
(28) \\
H_r \equiv TRIE(L_S(\Pi(\sigma, B)))\qquad\qquad\qquad\qquad\qquad\qquad \wedge \\
H_o \equiv KEC(RLP(L^*_H(B_\cup))) \qquad\qquad\qquad\qquad\qquad\quad \wedge \\
H_t \equiv TIRE(\{\forall _i < \parallel B_T \parallel, i \in \mathbb{P} : p(i, L_T(B_T[i]))\})\qquad \wedge \\
H_e \equiv TIRE(\{\forall _i < \parallel B_R \parallel, i \in \mathbb{P} : p(i, L_R(B_R[i]))\})\qquad \wedge \\
H_b \equiv \bigvee_{r \in B_R(rb)} \qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad\qquad\quad
$$
其中$$p(k,v)$$是一对简单的RLP变换，这里第一个元素是交易在区块中的下标，第二个元素是交易收据：
$$
(29)\qquad\qquad p(k,v) \equiv (RLP(k), RLP(v))
$$
进一步说：
$$
(30)\qquad\qquad TRIE(L_S(\sigma)) = P(B_H)_{H_r}
$$
因此$$TRIE(L_S(\sigma))$$即包含状态$$\sigma$$与RLP编码的值的键值对的默克尔树的根节点的散列，$$P(B_H)$$是区块$$B$$的父区块。

源于交易计算的结果集，确切的说是交易收据$$B_R$$以及交易状态累积函数$$\Pi$$在下文列出。



##### 序列化

函数$$L_B$$和$$L_H$$是区块及其区块头部的准备函数。与交易收据准备函数$$L_R$$类似，当需要RLP变换的时候，需要判断类型和结构顺序：
$$
(31)\qquad L_H(H) \equiv (H_p, H_o, H_c, H_r, H_t, H_e, H_b, H_d, \\
H_i, H_l, H_g, H_s, H_x, H_m, H_n) \\
(32)\quad\quad\quad\ \  \ L_B(B) \equiv (L_H(B_H), L^*_T(B_T), L^*_H(B_U))
$$
$$L^*_T$$和$$L^*_H$$都是元素级别的序列转换函数，因此：
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
其中，$$(35)\quad \mathbb{B_n} = \{B : B \in \mathbb{B} \wedge \parallel B \parallel = n\}$$。关于如何构建一个正式的区块结构现在有了严格的规范，**RLP**

函数则提供了权威的方法将区块结构变换成一个可用于传输的字节序列。



##### 区块头部校验

定义$$P(B_H)$$为区块$$B$$的父亲区块，形式如下：
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

$$(39)\qquad D_0 \equiv 131072$$

$$(40)\qquad x \equiv \lfloor \frac{P(H)_{H_d}}{2048} \rfloor$$ 

$$(41)\qquad \varsigma_2 \equiv max (1 - \lfloor \frac{H_s - P(H)_{H_s}}{10} \rfloor, -99)$$

$$(42)\qquad \epsilon \equiv \lfloor 2^ {\lfloor H_i \div 10000 \rfloor - 2} \rfloor$$

区块头部$$H$$的gas限制$$H_l$$必须满足关系：

$$(43)\qquad H_l < P(H)_{H_l} + \lfloor \frac {P(H)_{H_l}}{1024} \rfloor \qquad \wedge$$

$$(44)\qquad H_l > P(H)_{H_l} - \lfloor \frac{P(H)_{H_l}}{1024} \rfloor \qquad \wedge$$

$$(45)\qquad H_l \ge 125000$$

$$H_s$$是区块$$H$$的时间戳，且需满足关系：

$$(46)\qquad H_s > P(H)_{H_s}$$

这个机制迫使区块之间以时间来保持内部平衡，最新两个区块之间一个更小的时间窗会导致难度级别的增加，从而需要额外的计算来延长下一个时间窗。相反，如果时间窗太大则难度，即下一个区块的期望时间会被缩小：

$$(47)\qquad n \le \frac {2^{256}}{H_d} \wedge m = H_m, \qquad (n, m) = PoW(H_{n'}, H_n, d)$$

$$H_{n'}$$是新区块的头部，但是*没有*nonce和mix-hash分量，**d**是当前的**DAG** — 一个计算mix-hash需要的庞大的数据集，**PoW**是工作量函数：这个计算得出一个数组，第一个元素是mix-hash用于证明使用了一个正确的**DAG**；第二个元素是一个密码学上依赖于*H*和**d**的随机数。对于区间[0, $$2^{64}$$)上近似于均态的分布，找到结果的期望时间与难度$$H_d$$成正比。这是区块链的安全基础，也是为什么恶意节点无法将新创建的可以重写历史记录的区块进行传播的根本原因。由于*nonce*需要满足这个需求，且因为其满足的情况依赖于区块的内容，从而反过来随着时间推移，区块中包含的交易用于创建新的有效区块会非常困难，几乎需要矿工中全部可信节点的计算力才能完成。

因此我们可以定义头部校验函数$$V(H)$$:
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
其中，$$(n, m) = PoW(H_{n'}, H_n, d)$$，**extraData**最大不超过32字节。