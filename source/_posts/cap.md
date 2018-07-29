---
title: 笔记-CAP定理
tags:
  - CAP
categories: 分布式架构
date: 2018-07-07 08:18:44
---


CAP定理最开始是计算机科学家 Eric Brewer 提出的一个猜想。

<!-- more -->

CAP介绍了分布式系统中的三个性质：
- **一致性（Consistency）**：在同一时刻，每个节点本地的数据都完全相同。
- **可用性（Availability）**：失败的节点不会影响正常节点的运行。
- **分区容忍（Partition tolerance）**：出现网络消息丢失时，整个系统依然可以运行。

CAP定理的内容就是，在分布式系统中，只能满足上述性质的其中两项。可以参考下面的图片：

![](/uploads/cap.png)

## 三个分布式系统模型
我们根据上面的图，可以得到三种类型的分布式系统

- **CA （一致性+可用性）**：这种模型依赖全部的节点的正常工作，例如**2PC(2-Phase Commit)**。
- **CP（一致性+分区容忍）**：这种模型牺牲一定的可用性，但是不需要全部节点正常工作（只要超过半数的节点正常即可），例如**Paxos**。
- **AP（可用性+分区容忍）**：这种模型一般需要引入合并冲突的机制，例如**Dynamo**。

这里需要注意的是，**一致性**并不是一个非黑即白的选择，所有分布式数据存储服务都需要保证数据的一致性，否则就不能够成为“数据存储服务”。对于**可用性**也是同理，一个毫无**可用性**的分布式系统是不能成为“服务”的。

所以，**CAP**中的**C（一致性）**更具体应该指**强一致性**。

CAP最主要的作用就是让开发人员在设计分布式系统时意识到**C（一致性）**,**A（可用性）**,**P（分区容忍）**三个方向的平衡，即不可能在三个方面都达到极致。

## 参考资料
> [Distributed system for fun and profit](http://book.mixu.net/distsys/single-page.html)