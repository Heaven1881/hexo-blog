---
title: Paxos算法详解
tags:
  - paxos
  - 分布式
  - 一致性
categories:
  - 分布式架构
date: 2016-12-25 11:53:01
---


Paxos是一个非常有名的一致性算法，其目的是为在网络中的各个进程提供信息同步的方法。之前对Paxos的了解一直很模糊，前段时间认真的把相关的知识过了一遍，总算才理解了Paxos的主要思想。现在特地记录下来。

<!-- more -->

如果你英文还不错，我强烈推荐你去阅读Paxos的论文：[Paxos Made Simple by Lamport](http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf)和[Paxos的Wiki](https://en.wikipedia.org/wiki/Paxos_%28computer_science%29)，里面说得更详细。这篇博客主要基于上面提到的两篇文章和我自己的一些理解。

# 假设
为了简化问题，我们做出以下假设：

- 机器节点
 - 每台机器的处理速度是任意，他们完成一项任务的时间是不确定的。
 - 每台机器都有可能宕机
 - 每台机器可以在宕机后重新加入Paxos集群中

- 网络
 - 任意两台机器之间通过网络相连
 - 网络的传输时间是不确定的
 - 网络传输有可能出现丢失，乱序，或者重复的情况
 - 消息在传输过程中不会被更改，既一个节点接受到的消息内容一定与其发送时的一样

- 机器节点的数量
 - 一般来说，如果我们有$2F+1$台机器，只要同时宕机的机器不超过$F$台，一致性算法就能保证整个协议正常工作

# Paxos基本定义
## 基本角色
Paxos根据每个节点在Paxos集群的行为来描述不同的节点，他们包括客户端(**client**)，接受者(**acceptor**)，提议者(**proposer**)，学习者(**learner**)，以及领导者(**learner**)。

### Client：
**Client**负责向Paxos集群中发起请求，然后等待响应。一般情况下，**Client**发起的请求都是写请求，例如向分布式系统中写入一个文件。

### Acceptor：
我们也可以把**Acceptor**称作**Voter**(投票者)，因为他们的行为就类似于投票。在**Acceptor**中有一个概念叫“多数派”(英文中称作“**Quorum**”)，**Quorum**的定义我们会在下面提到，在这里我们需要了解的是**Quorum**包含超过半数的**Acceptor**：
 - 如果一台机器要给**Acceptor**发送消息，那么需要将消息发送给**Quorum**，也就是要发送给超过半数的**Acceptor**。
 - 一台机器会忽略来自**Acceptor**的单条消息，除非这则消息同时来自于**Quorum**(超过半数的**Acceptor**)。涉及到的消息包括`Promise`和`Accepted`等。

### Proposer
**Proposer**负责接收并处理来自**Client**的请求，然后尝试说服每个**Acceptor**节点来同意**Client**的请求(这也就是为什么**Acceptor**也有Voter的名称)。除此之外，当Paxos集群遇到冲突时，**Proposer**则充当其中的调剂者。

### Learner
**Learner**在Paxos集群中负责真正执行命令。一旦**Client**的请求被**Acceptor**同意，**Learner**便开始真正执行**Client**请求，然后将结果返回给**Client**。我们可以通过增加**Learner**的数量来提高Paxos集群的健壮性。

### Leader
**Leader**是特殊的**Proposer**，因为Paxos集群允许有多个**Proposer**，而**Leader**就是**Properser**中的特殊角色。需要注意的是，很可能有多个**Proposer**认为他们自己是**Leader**。对于Paxos集群来说，只能有一个**Leader**，否则集群会进入冲突状态。如果有两个**Proposer**认为自己是**Leader**，那么这两个**Leader**可能就会在提交请求时不断地遇到冲突，不过在这种情况下，Paxos集群的一致性仍然没有被破坏，所以我们可以先忽略这种情况，实际上，有很多变种Paxos算法可以解决这个问题，这里我们先不做讨论。

## Quorum
我们可以把**Quorum**理解为**Acceptor**的子集，**Quorum**的出现是为了保证Paxos集群一致性和安全性不受部分机器宕机的破坏。**Quorum**定义为**Acceptor**的多数派，也就是说，任意两个**Quorum**之间至少包含一个相同的**Acceptor**。例如对于**Acceptor**集合$\{A,B,C,D\}$，**Quorum**可以是$\{A,B,C\}$，$\{A,B,D\}$，$\{A,C,D\}$，$\{B,C,D\}$以及$\{A,B,C,D\}$中的一个。

## 提议编号(Proposal Number)和接受值(Agreed Value)
**Proposer**通过*提议*来尝试让**Acceptor**接受某一个**值**，当然，由各个**Acceptor**来分别决定是否要接受对应的*提议*。一个*提议*一旦被**Quorum**接受，那么*提议*的内容就成为*接受值*。

对于一个给定的**Proposer**来说，每一个*提议*都会附带一个全局唯一的编号，称为*提议编号*。

# 安全性和持续性
为了保证集群的正常运转，Paxos定义了三条基本属性，无论在什么情况下，都需要保证这三条属性成立。

- 只有*提议值*可以被**Learner**学习
- 每次只有一个*提议值*被**Learner**学习
- 如果**Proposer**发起了一个*提议*，那么最终**Learner**一定会学习到某一个*提议值*(除非大部分的机器全部失败了)

# 实际部署
在实际应用中，一个机器节点有可能同时在Paxos集群中扮演多个角色，这不会影响Paxos算法的正确性。

将多个角色合并到同一个节点可以有效的减少网络的传输消耗，所以实际情况下大多数Paxos集群都采用这样的部署方式。在这里，为了简单起见，我们在下面的介绍中假设一个节点代表一个角色。为了方便，我会使用角色对应的英文名来进行介绍。

# 基础Paxos
Paxos发展到现在，已经产生了很多变体，它们中的大多数都为了进行特定的优化。我们所指的基础Paxos，就是最原始的Paxos算法，不包括任何变体和优化。

Paxos集群不断重复地在运行Paxos协议，我们可以称之为*回合*，每一个Paxos*回合*都有若干*轮*，每一*轮*都包括两个阶段。*回合*的发起者是**Proposer**，**Proposer**会发起一个回合当且仅当其受到**Client**的请求同时可以连接到多数**Acceptor**(**Quorum**)。

## 阶段 1a：Prepare
在这一阶段**Proposer**(**Leader**)创建一个编号为$N$的*提议*，这里$N$必须大于该**Proposer**之前使用所有提议编号。然后**Proposer**将这条提议发送到**Quorum**(**Proposer**可以任意选择一个**Quorum**来发送消息)。

## 阶段 1b：Promise
当**Acceptor**接受到来自阶段1a的`Prepare`消息，如果**Proposer**发送的提议编号大于**Acceptor**之前收到的所有提议的编号，那么**Acceptor**必须返回对应的`Promise`消息，`Promise`消息并不表示*提议*被同意，而是表明从现在开始**Acceptor**会忽略所有提议编号小于$N$的提议。不过，如果**Acceptor**在收到这则消息之前已经同意(`Accpetd`)了另一个*提议*，那么他会在返回的`Promise`消息中附加对应的*提议值*。

如果新的*提议编号*小于**Acceptor**曾经发送`Promise`时对应的提议编号，那么**Acceptor**可以直接忽略这则消息。不过**Acceptor**最好可以回复`Nack`消息，明确拒绝当前*提议*，让**Proposer**可以早一点开始新一*轮*的提议。

## 阶段 2a：Accept
如果**Proposer**收到足够的`Promise`消息(来自于**Quorum**，也就是多数**Acceptor**)，那么它就可以为之前的提议设置对应的*提议值*了。如果**Proposer**发现收到的`Promise`消息中包含了*提议值*，那么它必须要把本次*提议*的*提议值*设置为`Promise`消息中的*提议值*。如果`Promise`消息中没有附加*提议值*，那么**Proposer**可以随意指定*提议值*。

在这个阶段，**Proposer**给**Acceptor**发送包含*提议值*的`Accept`消息。

## 阶段 2b：Accepted
当**Acceptor**收到*提议编号*为$N$的`Accept`消息，**Acceptor**会同意这个提议当且仅当**Acceptor**没有给其他**Proposer**发送编号大于$N$`Promise`消息。换句话说，如果在这个期间，**Acceptor**收到了编号为$N+1$的*提议*并且发出了`Promise`消息，那么它就不能同意当前的`Acceptor`消息，因为当前消息的*提议编号*为$N$，小于$N+1$。

如果**Acceptor**同意了当前的`Accept`消息，那么他会记录下当前的*提议值*，然后通知每一个**Learner**当前的结果。

我们可以发现，如果有多个**Proposer**同时发送`Prepared`消息，当前*轮*很有可能会失败。如果**Proposer**没有收到足够多的来自**Acceptor**的回复，那么它应该认定当前*轮*是失败的，这种情况下，**Proposer**应该开启新一轮Paxos流程(注意是新一轮，不是新一回合，在一回合包含多轮)。

在这个阶段，**Acceptor**是有可能同时同意多个*提议*的。不过由于**Quorum**概念的引入，可以保证最终只会有一个*提议值*被**Learner**学习。一台机器会忽略来自**Acceptor**的单条消息，除非这则消息同时来自于**Quorum**。**Learner**不会学习*提议值*，除非有超过半数的**Acceptor**通知了这个*提议值*。

注意到当**Acceptor**同意一个*提议*它们也会同时通知**Proposer**，Paxos也可以用来选举**Leader**。

# Paxos算法运行示例
在下面的示例中，我们假设Paxos集群中有一个**Client**和一个**Proposer**，3个**Acceptor**和2个**Learner**。首先由**Client**发起请求，**Proposer**负责向**Acceptor**提交请求(提议编号为1)，在**Acceptor**返回的`Promise`消息中，`null`表示对应的**Acceptor**之前没有同意过任何*提议*。

## 正常流程
```
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |<---------X--X--X       |  |  Promise(1,{null,null,null})
   |         X--------->|->|->|       |  |  Accept!(1,Vn)
   |         |<---------X--X--X------>|->|  Accepted(1,Vn)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
```
## Acceptor 失败
Paxos协议不要求所有**Acceptor**都持续工作，只要同时失败的**Acceptor**小于半数即可(这样剩余的机器仍然能够组成**Quorum**)。

下面的例子中只需要2个**Acceptor**即可组成**Quorum**。

```
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |          |  |  !       |  |  !! FAIL !!
   |         |<---------X--X          |  |  Promise(1,{null,null})
   |         X--------->|->|          |  |  Accept!(1,V)
   |         |<---------X--X--------->|->|  Accepted(1,V)
   |<---------------------------------X--X  Response
   |         |          |  |          |  |
```

## Learner 失败
Paxos集群只需要一台**Learner**就可以正常运行。

```
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |<---------X--X--X       |  |  Promise(1,{null,null,null})
   |         X--------->|->|->|       |  |  Accept!(1,V)
   |         |<---------X--X--X------>|->|  Accepted(1,V)
   |         |          |  |  |       |  !  !! FAIL !!
   |<---------------------------------X     Response
   |         |          |  |  |       |
```

## Proposer 失败
下面的例子显示的是，**Proposer**在发出`Accept`消息后失败。我们忽略**Proposer**的**Leader**选举过程。

```
Client  Proposer        Acceptor     Learner
   |      |             |  |  |       |  |
   X----->|             |  |  |       |  |  Request
   |      X------------>|->|->|       |  |  Prepare(1)
   |      |<------------X--X--X       |  |  Promise(1,{null, null, null})
   |      |             |  |  |       |  |
   |      |             |  |  |       |  |  !! Leader fails during broadcast !!
   |      X------------>|  |  |       |  |  Accept!(1,Va)
   |      !             |  |  |       |  |
   |         |          |  |  |       |  |  !! NEW LEADER !!
   |         X--------->|->|->|       |  |  Prepare(2)
   |         |<---------X--X--X       |  |  Promise(2,{null, null, null})
   |         X--------->|->|->|       |  |  Accept!(2,V)
   |         |<---------X--X--X------>|->|  Accepted(2,V)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
```

## 多个Proposer冲突
下面的例子假设一个**Proposer**在失败之后恢复，有两个**Leader**在同时给**Acceptor**发送*提议*的情况。在这种情况下，虽然Paxos集群有可能无止境的冲突下去，但是Paxos集群的正确性没有被破坏。

```
Client   Proposer        Acceptor     Learner
   |      |             |  |  |       |  |
   X----->|             |  |  |       |  |  Request
   |      X------------>|->|->|       |  |  Prepare(1)
   |      |<------------X--X--X       |  |  Promise(1,{null,null,null})
   |      !             |  |  |       |  |  !! LEADER FAILS
   |         |          |  |  |       |  |  !! NEW LEADER (knows last number was 1)
   |         X--------->|->|->|       |  |  Prepare(2)
   |         |<---------X--X--X       |  |  Promise(2,{null,null,null})
   |      |  |          |  |  |       |  |  !! OLD LEADER recovers
   |      |  |          |  |  |       |  |  !! OLD LEADER tries 2, denied
   |      X------------>|->|->|       |  |  Prepare(2)
   |      |<------------X--X--X       |  |  Nack(2)
   |      |  |          |  |  |       |  |  !! OLD LEADER tries 3
   |      X------------>|->|->|       |  |  Prepare(3)
   |      |<------------X--X--X       |  |  Promise(3,{null,null,null})
   |      |  |          |  |  |       |  |  !! NEW LEADER proposes, denied
   |      |  X--------->|->|->|       |  |  Accept!(2,Va)
   |      |  |<---------X--X--X       |  |  Nack(3)
   |      |  |          |  |  |       |  |  !! NEW LEADER tries 4
   |      |  X--------->|->|->|       |  |  Prepare(4)
   |      |  |<---------X--X--X       |  |  Promise(4,{null,null,null})
   |      |  |          |  |  |       |  |  !! OLD LEADER proposes, denied
   |      X------------>|->|->|       |  |  Accept!(3,Vb)
   |      |<------------X--X--X       |  |  Nack(4)
   |      |  |          |  |  |       |  |  ... and so on ...
```

# 总结
上面给出的例子只是Paxos中的一部分，也是最原始算法，实际上Paxos有很多变体来优化效率，同时还可以减少冲突。在这里我就不一一介绍了，有兴趣的读者可以到对应的WIKI上了解更多的细节。

# 参考资料
> [Paxos Made Simple by Lamport](http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf)
> [Paxos-Wiki](https://en.wikipedia.org/wiki/Paxos_%28computer_science%29)

