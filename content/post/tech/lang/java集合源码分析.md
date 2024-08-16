+++
title = "Java集合源码分析"
description = "Hashmap、ConcurrentHashMap、CopyOnWriteArrayList源码分析"
date = 2022-02-16
image = ""
draft = false
slug = "JavaCollection"
tags = ['Java']
categories = ['tech']
+++

## hashmap源码分析

### 构造函数

initialCapacity 初始化大小，一般最好2次幂

```
public HashMap(int initialCapacity, float loadFactor) {
//初始化值默认是16
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    //The load factor for the hash table. 扩载因子，默认0.75f
    this.loadFactor = loadFactor;
    //The next size value at which to resize (capacity * load factor).
    this.threshold = tableSizeFor(initialCapacity);
}

//通过其他map初始化新的map
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

### tableSizeFor

```
/**
 * Returns a power of two size for the given target capacity.
 */
 //生成一个大于等于 c 且为 2 的 N 次方的最小整数
static final int tableSizeFor(int cap) {
//减一是为了已经是2的幂次方的统一应对
    int n = cap - 1;
    //这一段的本质是将首位及其后面的数字都变为1，然后+1
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

### put()

```
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
先了解一下hash
static final int hash(Object key) {
    int h;
    //高16为与低16为做异或
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
//集合数组下标获取方式
if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
```

### hash()

一般称之为扰动函数，可以看出hash表进行存储的时候，用的是key的hash值，理论上是32位值，但数组没那么大呀，默认也才16，所以通过数组下标获取方式的源码来看

```
 //所以是拿map的空间对hash掩码，
 tab[i = (n - 1) & hash]
```

![](http://picgo.qisiii.asia/post/11-09-55-34-image.png)

![](http://picgo.qisiii.asia/post/11-09-56-00-image.png)

我们可以将32位与n-1（其实刚好是掩码）做与操作，但是这样存储，会严重依赖于低位的信息，具体内容可以看[HashMap中的hash函数 - 淡腾的枫 - 博客园](https://www.cnblogs.com/zhengwang/p/8136164.html)，所以官方在与之前，先让高位和低位进行异或，增加了随机性

### putVal()

将链表转化为树

*TREEIFY_THRESHOLD=8*

```
//evict为false则表示处于创建中，查看调用方就只有clone，new Map(otherMap),readObject时是false
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    //数组加链表结构
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
    //调用resize初始化
        n = (tab = resize()).length;
        //位置为空就直接塞进去，多线程的时候可能会导致值被覆盖
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        //同key覆盖value
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //如果是红黑树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
        //就只能是链表了
            for (int binCount = 0; ; ++binCount) {
            //节点的next为null，表示接着往后面插入
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //长度达到指定值，转为树，但是在treeifyBin中如果数组长度达不到64，那么只是扩容，而不是转为树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //和某个节点和hash值相同，key也相同，那就证明插过了，跳出循环，重新赋值，所以get的时候，不可能只是hash
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //迭代器用的，记录结构修改次数，使用该值的迭代器，遍历时不能进行增删动作
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

```
//get节点
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

### putMapEntries()

```
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        //为空的时候初始化
        if (table == null) { // pre-size
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        //过小的话就需要扩容
        else if (s > threshold)
            resize();
        //依次插入
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

### resize()

```
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    //原有空间大于0，则扩大一倍
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //剩下的都是初始化，指定为n，n!=0
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    //n为0，默认初始化
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    //修复大小
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                //普通节点往数组扔，可能会移动，主要看首位是1还是0，0的话不动，1的话+newCap
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                //这是1.8版本优化之后的，1.7版本采用头插法，会在并发的时候导致链表成环，在get的时候就会死循环
                //两个链表，一个链表用于存放还在原来桶的节点，一个链表用于放已经在扩容的桶的节点

                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        //将链表根据新旧桶区分，并先串起来
                        //参考文档 https://blog.csdn.net/Unknownfuture/article/details/105181447
                        //这里e.hash&oldCap就可以，是因为扩容前&的都是oldCap-1，比如oldCap为8，则&的为111，那么hash落入的必然是0-7，但是现在&的是1000，就变成了会落到0-7或者8-15中，简单的一个首位，就可以区分两个链表
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    //将两个链表的头放入桶中
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

![](http://picgo.qisiii.asia/post/11-09-57-47-image.png)

![](http://picgo.qisiii.asia/post/11-09-57-59-image.png)

### treeifyBin

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //node的总数过少，只会扩容，不会转树
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;//根节点和尾结点
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;//设置根节点
            else {
                p.prev = tl;//互相指向
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        // 到目前为止 也只是把Node对象转换成了TreeNode对象，把单向链表转换成了双向链表
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

## LinkedHashMap

[Map 综述(二):彻头彻尾理解 LinkedHashMap_Rico's Blogs-CSDN博客_linkedhashmap](https://blog.csdn.net/justloveyou_/article/details/71713781)

额外维护了一个双向链表，所以可以记录顺序

```
//新建节点会调子类方法
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}
//将当前节点加到尾结点后面
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
//删除节点之后
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    //删除节点后，连接前置和后驱节点
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}

//在访问之后，比如是访问顺序的情况下，即accessOrder为true；
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        //关联e节点的前驱和后继
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        //将e节点放到最后
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}


void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    //通过重写可以实现LRU算法
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
```

## ConcurrentHashMap

> volatile的数组只针对数组的引用具有volatile的语义，而不是它的元素

### putval()

```
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
            //查看是否有值，没值的话通过CAS保证写入
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //正在扩容，所以头结点的hash会为-1（MOVED），这个时候要去帮助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            //cas也失败了，只能使用synchronized锁来
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

### addCount()

```
//新增元素时，也就是在调用 putVal 方法后，为了通用，增加了个 check 入参，用于指定是否可能会出现扩容的情况
//check >= 0 即为可能出现扩容的情况，例如 putVal方法中的调用
private final void addCount(long x, int check){
    ... ...
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        //检查当前集合元素个数 s 是否达到扩容阈值 sizeCtl ，扩容时 sizeCtl 为负数，依旧成立，同时还得满足数组非空且数组长度不能大于允许的数组最大长度这两个条件才能继续
        //这个 while 循环除了判断是否达到阈值从而进行扩容操作之外还有一个作用就是当一条线程完成自己的迁移任务后，如果集合还在扩容，则会继续循环，继续加入扩容大军，申请后面的迁移任务
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null && (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            // sc < 0 说明集合正在扩容当中
            if (sc < 0) {
                //判断扩容是否结束或者并发扩容线程数是否已达最大值，如果是的话直接结束while循环
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS || (nt = nextTable) == null || transferIndex <= 0)
                    break;
                //扩容还未结束，并且允许扩容线程加入，此时加入扩容大军中
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            //如果集合还未处于扩容状态中，则进入扩容方法，并首先初始化 nextTab 数组，也就是新数组
            //(rs << RESIZE_STAMP_SHIFT) + 2 为首个扩容线程所设置的特定值，后面扩容时会根据线程是否为这个值来确定是否为最后一个线程
            else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
```

### helpTransfer（）

```
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 如果 table 不是空 且 node 节点是转移类型，数据检验
    // 且 node 节点的 nextTable（新 table） 不是空，同样也是数据校验
    // 尝试帮助扩容
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        // 如果 nextTab 没有被并发修改 且 tab 也没有被并发修改
        // 且 sizeCtl  < 0 （说明还在扩容）
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

### transfer()

```
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    //每个线程需要处理的桶的数量，不清楚这里为什么要除以8
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    //这里节点hash改为了MOVE
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            //当i没有走到bound时，表明当前都还是一批
            if (--i >= bound || finishing)
                advance = false;
            //跳出循环，表明迁移完了，会用到下面i<0的条件
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //这里实际上是从n开始逆序分配，一次减少指定步长，然后在for循环中处理这一批
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
                    //i<0存在多种情况，比如单个线程单次干完了，再次分配的时候，发现TRANSFERINDEX还没到0，就又分到了一批，因此还得继续
            //多个线程都分配了，当前线程尝试再次分配的时候，发现，i已经为-1了，这个时候，这个线程就可以撤了
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            //如果全部结束，table要更新，阈值也要更新
            if (finishing) {
                nextTable = null;
                table = nextTab;
                //1.5n=2n-0.5n，实际是0.75*2n
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            //所以能到这里的，都是自己活干完的线程，来这里-1；
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        //这里是直接移动到新数组
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)
        //已经被线程处理了
            advance = true; // already processed
        else {
        //这里会锁节点，所以实际过程中，put和扩容会抢占，由于任意一方可能更改，如果是put则可能是新值，如果是扩容，则可能变为MOVE，因此拿到锁之后要先检查一遍是否和原来相等
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        //lastRun的机制为最后一次变动的地方，这样没有变动的节点就可以直接拼在下面的ln或者hn中了
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            //只有和之前不一样，lastRun才会变动
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        //逆序构建两个链表，但是由于lastRun的存在，持有lastRun节点的链表，lastRun前面是逆序，后面是正序
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            //和hashmap同理
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        //将两个链表放到新数组的两个位置，同时标记旧数组当前节点为MOVE，虽然当前节点已经迁移成功了，但是要呼唤其他线程来帮忙
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    //暂不分析
                    else if (f instanceof TreeBin) {

                    }
                }
            }
        }
    }
}
```

### 为什么不需要锁？

在1.8中ConcurrentHashMap的get操作全程不需要加锁，这也是它比其他并发集合比如hashtable、用Collections.synchronizedMap()包装的hashmap;安全效率高的原因之一

get操作全程不需要加锁是因为Node的成员val是用volatile修饰的和数组用volatile修饰没有关系。

数组用volatile修饰主要是保证在数组扩容的时候保证可见性。

## CopyOnWriteArrayList

ArrayList是线程不安全的，看add方法就知道了，很有可能多个线程取到的size都是相同的，然后又同时更新了

```SQL
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

CopyOnWrite字如其名，当存在写的时候，会先将数据复制一份出来，然后对复制的进行修改，然后将引用指过去。

为啥这么设计呢？因为这样和读可以不冲突了，而写的时候都是通过加锁

```SQL
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
private E get(Object[] a, int index) {
    return (E) a[index];
}

public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        //是不是最后一个
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
        //如果不是，则先复制前半分，再复制后半份
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

## 参考资料

[【搞定Java8新特性】之Java7/8 中的 HashMap 和 ConcurrentHashMap 全解析_Java学习-CSDN博客](https://blog.csdn.net/pcwl1206/article/details/85040309)

[疫苗：Java HashMap的死循环 | 酷 壳 - CoolShell](https://coolshell.cn/articles/9606.html)

[深入理解HashMap(三)resize方法解析_BodyCoding-CSDN博客_hashmap resize()](https://blog.csdn.net/weixin_41565013/article/details/93190786)

[HashMap的最大容量为什么是2的30次方(1左移30)?_与望-CSDN博客](https://blog.csdn.net/sayWhat_sayHello/article/details/83120324)

[ConcurrentHashMap1.8 - 扩容详解_concurrenthashmap1.8的扩容机制-CSDN博客](https://blog.csdn.net/ZOKEKAI/article/details/90051567)

[ConcurrentHashMap之transfer()扩容深入源码分析-CSDN博客](https://blog.csdn.net/Unknownfuture/article/details/105369019)
