+++
title = 'Zookeeper源码分析-选举机制'
date = 2024-08-09T18:01:30+08:00
draft = false
tags = ['中间件','注册中心','zookeeper']
categories = ['tech']
slug ="election"
+++

## 部署

在单机上部署伪集群，需要根据不同的配置文件启动不同的进程。

```shell
#这是zoo1.cfg的内容
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/opt/soft/zookeeper/data/1
clientPort=2181
#第一个端口号用于zk集群同步数据，第二个端口用于重新选举
server.1=127.0.0.1:2287:3387 
server.2=127.0.0.1:2288:3388
server.3=127.0.0.1:2289:3389
#zoo2.cfg需要修改两个地方
clientPort=2182
dataDir=/opt/soft/zookeeper/data/2
#zoo3.cfg需要修改两个地方
clientPort=2183
dataDir=/opt/soft/zookeeper/data/3
```

在三个data目录下创建myid文件，内容只有单个数字代表当前zookeeper节点的myid,对应上面配置里server.id的id

```
touch 1/myid
echo 1 >1/myid
echo 2 >2/myid
echo 3 >3/myid
```

![](http://picgo.qisiii.asia/post/09-18-03-28-image.png)

```shell
#启动三个进程
./apache-zookeeper-3.7.1-bin/bin/zkServer.sh start conf/zoo1.cfg
./apache-zookeeper-3.7.1-bin/bin/zkServer.sh start conf/zoo2.cfg
./apache-zookeeper-3.7.1-bin/bin/zkServer.sh start conf/zoo3.cfg
#然后可以查看三个节点的状态
```

![](http://picgo.qisiii.asia/post/09-18-01-36-image.png)

## 选举逻辑：

![](http://picgo.qisiii.asia/post/13-10-01-00-image.png)

集群模式的核心类是QuorumPeer，其启动方法中核心的就只有startLeaderElection和super.start这两个方法的调用

```java
public synchronized void start() {
        if (!getView().containsKey(myid)) {
            throw new RuntimeException("My id " + myid + " not in the peer list");
        }
        loadDataBase();
        //同单机逻辑一样
        startServerCnxnFactory();
        try {
            adminServer.start();
        } catch (AdminServerException e) {
            LOG.warn("Problem starting AdminServer", e);
        }
        //创建QuorumCnxManager和FastLeaderElection两个类，一个是管理连接，一个是处理选举时收发消息
        startLeaderElection();
        startJvmPauseMonitor();
        //真正的选举逻辑在这里，QuorumPeer也是个线程，所以要看run的逻辑
        super.start();
    }
```

其中，QuorumCnxManager负责节点之间的通信，节点中两两之间都需要建立连接。

QuorumCnxManager通过给每个节点地址分配了一个ListenerHandler，来接受或者建立连接。ListenerHandler内的SendWorker和RecvWorker则是建立连接后进行真正进行发送和接受消息的类。

FastLeaderElection是专门负责选举工作的，其内部针对于通信有两个组件，分别是workerSender和workerReceiver，这两个类只是负责将选票信息封装为ByteBuffer，然后转发到queueSendMap，由QuorumCnxManager的SendWorker去进行发送。

### 建立连接

首次发起建立连接的地方是在FastLeaderElection发消息前，如果判断没有建立好连接，就要建立连接

```java
public void toSend(Long sid, ByteBuffer b) {
        if (this.mySid == sid) {
            b.position(0);
            //发给自己的消息，直接添加到接受队列，不走网络请求了
            addToRecvQueue(new Message(b.duplicate(), sid));
            /*
             * Otherwise send to the corresponding thread to send.
             */
        } else {
            BlockingQueue<ByteBuffer> bq = queueSendMap.computeIfAbsent(sid, serverId -> new CircularBlockingQueue<>(SEND_CAPACITY));
            addToSendQueue(bq, b);
            //发消息给其他节点之前要建立连接
            connectOne(sid);
        }
    }
synchronized void connectOne(long sid) {
        //如果已经存在连接，直接返回；senderWorkerMap中存在对应的senderWorker类表明已经建立好连接
        if (senderWorkerMap.get(sid) != null) {
            LOG.debug("There is a connection already for server {}", sid);
            if (self.isMultiAddressEnabled() && self.isMultiAddressReachabilityCheckEnabled()) {
                  senderWorkerMap.get(sid).asyncValidateIfSocketIsStillReachable();
            }
            return;
        }
```

建立的逻辑，就不细分析了；主要是通过QuorumConnectionReqThread类去实现的，且当建立成功后会在senderWorkerMap中存储对应的线程，对应上面代码判断存在线程直接返回。

![](http://picgo.qisiii.asia/post/13-10-24-15-image.png)

接受连接请求的地方就是在ListenerHandler中

![](http://picgo.qisiii.asia/post/13-10-26-58-image.png)

由于是每两个节点都要建立连接，且两个节点都可能发起连接，那么以哪个为主呢？zookeeper的设计是以myId大的为主，即只能myid大的发起连接，这样才会判定有效

```java
if (sid < self.getMyId()) {
            SendWorker sw = senderWorkerMap.get(sid);
            if (sw != null) {
                sw.finish();
            }
            /*
             * Now we start a new connection
             */
            LOG.debug("Create new connection to server: {}", sid);
            closeSocket(sock);
            //如果是小id连大id，那就重新建立连接
            if (electionAddr != null) {
                connectOne(sid, electionAddr);
            } else {
                connectOne(sid);
            }

        } else if (sid == self.getMyId()) {
            // we saw this case in ZOOKEEPER-2164
            LOG.warn("We got a connection request from a server with our own ID. "
                     + "This should be either a configuration error, or a bug.");
        } else { // Otherwise start worker threads to receive data.
            SendWorker sw = new SendWorker(sock, sid);
            RecvWorker rw = new RecvWorker(sock, din, sid, sw);
            sw.setRecv(rw);

            SendWorker vsw = senderWorkerMap.get(sid);

            if (vsw != null) {
                vsw.finish();
            }

            senderWorkerMap.put(sid, sw);

            queueSendMap.putIfAbsent(sid, new CircularBlockingQueue<>(SEND_CAPACITY));

            sw.start();
            rw.start();
        }
```

### 选举

ELECTION 选举

Vote 是选票的意思，结构是（myid,LastZxid,currentEpoch）(节点id，最大事务id，当前周期)

Epoch 直译是纪元、时期，我理解是当前周期，类比于第15届领导中的这个15，选票需要表明这个是为哪期领导投票的

ServerState  节点状态

```java
public enum ServerState {
        LOOKING,//正在选举
        FOLLOWING,//节点处理跟从者状态，拥有投票的权利
        LEADING,//节点处于领导状态，拥有投票的权利
        OBSERVING//观察者,节点无投票权利
    }
```

QuorumPeer本身就是一个线程，其真正的选举逻辑也在run方法中，去除掉杂七杂八的逻辑后，其核心逻辑如下：

默认是LOOKING状态，所以会进入到LOOKING的分支：根据readonlymode.enabled决定是否启动一个只读的服务器，然后进入选举，核心代码是`setCurrentVote(makeLEStrategy().lookForLeader());`

当选举结果出来之后，每个节点会更改自身的状态，所以会进入到其他分支，处理不同的逻辑，以Leader为例，会创建一个LeaderZooKeeperServer来接受客户端请求，并做好一些转发工作等，这里先不展开讲。

```java
public void run() {
            while (running) {
                switch (getPeerState()) {
                case LOOKING:
                    LOG.info("LOOKING");
                    ServerMetrics.getMetrics().LOOKING_COUNT.add(1);

                    if (Boolean.getBoolean("readonlymode.enabled")) {
                        LOG.info("Attempting to start ReadOnlyZooKeeperServer");
                        //选举期间启动一个只读的server，不提供写的能力，其processor是ReadOnlyRequestProcessor
                        final ReadOnlyZooKeeperServer roZk = new ReadOnlyZooKeeperServer(logFactory, this, this.zkDb);
                        roZk.startup();
                        try {
                            //发起选举并记录最终成功的选票
                            setCurrentVote(makeLEStrategy().lookForLeader());
                            checkSuspended();
                        } finally {
                            roZkMgr.interrupt();
                            roZk.shutdown();
                        }
                    } else {
                         setCurrentVote(makeLEStrategy().lookForLeader());
                    }
                    break;
                case OBSERVING:
                    try {
                        LOG.info("OBSERVING");
                        setObserver(makeObserver(logFactory));
                        observer.observeLeader();
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception", e);
                    } finally {
                        observer.shutdown();
                        setObserver(null);
                        updateServerState();
                    break;
                case FOLLOWING:
                    try {
                        LOG.info("FOLLOWING");
                        setFollower(makeFollower(logFactory));
                        follower.followLeader();
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception", e);
                    } finally {
                        follower.shutdown();
                        setFollower(null);
                        updateServerState();
                    }
                    break;
                case LEADING:
                    LOG.info("LEADING");
                    try {
                        setLeader(makeLeader(logFactory));
                        leader.lead();
                        setLeader(null);
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception", e);
                    } finally {
                        if (leader != null) {
                            leader.shutdown("Forcing shutdown");
                            setLeader(null);
                        }
                        updateServerState();
                    }
                    break;
                }
            }
        }
```

下面具体分析一下选举逻辑

`makeLEStrategy().lookForLeader()`实际上是`FastLeaderElection.lookForLeader`

#### 选票PK逻辑：

1. 周期大的为准

2. 同周期的情况下，以事务Id大的为准

3. 同周期同事务的情况下，以myid大的为准

```java
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
        //myid对应的权重为0，表示不算选票
        if (self.getQuorumVerifier().getWeight(newId) == 0) {
            return false;
        }
        return ((newEpoch > curEpoch)
                || ((newEpoch == curEpoch)
                    && ((newZxid > curZxid)
                        || ((newZxid == curZxid)
                            && (newId > curId)))));
    }
```

#### lookForLeader流程图：

![](http://picgo.qisiii.asia/post/13-11-58-20-image.png)

```java
public Vote lookForLeader() throws InterruptedException {
        try {
            //选举期间每个节点的选票集合
            Map<Long, Vote> recvset = new HashMap<>();
            //选举成功之后的选票存储在这里，是在其他节点发消息时存储的，可以让迟到的节点立马知道谁是领导
            Map<Long, Vote> outofelection = new HashMap<>();

            int notTimeout = minNotificationInterval;

            synchronized (this) {
                //选举周期的具体值，用于和其他节点选票中的周期比较
                logicalclock.incrementAndGet();
                //更新提案-其实就是更新选票
                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
            }
            //发送投票
            sendNotifications();

            SyncedLearnerTracker voteSet = null;

            while ((self.getPeerState() == ServerState.LOOKING) && (!stop)) {
                //收取选票消息
                Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);
                if (n == null) {
                    //假如所有连接的待发送消息都发送了，就可以再次尝试发消息了
                    if (manager.haveDelivered()) {
                        sendNotifications();
                    } else {
                        //没有的话就代表有些连接还没建立，去建立连接
                        manager.connectAll();
                    }

                    notTimeout = Math.min(notTimeout << 1, maxNotificationInterval);

                    //这里没理解这个timeout的作用
                    if (self.getQuorumVerifier() instanceof QuorumOracleMaj
                            && self.getQuorumVerifier().revalidateVoteset(voteSet, notTimeout != minNotificationInterval)) {
                        setPeerState(proposedLeader, voteSet);
                        Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
                        leaveInstance(endVote);
                        return endVote;
                    }
                } else if (validVoter(n.sid) && validVoter(n.leader)) {
                    switch (n.state) {
                    case LOOKING:
                        //当选举周期大于当前节点的周期时，选举周期更新
                        if (n.electionEpoch > logicalclock.get()) {
                            logicalclock.set(n.electionEpoch);
                            recvset.clear();
                            //比拼选票，赢了按我的来，输了按你的来
                            if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                                updateProposal(n.leader, n.zxid, n.peerEpoch);
                            } else {
                                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
                            }
                            sendNotifications();
                            //如果选举周期小于当前节点的周期，直接不予响应
                        } else if (n.electionEpoch < logicalclock.get()) {
                            break;
                            //周期相等就pk吧
                        } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                            sendNotifications();
                        }

                        //记录每个节点的选票
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                        //每次收到消息都要重新统计，正常来讲应该只有一个Tacker
                        voteSet = getVoteTracker(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch));
                        //有结果了，值超过半数票
                        if (voteSet.hasAllQuorums()) {
                            //当不再有新消息时，表明选举成功；当还有新消息时，判断投票是否有效，有效的话再次轮回
                            // Verify if there is any change in the proposed leader
                            while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {
                                //最终校验
                                if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                                    recvqueue.put(n);
                                    break;
                                }
                            }
                            //不再有新的比当前领导票更新的消息了，那就以当前领导为准
                            if (n == null) {
                                //设置节点状态
                                setPeerState(proposedLeader, voteSet);
                                Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
                                leaveInstance(endVote);
                                return endVote;
                            }
                        }
                        break;
                    case OBSERVING:
                        break;
                    case FOLLOWING:
                        //收到follower的消息，有follower就代表有leader了,所以更新自己的状态
                        Vote resultFN = receivedFollowingNotification(recvset, outofelection, voteSet, n);
                        if (resultFN == null) {
                            break;
                        } else {
                            return resultFN;
                        }
                    case LEADING:
                        //收到leader的消息，表明半数人已经同意了，直接更新自己的leader
                        Vote resultLN = receivedLeadingNotification(recvset, outofelection, voteSet, n);
```

#### VoteTracker

该类是选票记录器，主要工作是在当前节点确定了自己的选票后，统计所有节点选票和自己选票相符的数量，可以理解为画正

![](http://picgo.qisiii.asia/post/13-12-09-29-image.png)

```java
protected SyncedLearnerTracker getVoteTracker(Map<Long, Vote> votes, Vote vote) {
        //这里原则上其实只有1个的，但是假如LastSeenQuorumVerifier有值，就也要参考
        SyncedLearnerTracker voteSet = new SyncedLearnerTracker();
        voteSet.addQuorumVerifier(self.getQuorumVerifier());
        if (self.getLastSeenQuorumVerifier() != null
            && self.getLastSeenQuorumVerifier().getVersion() > self.getQuorumVerifier().getVersion()) {
            voteSet.addQuorumVerifier(self.getLastSeenQuorumVerifier());
        }
        //比如我的选票投给2，那么统计所有选票为2的节点
        for (Map.Entry<Long, Vote> entry : votes.entrySet()) {
            if (vote.equals(entry.getValue())) {
                voteSet.addAck(entry.getKey());
            }
        }
        return voteSet;
    }
    //判断同意的票数是否大于一半
    public boolean hasAllQuorums() {
        for (QuorumVerifierAcksetPair qvAckset : qvAcksetPairs) {
            if (!qvAckset.getQuorumVerifier().containsQuorum(qvAckset.getAckset())) {
                return false;
            }
        }
        return true;
    }
    public boolean containsQuorum(Set<Long> ackSet) {
        //超过半数
        return (ackSet.size() > half);
    }
```

### ReadOnlyZookeeperServer

当处于选举状态时，假如`readonlymode.enabled`为true，那么就创建一个只读的服务器，该服务器只提供查询，不能写入。

在[zookeeper学习-单机模式]({{<ref "zookeeper学习">}})中，我们知道zkserver是由三个processor处理的，其中SyncRequestProcessor是处理事务的，将事务写入日志里。而在只读模式下可以看到，不存在这个processor，所以只能查，不能写。

```java
protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        RequestProcessor prepProcessor = new PrepRequestProcessor(this, finalProcessor);
        ((PrepRequestProcessor) prepProcessor).start();
        firstProcessor = new ReadOnlyRequestProcessor(this, prepProcessor);
        ((ReadOnlyRequestProcessor) firstProcessor).start();
    }
```

![](http://picgo.qisiii.asia/post/13-12-21-29-image.png)

对于所有的写入命令，都直接抛异常

![](http://picgo.qisiii.asia/post/13-12-23-24-image.png)

当然了，这也只是选举期间的一个临时措施，在选举结束后会关闭该服务器，并创建对应节点状态的服务器。

## 参考文档:

[Zookeeper系列 - Leader选举_zk集群查看哪个节点是leader-CSDN博客](https://blog.csdn.net/y510662669/article/details/106624186)

[ZooKeeper: Because Coordinating Distributed Systems is a Zoo](https://zookeeper.apache.org/doc/current/zookeeperStarted.html)

[zookeeper（单机、伪集群、集群）部署-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1610867)

[Zookeeper的安装和使用 - 商商-77 - 博客园](https://www.cnblogs.com/umgsai/p/5835825.html)

[【Zookeeper源码阅读】leader选举源码分析 - 掘金](https://juejin.cn/post/6989543523213115406#heading-22)

[【图解源码】Zookeeper3.7源码剖析，Session的管理机制，Leader选举投票规则，集群数据同步流程 - 掘金](https://juejin.cn/post/7112681401601769486)
