+++
title = 'Zookeeper源码分析-watch机制'
date = 2024-08-08T23:34:10+08:00
draft = false
tags = ['中间件','注册中心','zookeeper']
categories = ['tech']
slug ="watch"
+++

Zookeeper框架设计了一种叫做watch的机制，客户端可以给某个节点添加watcher，当节点改变的时候，服务端会推送一个消息到客户端。

在3.6.0之前的版本，node上的watcher都是一次性的，即当收到服务端的通知后，如果还想监听需要再次设置，在3.6.0之后，可以设置一个永久且递归的watcher。

这个技术是很有用的，比如在dubbo中，consumer端可以只关注自己感兴趣的provider，而不用关心全部的provider，本文就来梳理一下这个watch机制。

目前普通的方法，watch机制仅允许在getData、exist、getChildren这三个方法调用的时候进行添加。或者直接调用addWatch方法给指定节点添加watcher。

当我们使用的时候，可以自定义watcher的逻辑，也可以使用创建zookeeper时指定的watcher逻辑。

```java
//使用默认的watch
zookeeper.getData(nodePath, true, null);
zookeeper.addWatch(nodePath,AddWatchMode.PERSISTENT);
//使用自定义逻辑
zookeeper.getData(nodePath, watchedEvent->{
            if(watchedEvent.getType()==Watcher.Event.EventType.NodeCreated){
                System.out.println("新增节点"+watchedEvent.getPath());
            }
            if(watchedEvent.getType()==Watcher.Event.EventType.NodeDeleted){
                System.out.println("删除节点"+watchedEvent.getPath());
            }
        },null);
zookeeper.addWatch(nodePath,watchedEvent->{
            //业务逻辑
        },AddWatchMode.PERSISTENT);
```

## 客户端

zookeeper的客户端本质上也是一个socket客户端，主要分为两个线程，一个是sendThread，主要用于发送和接受消息。另一个则是eventThread，就是用来处理诸如watcher的回调信息的。

当我们通过getData方法添加了一个watcher时，底层做了两件事情：

一是在request中标识watch为true，告诉服务端我对于这个路径注册了watcher

![](http://picgo.qisiii.asia/post/09-15-27-35-image.png)

二是当请求执行完的时候，将watcher添加到watchManager中的集合中去。

![](http://picgo.qisiii.asia/post/09-15-25-15-image.png)

![](http://picgo.qisiii.asia/post/09-15-26-20-image.png)

![](http://picgo.qisiii.asia/post/09-15-26-41-image.png)

当收到服务端回调的时候，本质上是一个code为NOTIFICATION_XID的消息，在sendThread接收到消息后，会通过调用EventThread的queueEvent方法通过队列解耦，让eventThread去异步处理事件

![](http://picgo.qisiii.asia/post/09-15-33-10-image.png)

```java
private void queueEvent(WatchedEvent event, Set<Watcher> materializedWatchers) {
            if (event.getType() == EventType.None && sessionState == event.getState()) {
                return;
            }
            sessionState = event.getState();
            final Set<Watcher> watchers;
            if (materializedWatchers == null) {
                // 会根据路径找到本地存储的对应的watcher
                watchers = watchManager.materialize(event.getState(), event.getType(), event.getPath());
            } else {
                watchers = new HashSet<>(materializedWatchers);
            }
            WatcherSetEventPair pair = new WatcherSetEventPair(watchers, event);
            // 添加到队列去异步处理
            waitingEvents.add(pair);
        }
```

![](http://picgo.qisiii.asia/post/09-15-30-25-image.png)

![](http://picgo.qisiii.asia/post/09-15-31-07-image.png)

## 服务端

服务端核心的类是WatchManager。在DataTree类中实例化了两个对象，分别是dataWatchers（对应getData)和childWatches(对应getChildren)

![](http://picgo.qisiii.asia/post/09-14-35-58-image.png)

而服务端又将watcher分为三种类型:普通、持久、持久且递归，后两种只能通过zookeeper.addWatch来指定

```java
public enum WatcherMode {
    STANDARD(false, false),
    PERSISTENT(true, false),
    PERSISTENT_RECURSIVE(true, true),
    ;
```

当接收到getData、exist、getChildren、setWatcher的请求时,会调用WatchManager的两个addWatch重载方法中的一个，将其添加到watchTable的map中，key为nodePath，value是watcher的集合。![](http://picgo.qisiii.asia/post/09-14-38-51-image.png)

这里需要强调一下！map中的value虽然类型是Watcher，但是其本质上确实一个socket连接，即客户端和服务端的连接。

以getData为例追溯一下这个watcher的赋值

![](http://picgo.qisiii.asia/post/09-14-56-41-image.png)

![](http://picgo.qisiii.asia/post/09-14-58-24-image.png)

这里又对应上客户端组装request时，给watch赋值的情况![](http://picgo.qisiii.asia/post/09-15-02-20-image.png)

当发生了对应的事件，比如createNode、deleteNode、setData等变更事件时，会调用WatchManager.triggerWatch方法来触达

![](http://picgo.qisiii.asia/post/09-14-45-17-image.png)

这里先了解一下WatchStats是什么，其实就是各个模式的组合，然后看源码

![](http://picgo.qisiii.asia/post/09-15-04-10-image.png)

```java
public WatcherOrBitSet triggerWatch(String path, EventType type, long zxid, WatcherOrBitSet supress) {
        WatchedEvent e = new WatchedEvent(type, KeeperState.SyncConnected, path, zxid);
        Set<Watcher> watchers = new HashSet<>();
        synchronized (this) {
            PathParentIterator pathParentIterator = getPathParentIterator(path);
            for (String localPath : pathParentIterator.asIterable()) {
                //取出path对应的watcher
                Set<Watcher> thisWatchers = watchTable.get(localPath);
                if (thisWatchers == null || thisWatchers.isEmpty()) {
                    continue;
                }
                Iterator<Watcher> iterator = thisWatchers.iterator();
                while (iterator.hasNext()) {
                    Watcher watcher = iterator.next();
                    //取出watch对应的监听模式
                    Map<String, WatchStats> paths = watch2Paths.getOrDefault(watcher, Collections.emptyMap());
                    WatchStats stats = paths.get(localPath);
                    if (stats == null) {
                        LOG.warn("inconsistent watch table for watcher {}, {} not in path list", watcher, localPath);
                        continue;
                    }
                    if (!pathParentIterator.atParentPath()) {
                        //添加到待处理的集合
                        watchers.add(watcher);
                        //移除标准模式的watcher，这也是为什么普通的watcher是一次性的原因
                        WatchStats newStats = stats.removeMode(WatcherMode.STANDARD);
                        //如果完全没有任何模式的监听，则移除该watcher
                        if (newStats == WatchStats.NONE) {
                            iterator.remove();
                            paths.remove(localPath);
                        } else if (newStats != stats) {
                            paths.put(localPath, newStats);
                        }
                        //处理持久且递归的情况
                    } else if (stats.hasMode(WatcherMode.PERSISTENT_RECURSIVE)) {
                        watchers.add(watcher);
                    }
                }
                if (thisWatchers.isEmpty()) {
                    watchTable.remove(localPath);
                }
            }
        }
        ...
        for (Watcher w : watchers) {
            if (supress != null && supress.contains(w)) {
                continue;
            }
            //通知客户端
            w.process(e);
        }
//最终其实就是发送了一个code为NOTIFICATION_XID的消息到客户端
public void process(WatchedEvent event) {
        ReplyHeader h = new ReplyHeader(ClientCnxn.NOTIFICATION_XID, event.getZxid(), 0);
        WatcherEvent e = event.getWrapper();
        int responseSize = sendResponse(h, e, "notification", null, null, ZooDefs.OpCode.error);
        ServerMetrics.getMetrics().WATCH_BYTES.add(responseSize);
    }
```

至此，和客户端完成闭环。
