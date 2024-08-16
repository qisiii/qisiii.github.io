+++
title = "JavaLock类源码分析"
description = ""
date = 2022-02-14
image = ""
draft = false
slug = "Javalock"
tags = ['java','lock']
categories = ['tech']
+++

## Lock和Condition接口

Lock是java中显示使用锁的方案，相比于synchronized，要更加灵活；

synchronized在可重入的情况下，加锁和解锁是相反的顺序，但是Lock可以以任意的顺序解锁；

而且synchronized必须限定在同一范围内使用，lock则可以在不同的范围；

另外，synchronized仅限于对同一资源的独占的处理，而lock的实现机制则可以存在共享锁的存在，如ReentrantReadWriteLock。

通常的使用如下面的代码一般，使用lock和unlock方法

```Java
 Lock l = ...;
 l.lock();
 try {
   // access the resource protected by this lock
 } finally {
   l.unlock();
 }
```

Condition则是一种线程间通信的方案，是对于Object的wait和notify的替代。Condition实例由Lock.newCondition产生，而且Condition的使用一般都在lock和unlock中间；

相比于Object的wait和notify来说，condition可以更精确的控制某个线程的唤醒，只需要将condition.await置在需要挂起的线程，那么当任意线程执行condition.signal的时候，则可以精确唤醒该线程；

LockSupport提供了两个方法park和unpark，等同于阻塞和唤醒，但是原理是park是消耗一个凭证，unpark是提供一个凭证；所以也可以先unpark，那么下次调用park就不会阻塞

## AQS

AbstractQueuedSynchronizer其本质是两个队列。

一个是同步队列，双向链表，用于存储等待获取锁的线程，状态和线程封装在NODE类中；

一个是条件队列，单向链表，用于存储await的线程节点，通过nextWaiter连接链表；

### CLH

CLH算法是一个自旋锁的算法，是对于普通的cas的优化。*(Craig, Landin, and Hagersten)*

对于普通的cas来说，如果存在多个线程，那么每个线程都要在循环里尝试cas，这样会给cpu带来比较大的复合；另外有可能导致饥饿，即某个线程一直在等待，无法获取到锁。

CLH持有一个尾结点，添加节点通过prev链接，节点有一个状态默认是true表示正在获取锁或者已经持有锁，线程通过判断前一个节点的状态什么时候变成false或者没有节点才可以获取锁

```Java
public class CLH {
    AtomicReference<Node> tail = new AtomicReference<>();
    ThreadLocal<Node> cur = new ThreadLocal<>();

    static class Node {
        Node prev;
        String name;
        volatile boolean state = true;

        public Node(String name) {
            this.name = name;
        }
    }

    void lock() {
        Node node = cur.get();
        if (node == null) {
            node = new Node(Thread.currentThread().getName());
            cur.set(node);
            boolean tailSet = false;
            while (tail.get() == null) {
                tailSet = tail.compareAndSet(null, node);
            }
            if (!tailSet) {
                node.prev = tail.getAndSet(node);
//                System.out.println(node.name+"前一个是"+node.prev.name);
            } else {
//                System.out.println("首节点是"+node.name);
            }
        }
        do {
//            System.out.println(Thread.currentThread().getName() + "尝试获取锁");
        } while (node.prev != null && node.prev.state);
        System.out.println(Thread.currentThread().getName() + "获取到锁");
    }

    void unlock() {
        System.out.println(Thread.currentThread().getName() + "释放锁");
        Node node = cur.get();
        node.state = false;
        cur.remove();
    }

    public static void main(String[] args) {
        CLH clh = new CLH();
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 10; i++) {
            executorService.submit(() -> {
                clh.lock();
                try {
                    Thread.sleep(1000);
                    clh.unlock();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
        }
        executorService.shutdown();
    }
}
```

### 同步队列

在了解同步队列运行机制前，先看一下NODE类

```Java
static final class Node {    
    //用于区分节点是共享还是独占
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;    
   //waitStatus的四种状态，其中cancel表示节点取消（作废）
    static final int CANCELLED =  1;
    //signal用于唤醒下一个需要获取锁的线程（区别于CLH的设计）
    static final int SIGNAL    = -1;
    //condition条件则是条件队里应用的，表示该节点目前是条件队里的一员
    static final int CONDITION = -2;
    //PROPAGATE是在共享锁用到,当单个节点没有获取完锁，后续的节点可以继续获取锁
    static final int PROPAGATE = -3;
    volatile int waitStatus;    //用于处理取消的节点，将未取消的节点重新挂载在可用节点后
    volatile Node prev;    //用于唤醒机制，即head.next才是将要获取锁的
    volatile Node next;    //当前工作线程
    volatile Thread thread;    //用于标识当前节点在条件队列的位置
    Node nextWaiter;
```

然后通过下图，我们了解一下aqs的机制

1. 没有任何线程获取锁，这个时候不存在节点，头即是尾

2. Thread1和Thread2通过尝试获取锁，假设Thread1获取锁成功，那么Thread2则会执行addWaiter方法，在这个方法中，初始化了head节点，通过将Thread2构造的节点作为head.next，并且作为tail；然后会执行acquireQueued方法，在这里会将Thread2.prev也就是head的waitStatus置为signal(-1)，然后会挂起当前线程；

3. Thread3和Thread4尝试获取锁，假设Thread4获取成功，那么Thread4会追加到Thread2，并临时作为tail；而Thread3由于在addWaiter方法中compareAndSetTail失败，所以会通过enq方法追加到Thread4后面，并作为Tail；同理，Thread2和Thread4的waitStatus都会被后一个节点设置为signal；

4. 这一步则是释放锁，当释放锁的时候，会判断如果head节点的waitStatus如果不等于0，然后就唤醒下一个可用的节点----Thread2，因为在acquireQueued方法开始继续执行，然后尝试获取锁，获取成功的话，就将自己设置为了head；

![](http://picgo.qisiii.asia/post/11-09-08-14-image.png)

```Java
public final void acquire(int arg) {
    //tryAcquire是个钩子方法，由具体的实现类实现
    if (!tryAcquire(arg) &&
    //addWaiter则是构建了一个node并且添加到队列尾部；
    //acquireQueued则是将当前工作线程阻塞在死循环里等待唤醒；
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    //队列有内容，则使用cas尝试追加到尾部
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //并发情况下cas只有一个能成功，所以需要enq来处理
    enq(node);
    return node;
}
//enq负责两件事，一是初始化头，二是保证尾部追加
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
        //当前一个节点是head的时候，尝试获取锁
        //两种情况，一是第一次进来的时候，有可能head刚好释放锁；另一种是从阻塞中唤醒；
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //shouldParkAfterFailedAcquire主要是设置node的前一个可用节点为signal
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

public final boolean release(int arg) {
//tryRelease也是钩子方法
    if (tryRelease(arg)) {
        Node h = head;
        //只要head是signal，那么就要唤醒下一个可用节点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

### 条件队列

条件队列是condition相关的队列，先来看一下aqs中的ConditionObject对象

```Java
public class ConditionObject implements Condition, java.io.Serializable {
    private static final long serialVersionUID = 1173984872572414699L;
    /** First node of condition queue. */
    private transient Node firstWaiter;
    /** Last node of condition queue. */
    private transient Node lastWaiter;
    }
```

可以看出ConditionObject只持有首尾，而串联起来Node的则是Node的nextWaiter字段，所以这是个单链表

![](http://picgo.qisiii.asia/post/11-09-09-00-image.png)这是上面同步队列最后一步时的状态

1. Thread2调用await，会构建一个Thread2的condition节点（waitStatus为condition）作为firstWaiter，同时会释放Thread2已经持有的锁，唤醒Thread2.next（Thread4），因此同步队列里Thread4会变为头结点；

2. Thread4调用await，同Thread2一样，会新构建一个condition节点，然后→Thread2.nextWaiter指向Thread4，此时Thread4就是lastWaiter，同步队列也就只剩Thread3了

3. Thread2调用signal，将条件队列最靠前的可用的节点移出条件队列，重新追加到同步队列中去，将本来的tail节点状态置为signal

```Java
//阻塞
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
        //构建节点，追加到条件队列
    Node node = addConditionWaiter();
    //释放锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    //当确定不在同步队列的时候，阻塞线程；在该方法里isOnSyncQueue一定是，因为构建的节点status是condition
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //重新获取锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    //构建新节点，要么作为头，要么追加到尾
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}

//唤醒
private void doSignal(Node first) {
    do {
    //指针循环往下指
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
            //移出条件队列
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}


final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    //重新加到同步队列里面
    Node p = enq(node);
    int ws = p.waitStatus;
    //修改本来的tail节点的状态
    //如果取消或者设置不对,立即唤醒
    //但这里和同步队列唤醒机制不一样，因为线程是park在await中，而不是acquireQueue
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}

private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    //和dosignal的区别就是不作为条件，而是放在循环里了，所以是唤醒所有的条件及诶单
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```

### 共享锁

主要是有个传递机制，其他没啥太大的区别，一旦有锁释放了，那么后面的就会改为传递

```Java
private void doAcquireShared(int arg) {
    //追加节点到同步队列
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            //如果是head.next则尝试获取锁
            if (p == head) {
            //这里锁的结果；r<0代表没有获取到；r=0代表获取了所有的锁；
            //r>0则代表获取完之后还有锁
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    //如果获取完之后还有锁，则继续让后面的节点获取锁
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            //等同于独占锁，获取不到则设置signal，然后挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}


private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    ...
    //1.propagate > 0 意味着还有锁，则表示可以继续获取锁
    //2.h==null和h==head==null是预防空指针，实际上不太可能存在
    //3.h可能是新头或者旧头，但是如果waitStatus小于0，三种情况
    //Cancel表示头结点取消了，那么肯定释放了锁，所以要继续获取锁
    //Signal表示当前节点获取了锁（是最正常的状态），但是由于还有锁，所以可以继续往后
    //PROPAGATE表示有其他线程调用了doReleaseShared，因为只有这里会将头结点设置为PROPAGATE，所以有地方调用表明释放了线程，也可以继续获取锁
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}


private void doReleaseShared() {
    for (;;) {
        Node h = head;
        //从头结点开始释放锁
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            //如果头结点是signnal，那么就释放头结点的锁
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            //如果不是那证明别的线程可能释放了，就将后继节点改为可以获取锁，证明锁又空闲了
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

## ReentrantLock

本质是AQS，不过实现了公平锁和非公平锁，默认是非公平锁。

```Java
//默认非公平
public ReentrantLock() {
    sync = new NonfairSync();
}
lock方法不公平就体现在任何线程来了就先尝试获取锁，并且不检查是否需要排队
//NoFair
final void lock() {
    acquire(1);
}

//Fair
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}

protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
    //必须得是head.next才能尝试获取锁，所以保证了公平
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

## ReentrantReadWriteLock

读写锁的设计是将int分为高低16位，高16为读锁，低16位为写锁。

如果是

00000000 0000010 00000000 00000010

说明当前线程获得两次读锁，两次写锁

```Java
static final int SHARED_SHIFT   = 16;
//65536,其实就是高位1
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
//65535 0（16）1（16）
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

/** 无符号右移16位，即只要高16位  */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** 与上0000000000000000 1111111111111111，即只要低16位  */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

相比于ReentrantLock多了两个内部类

ReadLock和WriteLock

//当前线程持有写锁，其他线程都不能持有写锁，但是当前线程还可以持有读锁

//当前线程持有读锁，其他线程也可以获取读锁

```TypeScript
//写锁仍然是独占
public void lock() {
    sync.acquire(1);
}

//写锁的lock
final boolean tryWriteLock() {
    Thread current = Thread.currentThread();
    int c = getState();
    if (c != 0) {
        int w = exclusiveCount(c);
        //如果c不为0，但是w为0，则表明当前存在读锁（即共享锁），存在读锁时，任何线程无法获取写锁
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
    }
    if (!compareAndSetState(c, c + 1))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}

//写锁的释放
//由于写锁是低16位，所以直接做减法就可以了
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}


//读锁是共享锁
public void lock() {
    sync.acquireShared(1);
}


//共享锁获取
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
   //如果有独占锁，且不是当前线程，则不能获取共享锁；
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
        //获取目前的共享锁
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        //原有的锁加上高位1
        compareAndSetState(c, c + SHARED_UNIT)) {
        //下面的部分主要是记录哪些线程获取了几次锁
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}



protected final boolean tryAcquire(int acquires) {
    /*
     * Walkthrough:
     * 1. If read count nonzero or write count nonzero
     *    and owner is a different thread, fail.
     * 2. If count would saturate, fail. (This can only
     *    happen if count is already nonzero.)
     * 3. Otherwise, this thread is eligible for lock if
     *    it is either a reentrant acquire or
     *    queue policy allows it. If so, update state
     *    and set owner.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
//共享锁释放
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    //减少线程持有的锁的数量
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    //减少真正的锁的数量
    for (;;) {
        int c = getState();
        //现有锁减去高位1
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```

## 参考文档:

[AQS深入理解 shouldParkAfterFailedAcquire源码分析 状态为0或PROPAGATE的情况分析-CSDN博客](https://blog.csdn.net/anlian523/article/details/106448512)

[AQS基础——多图详解CLH锁的原理与实现](https://zhuanlan.zhihu.com/p/197840259)

[Java AQS 核心数据结构-CLH 锁](https://zhuanlan.zhihu.com/p/398582011)

[AQS深入理解 setHeadAndPropagate源码分析 JDK8-CSDN博客](https://blog.csdn.net/anlian523/article/details/106319294)
