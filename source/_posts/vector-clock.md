---
title: Vector Clock
tags:
  - Vector Clock
  - Lamport Clock
categories: 分布式架构
date: 2018-05-19 14:11:38
---


Vector Clocket其实很简单。即使用一个包含各个节点版本号的向量来代替分布式系统中不是特别可靠的时间戳。

<!-- more -->

## 背景
在分布式系统中，不同机器上的物理时钟是相互独立的。因此，不同的机器在同一时间点取到的时间不是完全一样的。借助NTP时间同步协议，我们一般可以将不同机器之间的时钟误差控制在几十毫秒的量级。但是仍然无法通过各个机器上的时间戳来准确判断事件的先后。

## Lamport Clock
上面问题的原因就是：在单机系统中，因为只有一个物理时钟，因此事件的发生时间是全序的，即单机系统上的任意两个事件都可以比较先后。但是在分布式系统中，事件的发生时间是偏序的，虽然单一节点上的时间可以比较先后，但是不同机器上的事件无法直接比较先后。

后来，人们想到使用计数器来代替物理时钟时间戳进行事件先后比较。 Lamport Clock就是一个最简单的例子，在使用lamport Clock的分布式系统中，每个节点本地都维护一个计数器，并且遵守以下规则：
-  当节点完成一项工作，Lamport Clock 计数器加一
-  当节点收到一条带有Lamport Clock的消息时，如果收到的计数和本地计数不相等，那么将本地的计数器更新为最大的那个值。最后，无论是否更新了本地的计数器，都将计数器的值加一

根据这个规则，我们可以可以得到如下结论：
- 如果事件A发生在事件B之前，那么我们可以有`LamportClock(A) < LamportClock(B)`

不过遗憾的是，我们并不能将上面的结论反过来使用，因为：如果`LamportClock(A) < LamportClock(B)`，那么一种可能是事件A发生在事件B之前，另一种可能是事件A和事件B之间的顺序无法比较。

虽然我们不能判断任意两个事件发生的顺序，但是Lamport Clock仍然有一个不错的性质：对于单个节点而言，如果发送了一条消息`a`，那么所有收到的回应消息`b`，都有`LamportClock(a) < LamportClock(b)`。这个方法可以方便排除过时的响应消息。

## Vector Clock
Lamport Clock 在分布式系统上的限制本质上是因为它只维护了一个时间线。而这样的假设并不能很好的和分布式模型贴合。

Vector Clock 实际上是 Lamport Clock 的扩展，其维护一个向量$[t_1, t_2, t_3, ..., t_n]$。其中$t_n$代表节点n的时间计数。相比来说Vector Clock为每一个节点都分配了一条时间线，因此我们可以通过Vector Clock准确比较两个事件的状态。每个节点根据以下几条规则来维护Vector Clock：
- 当节点m完成一项工作时，向量中$t_m$的值加一
- 当节点接受到含有Vector Clock的消息时：
	- 对于向量中的每个元素，如果接收到的向量对应元素更大，则使用较大的那个值。即：`local = max(local, received)`
	- 取$t_m = t_m + 1$

对于`VectorClock(A)`和`VectorClock(B)`：
- 如果`A`中向量的所有元素都比`B`小，那么认为事件A发生在事件B之前
- 如果`A`中向量的所有元素都比`B`大，那么认为事件A发生在事件B之后
- 否则，事件A和事件B不可比较，认为两个事件同时发生。

下图是一个Vector Clock的示例。

![](/uploads/vector-clock.png)

我们以`{A:2, B:4, C:1}`这个节点为例，所有蓝色背景中的时间点都发生在这个节点之前，红色背景的节点都发生在这个节点之后。而白色背景的节点与这个节点的顺序是不可比较的，可以认为他们同时发生。

对于Vector Clock，最主要的问题是每条消息都需要附带这样一个向量，在节点数很多的时候，这个向量有可能变得很大。目前也有很多Vector Clock 的衍生技术，旨在通过周期性计数器回收和降低精度的方法来减小Vector Clock的数据量。

## 参考资料
> [Distributed systems for fun and profit :  charpter 3  time and order](http://book.mixu.net/distsys/single-page.html)
> [Vector Clock - Wiki](https://en.wikipedia.org/wiki/Vector_clock)
> [Vector Clock/Version Clock - cnblogs](http://www.cnblogs.com/foxmailed/p/4985848.html)