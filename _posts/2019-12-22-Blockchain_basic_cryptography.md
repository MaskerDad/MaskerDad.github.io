---
layout: post
title: "Blockchain Basic: Cryptography"
description: ""
categories: [BlockChainLearning]
tags: [BlockChain]
redirect_from:
  - /2019/11/07/
typora-root-url: ..
---

比特币实际是一种加密货币(crypto-currency)

### 一. 密码学基础——哈希函数（cryptography hash function）的性质

哈希函数：$Y=H(X)$ ，其中$X$为无限长的字符串。$Y$为指定长度的字符串。就是将无限的空间映射到一个有限的空间中。

1. **collision resistance**: 抵抗哈希碰撞

   哈希碰撞就是存在$X \ne Y$, 但是 $H(X)=H(Y)$, 这很容易理解，毕竟X无限，Y有限，输入空间远远大于输出空间，肯定不能一一对应，所以一定会出现哈希碰撞。但是我们的目标就是找一个尽量不让他出现哈希碰撞的算法。理论上是不可实现的，但是在实际应用中可以验证某个算法的collision resistance性质比较好。

   - 即已知$X$,得不到$Y$使$H(X)=H(Y)$ 只能通过暴力手段一个一个试$X$。

2. **hiding**: 单向的，不可逆的

   只能由$X$计算$H(X)$, 已知$H(X)$不能反推得到$X$，这也就是加密的本质呀，要是随便都能反推，那还加什么密啊。要满足这个条件，要要求输入空间足够大，才不容易被破解。（输入空间小的话，一个一个试就能破解）

   -  正是这个性质，产生了区块链中的数字签名（digital commitment）

3. **puzzle friendly**: 这个是区块链中要求哈希函数要满足的性质

   翻译成人话就是，事前不可预测，只能一个一个试。

   咱们的目标是得到一个数$X$, 使$H(X)\le \text{target}$, 这里target已知。

   （target转化为二进制主要就是前面的0，前面的0越多，target的值就越小，输入空间无限大，输出空间变小了，求解$X$就变难啦）

   所以puzzle friendly就是已知target，要找$X$的话，只能在输入空间里一个一个的试，看看满不满足，而不能直接去求解。

   让我们代入到区块链中，区块链挖矿的过程就是$H(\text{block header} + \text{noise})\le \text{target}$ 这里，block header就是携带系统当前信息的一个字符串，这个没办法改。target是系统指定给的。noise是随机数，我们可以随便改。所以挖矿工人需要做的就是一个个试noise随机数，看看哪个随机数会满足$H(\text{block header} + \text{noise})\le \text{target}$，这就是挖矿的过程，第一个找到了一个noise随机数满足这个不等式的人，就相当于挖到了矿。别人会验证你找到的正不正确。验证的过程就很简单了，直接代入即可。

   - 挖矿很难，验证很容易。different to solve, easy to verify.

   上面的原理很简单，但是其实找这个noise的过程很慢，要试好多好多好多的数，所以就需要大量的算力，谁算得快，谁就有更大概率挖到矿。这也就是工作量证明(proof of work, PoW), 一般就是尝试的noise最多，也就是工作的最多的人能挖到矿，为什么叫挖矿呢，因为第一个找到noise的人，系统会有奖励呀，比如比特币，会奖励10个比特币的。比特币是可以兑换成钱的，所以叫挖矿。用来找noise而搭设的计算机叫矿机。

### 二、数据结构——哈希指针(hash pointers)

Blockchain is a linked list using hash pointers.

哈希指针跟之前C语言里面学的指针有关系，C语言里面学的指针只储存了一个指向的**内存地址**。而哈希指针则储存了指向的**内存地址**以及指向的内容的**哈希值**.

![blockchain](/images/posts/2019-12-22/blockchain.jpg)

- tamper-evident logging 篡改记录。区块链的很大的一个特性就是不可篡改。主要就是哈希指针起作用了，一旦篡改之前的某个区块，造成这个区块的哈希值改变，下一个区块的哈希指针中的哈希值就得变。然后下个区块也就被改了，就这样，从要篡改的区块后面所有的区块都要被修改，就像多米诺骨牌效应。导致最后的，，也就是指向最近，最新的区块的哈希指针，也就是当前指针被修改。因为当前指针被所有矿工保存着呢，一旦被修改，矿工是不认识，不承认的。所以。。。就没办法修改。

### 三、数据结构——Merkle tree

刚才说了整个区块链是怎么连接起来的，现在说说区块链中的区块保存的内容和数据结构——Merkle tree。

Merkle tree就是把二分树(binary tree)的普通指针换成哈希指针的。对，没错，就这么简单。加上哈希值，就没办法篡改其中的内容了。









