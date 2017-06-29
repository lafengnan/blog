# 以太坊黄皮书笔记(一)

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

