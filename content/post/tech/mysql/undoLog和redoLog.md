+++
title = "UndoLog和redoLog"
description = ""
date = 2023-12-30
image = ""
draft = false
slug = "UndoLog_redoLog"
tags = ['mysql']
categories = ['tech']
+++

## redolog

从ACID了解到，对于数据库来说，原子性和持久性是为了保证数据修改之后就不会再有变换，但是持久性这个动作不是一个原子性的操作，包含写入、未写入和正在写多个状态。

假如是未提交事务（内存操作），这个时候就将数据落盘，那么这个时候如果崩溃了，在恢复数据的时候，就要将落盘的数据修改为未提交事务之前的数据。

假如是提交事务了（内存操作），这个时候数据还没有落盘，这个时候如果崩溃，就要在磁盘补充所落下的部分。

因此，不能将写入磁盘作为一个简单的动作看待，目前通用的方案是通过日志顺序写入，在提交事务后，日志追加Commit record语句，然后开始写入磁盘，当完全写入之后，追加End record语句，这种方案被称为Commit Log，这个日志被称为redo Log.

至此redolog其实是为了解耦内存和硬盘的一致性，那核心原因是什么呢？

就在于顺序IO和随机IO的不同

## undolog

但是这样存在的问题就是，写入磁盘的动作一定是在提交事务（日志写入Commit record）之后，这样子无法利用可能空闲的磁盘IO，这样对数据库的性能有一定的影响。

因此，ARIES理论提出了一个 write-head logging（提前写入)的方案，他将根据事务提交前后分为不同的状态。

Force：事务提交后就一定要落入磁盘。

NoForce: 事务提交后不一定要立即落入磁盘。因为有了日志，随时可以落入磁盘。

STEAL：事务在提交前，允许数据提前写入。

NOSTEAL：事务在提交前，不允许数据提前写入。

像上面的Commit Log，就是属于 NotForce-NoSteal，这是因为假如事务提交前，数据已经提前写入，这个时候崩溃了的话，恢复的时候，不知道恢复成什么样子。

因此，为了可以提前写入数据，崩溃时为了知道恢复成什么样子，就需要另一个日志来保证恢复数据，这个日志记录要记录各个版本的数据。这个日志就是undolog。

这样子的话，崩溃恢复的时候，就会经历三个阶段

分析：找到所有没有end record的事务

重做：将所有有Commit record的数据落入磁盘，并追加end record

回滚：将经历过上两个状态的剩余的事务，即没有Commit record的数据，根据undolog进行回滚。

![](http://picgo.qisiii.asia/post/11-16-54-37-image.png)

## redolog和binlog的同步问题

redolog分析过了，主要是实现事务的原子性；

由binlog可知，其内部也会记录事务相关的数据，主要用于恢复和主从中，那么redolog和binlog以谁为准呢？先写redolog还是binlog呢？

假如先写redolog，那么在binlog写入之前crash的话，主库会由这个事务的更新记录，从库因为binlog中没有，所以主从不一致。

假如先写binlog，在写redolog中发生crash，那么则是从库由，主库没有。

因此，对于这个问题，不能简单的采用先后来完成，而是采用XA的两段式提交方案，

先写redolog，此时redolog是prepare状态；然后写binlog（write/fsync)，在binlog写完之后，redolog追加commit标志事务提交。

换句话说，就是将redolog和binlog视作不同的资源，只有两个资源都准备完成的时候，才进行提交，否则就进行回滚。

而整体是以binlog为基准的

当redolog在做的时候crash的话，由于此时事务还未提交，binlog中也没有，所以直接回滚即可，数据是一致的。

当redolog做完，在写binlog的时候crash的话，binlog中也没有，因此还是回滚。

当binlog写完的时候，crash的话，由于binlog中存在，已找到最新的事务id，然后在redolog重做，没有commit record的要追加commit record；

但是这里的前提是，开始了binlog，因为binlog 是有可能不开启的。

![](http://picgo.qisiii.asia/post/11-16-55-12-image.png)

## 关于XA提交时引发的性能问题？

[XA协议binlog 和 redo log的一致性问题](https://zhuanlan.zhihu.com/p/98791003)

# 参考文档：

[11 | 本地事务如何实现原子性和持久性？-极客时间](https://time.geekbang.org/column/article/319481)

[02 | 日志系统：一条SQL更新语句是如何执行的？-极客时间](https://time.geekbang.org/column/article/68633)

[MySQL中binlog和redo log的一致性问题_binlog 和 redo不同步-CSDN博客](https://blog.csdn.net/huangjw_806/article/details/100927097)

[23 | MySQL是怎么保证数据不丢的？-极客时间](https://time.geekbang.org/column/article/76161)
