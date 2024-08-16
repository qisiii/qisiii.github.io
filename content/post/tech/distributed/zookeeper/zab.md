+++
title = "Zookeeper源码分析-Zab"
description = ""
date = 2024-08-14T15:08:39+08:00
image = ""
draft = true
slug = "Zab"
tags = ['中间件','注册中心','zookeeper']
categories = ['tech']
+++

## 基本概念

ZAB是zookeeper专门为了保持分布式共识而实现的算法，算法的基石还是Paxos的原理，或者说Mluti Paxos的原理。和Raft算法很像，都具有leader的概念，消息都是通过leader去统一管理，其他节点只需要从leader同步消息即可，并且都是超过半数提交事务就认为成功了。

如果对分布式算法完全陌生，建议了解一下Paxos，MlutiPaxos，Raft的基本概念。

在ZAB协议中，其设计主要包括两部分：原子广播和崩溃恢复。

广播的意思是指当节点收到消息后，要将消息广播给其他的节点。那什么叫做原子呢？原子的意思是在这次广播中，要么所有的节点都收到消息并进行广播；要么所有的节点都放弃该条消息不做广播。

但仅是广播并不能保证所有节点的数据完全一致，还需要保证消息必须有序的处理，每个节点必须处理完上一个消息，才能处理下一条消息。

这是原子广播的两个基本要求：原子和有序。

当leader挂掉的时候，新的leader必须尽快了解并统一整个集群当前的进度。因此，在[zk选举流程]({{<ref "zookeeper学习-集群.md#选举逻辑">}}) 中，可以理解为什么必须选择最大事务ID作为leader，因为只有这样，才能在当前节点拥有最全的事务日志。当拥有了最全的事务后，从节点去进行同步到最新的进度。假如原来的leader又恢复了，那么就需要原来的leader丢弃超出现有leader的事务。这就是ZAB的另一部分--崩溃恢复。

## 参考文档:

[Zab协议 (史上最全) - 疯狂创客圈 - 博客园](https://www.cnblogs.com/crazymakercircle/p/14339702.html#autoid-h2-5-5-4)

[【图解源码】Zookeeper3.7源码剖析，Session的管理机制，Leader选举投票规则，集群数据同步流程 - 掘金](https://juejin.cn/post/7112681401601769486#heading-23)

[《Zookeeper》源码分析（十九）之 LearnerHandler-CSDN博客](https://blog.csdn.net/Lanna_w/article/details/132414570)

[20 共识算法：一次性说清楚 Paxos、Raft 等算法的区别](https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/24%e8%ae%b2%e5%90%83%e9%80%8f%e5%88%86%e5%b8%83%e5%bc%8f%e6%95%b0%e6%8d%ae%e5%ba%93-%e5%ae%8c/20%20%20%e5%85%b1%e8%af%86%e7%ae%97%e6%b3%95%ef%bc%9a%e4%b8%80%e6%ac%a1%e6%80%a7%e8%af%b4%e6%b8%85%e6%a5%9a%20Paxos%e3%80%81Raft%20%e7%ad%89%e7%ae%97%e6%b3%95%e7%9a%84%e5%8c%ba%e5%88%ab.md)
