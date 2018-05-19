---
title: FLP Impossibility
tags:
  - FLP
  - 一致性
categories: 分布式架构
date: 2018-05-19 13:59:41
---



FLP的名字来自于该定理论文的三个作者（Fischer, Lynch, Patterson）

<!--more-->

# 主要内容

FLP Impossibility（FLP不可能性）的主要内容如何下：

在异步分布式模型中，即使只有一个节点失效（宕机），即使网络消息不会丢失，也不存在可以解决一致性问题的算法。

原文如下：
> There does not exist a (deterministic) algorithm for the consensus problem in an asynchronous system subject to failures, even if messages can never be lost, at most one process may fail, and it can only fail by crashing (stopping executing)

证明的方法也很简单：假设存在这么一个算法，那么我们可以构造一个节点逻辑，这个节点逻辑会等待任意长的时间后发送结果（这是符合异步时间模型的）。这样整个分布式系统就无法实现一致性。

FLP不可能性最重要的结果就是让开发人员意识到自己需要做一个不可避免的权衡，即：如果无法控制消息传输延迟的上界，那么一致性算法要么放弃安全性（Safety）要么放弃运行（Liveness）。

# 和CPA比较
其实FLP和CPA两个理论很相似，区别是：
- CAP主要假设网络失败而不是节点失败
- CAP定理在分布式系统设计上的指导性更强