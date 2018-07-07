---
title: CRDT介绍(1)：Counter
tags:
  - CRDT
  - 最终一致性
categories: 分布式架构
date: 2018-06-09 21:41:09
---


CRDT是Conflict-Free Replicated Data Types的缩写，直译的话即“无冲突可复制数据类型”。

<!-- more -->

# 背景
在分布式系统中，经常需要在多个节点上部署同一份数据。这时候保持这些数据的一致性就成了一个最常遇到的问题。关于一致性，一般有两种实现模型：

一种模型是**原子一致性（atomic consistency）**也叫作**Linearizability**。例如Paxos算法或者Raft算法。这些方法可以保证多份数据的强一致性，但是因为需要多次进行网络通讯，所以执行速度相对较慢。并且在对数据修改时需要保证能够连接到超过半数的节点(Quorum)，无法对孤立的节点进行数据修改。

另一种模型则是**最终一致性（eventually consistency）**。即只要数据最终能保证一致性，那么允许数据在修改过程中出现不一致的情况。这样的方法在对数据修改时不需要立即将修改同步到其他节点，因此就算在孤立的节点也能对数据进行修改。

但是既然两个节点的数据出现了不一致，那么为了达到最终一致性，就不可避免要对两个节点的数据进行合并操作。而CRDT数据类型就能保证在合并过程中不会出现冲突，即所有节点的数据都能自动达成一致。

# CRDT
实际上，CRDT的应用非常广泛。Radis底层的数据库就使用了CRDT数据类型，还有Amazon的Dynamo, Facebook的Apollo等都使用到了CRDT。CRDT可以有两种形式：

- **基于状态**：(英文：state-based)，即将各个节点之间的CRDT数据直接进行合并，所有节点都能最终合并到同一个状态，数据合并的顺序不会影响到最终的结果。
- **基于操作**：(英文：operation-based，可以简写为op-based)。将每一次对数据的操作通知给其他节点。只要节点知道了对数据的所有操作（收到操作的顺序可以是任意的），就能合并到同一个状态。

在实际应用中**基于状态**的的CRDT在网络同步上的效率相对较低，因为数据的状态可能是一个很大的数据块，所以实际使用时一般只是将发生修改的部分通知到其他节点，不过这样做实际上是依赖了**基于操作**的相关机制。

> 实际上，**基于状态**和**基于操作**的CRDT已经在数学上被证明可以相互转换。感兴趣的读者可以参考底部给出的论文链接。

下面介绍几个比较有意思的CRDT类型。

### G-Counter (Grow-only Counter)
这是一个只增不减的计数器。对于N个节点，每个节点上维护一个长度为N的向量$V=\{P_0, P_1, P_2, ..., P_{n-1}\}$。$P_m$表示节点m上的计数。当需要增加这个计数器时，只需要任意选择一个节点操作，操作会将对应节点的计数器$P_m := P_m + 1$。当要统计整个集群的计数器总数时，只需要对向量$V$中的所有元素求和即可。

举个例子，我们有A, B, C三个节点上分别部署了同一个G-Counter。初始状态如下：
```
A: {A: 0, B: 0, C: 0} = 0
B: {A: 0, B: 0, C: 0} = 0
C: {A: 0, B: 0, C: 0} = 0
```
每个节点都维护着其他节点的计数器状态，初始状态下，所有计数器都为0。现在我们假设有三个客户端a,b,c，a和b先后在节点A上增加了一次计数器，c在节点B上增加了一次计数器：
```
A: {A: 2, B: 0, C: 0} = 2
B: {A: 0, B: 1, C: 0} = 1
C: {A: 0, B: 0, C: 0} = 0
```
此时如果分别向A, B, C查询当前计数器的值，得到的结果分别是{2, 1, 0}。而实际上一共有3次增加计数器的操作，因此全局计数器的正确值应该为3，此时系统内状态是不一致的。不过没关系，我们追求的是最终一致性。假设经过一段时间，B向A发起了合并的请求:
```
A: {A: 2, B: 1, C: 0} = 3
B: {A: 2, B: 1, C: 0} = 3
C: {A: 2, B: 1, C: 0} = 3
```
经过合并后，A和B的计数器状态合并，现在从A和B读取到的计数器的值变为3。接下来C和A进行合并：
```
A: {A: 2, B: 1, C: 0} = 3
B: {A: 2, B: 1, C: 0} = 3
C: {A: 2, B: 1, C: 0} = 3
```
现在节点ABC都达到了相同的状态，从任意一个节点获取到的计数器值都为3。也就是说3个节点达成了最终一致性。

我们可以用以下伪代码描述G-Counter的逻辑：

```
payload integer[n] P
	initial [0, 0, 0, ..., 0]
update increment()
	let g = myId()
	P[g] := P[g] + 1 // 只更新当前节点对应的计数器
query value() : integer v
	let v = P[0] + P[1] + ... + P[n-1] // 将每个节点的计数器进行累加
merge (X, Y) : payload Z
	for i = 0, 1, ... , n-1:
		Z.P[i] = max(X.P[i], Y.P[i]) // 通过max操作来合并各个节点的计数器
```

G-Counter使用max()操作来进行各个状态的合并，我们知道函数max满足可交换性，结合性，幂等性，即：
- **可交换性**： $max(X, Y) = max(Y, X)$
- **结合性**： $max(max(X, Y), Z) = max(X, max(Y, Z))$
- **幂等性**： $max(X, X) = X$

所以G-Counter可以在分布式系统中使用，并且可以无冲突合并。

### PN-Counter (Positive-Negative-Counter)
G-Counter有一个限制，就是计数器只能增加，不能减少。不过我们可以通过使用两个G-Counter来实现一个既能增加也能减少计数器（PN-Counter）。简单来说，就是用一个G-Counter来记录所有累加结果，另一个G-Counter来记录累减结果，需要查询当前计数器时，只需要计算两个G-Counter的差即可。

```
payload integer[n] P, integer[n] N
	initial [0, 0, ..., 0], [0, 0, ..., 0]
update increment()
	let g = myId()
	P[g] := P[g] + 1 // 只更新当前节点对应的计数器
update decrement()
	let g = myId()
	N[g] := N[g] + 1
query value() : integer v
	let v = sum(P) - sum(N) // 用P向量的和减去N向量的和
merge (X, Y) : payload Z
	for i = 0, 1, ... , n-1:
		Z.P[i] = max(X.P[i], Y.P[i]) // 通过max操作来合并各个节点的计数器
		Z.N[i] = max(N.P[i], Y.N[i])
```

## 参考资料
> [谈谈CRDT - github.io](http://liyu1981.github.io/what-is-CRDT/)
> [A comprehensive study of CRDT](https://hal.inria.fr/file/index/docid/555588/filename/techreport.pdf)