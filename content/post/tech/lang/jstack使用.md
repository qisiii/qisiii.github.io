+++
title = 'Jstack使用'
date = 2024-07-28T17:14:47+08:00
draft = false
tags = ['java','tool']
categories = ['tech']
slug="jstack"
+++

## Jstack应用

jstack是java提供的工具，一般用于分析栈-线程相关的问题，常见的比如死锁问题，cpu过高问题。

jstack 命令使用方式如下：

`jstack [-F][-l][-m] pid` 或者`jstack [-F][-l][-m] [server_id@]<remote server IP or hostname`

可以通过-h或者-help来了解每个参数具体的作用，pid指的是java项目所属的进程Id，一般通过jps命令或者top、ps等命令获取。

### Jstack文件分析

#### 参数分析

![](http://picgo.qisiii.asia/post/202407283188d3b762db93252b8fe9318264d476.png)

大部分的信息都如上图所示：

"pool-1-thread-2" 表示线程的名字

#12 猜测是线程的nativeId，和arthas的线程id列很像，但是arthas官网文档又强调不是nativeId

prio表示java内定义的线程的优先级

os_prio操作系统级别的优先级

tid 猜测是一个地址，但具体不清楚

nid 操作系统级别线程的线程id,可通过top -pid [java应用进程id] 获取

下面的红色部分则是状态、调用链路、一些锁的操作信息，具体下面分开讲

#### 状态

Java中线程的状态有六种：NEW、RUNNABLE、BLOCKED、WAITING、TIMED_WAITING、TERMINATED

其中，NEW和TERMINATED是创建和销毁时的状态，在stack中没有见到过，基本上都是其他四种状态在流转。)
![](http://picgo.qisiii.asia/post/20240729b959e899f9a391eecb650ee6a786e976.png)

RUNNABLE:表示当前可以被cpu调度执行或者已经在执行。

为了更好的解释BLOCKED，这里要结合synchronized的底层原理Monitor来进行讲解：
![monitor](https://i.loli.net/2020/05/20/1vf4anVdw8juzt3.png)

Monitor是java对象规定的一个区域，可以视作一个房间。

当持有synchronized锁的时候，表示当前线程持有Monitor，释放锁表示释放Monitor。因此，同一时刻只有一个线程可以持有，这个信息被记录在对象头的markword部分中。

左边的entrySet用于记录等待获取锁的线程（即最开始抢锁失败的线程），此时这些线程的状态就为BLOCKED，抢到锁的线程状态为RUNNABLE。

当拥有锁的线程释放锁的时候，存在两种可能：

一种是释放了并且再也不使用了，即线程工作完成了，此时线程就结束掉，状态变为TERMINATED。

一种是释放了但是线程是核心线程，可能还会继续调用，此时线程状态会为WAITING或者TIMED_WAITING，此时这个线程就会被放入到右边的waitSet。当调用notify或者到指定时间之后，就会将线程重新添加到左边的entrySet中再次尝试获取锁。

#### 线程的操作

线程的操作是指在stack文件堆栈信息中，用于描述线程对锁、对象的一些操作，堆栈信息一般如下面这般，-locked就是线程的操作，分为几类：

```java
java.lang.Thread.State: TIMED_WAITING (sleeping)
 at java.lang.Thread.sleep(Native Method)
 at com.qisi.simple.Jvisualvm.JStackExample.test1(JStackExample.java:18)
 - locked <0x000000076adddbb8> (a com.qisi.simple.Jvisualvm.JStackExample)
 at com.qisi.simple.Jvisualvm.JStackExample$$Lambda$1/777874839.run(Unknown Source)
 at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
 at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
 at java.lang.Thread.run(Thread.java:750)

java.lang.Thread.State: WAITING (parking)
    at sun.misc.Unsafe.park(Native Method)
    - parking to wait for  <0x000000076adddb80> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2044)
    at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
    at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:750)
```

**locked [地址]**：当前线程持有锁，即持有monitor

**waiting on [地址]** ：线程释放锁，等待通知，即在waitSet中

**waiting to lock [地址]**：线程获取锁失败，正在等待获取锁，即在entrySet中

**parking to wait for [地址]**：这种的是属于非synchronized锁的信息，多用于Lock锁，对应park方法，具体参考[multithreading - Java thread dump: Difference between &quot;waiting to lock&quot; and &quot;parking to wait for&quot;? - Stack Overflow](https://stackoverflow.com/questions/11337384/java-thread-dump-difference-between-waiting-to-lock-and-parking-to-wait-for)

#### 例子：

```java
static Boolean flag = true;
    synchronized void producer(){
        System.out.println(Thread.currentThread().getName()+"抢到了锁");
        while (flag) {
            try {
                flag = false;
                Thread.sleep(5000);
                this.wait();
                System.out.println(Thread.currentThread().getName()+"执行了test1");
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }
    synchronized void consumer(){
        System.out.println(Thread.currentThread().getName()+"抢到了锁");
        while (!flag) {
            flag = true;
            this.notify();
        }
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        System.out.println(Thread.currentThread().getName()+"执行了test2");
    }

 ExecutorService executorService = Executors.newFixedThreadPool(2);
        JStackExample example = new JStackExample();
        for (int i = 0; i < 10; i++) {
            executorService.execute(example::producer);
            executorService.execute(example::consumer);
        };
```

一个简单的生产者-消费者例子，生成两次堆栈，分别在producer sleep时和

```java
2024-07-28 21:52:21
Full thread dump OpenJDK 64-Bit Server VM (25.392-b08 mixed mode):

"Attach Listener" #14 daemon prio=9 os_prio=31 tid=0x0000000128811800 nid=0x5b03 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"DestroyJavaVM" #13 prio=5 os_prio=31 tid=0x0000000159922000 nid=0x1503 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"pool-1-thread-2" #12 prio=5 os_prio=31 tid=0x0000000159921000 nid=0x5a03 waiting for monitor entry [0x0000000172c56000]
   java.lang.Thread.State: BLOCKED (on object monitor)
    at com.qisi.simple.Jvisualvm.JStackExample.test2(JStackExample.java:27)
    - waiting to lock <0x000000076adddbb8> (a com.qisi.simple.Jvisualvm.JStackExample)
    at com.qisi.simple.Jvisualvm.JStackExample$$Lambda$2/930990596.run(Unknown Source)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:750)

"pool-1-thread-1" #11 prio=5 os_prio=31 tid=0x00000001598dc800 nid=0x5903 waiting on condition [0x0000000172a4a000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
    at java.lang.Thread.sleep(Native Method)
    at com.qisi.simple.Jvisualvm.JStackExample.test1(JStackExample.java:18)
    - locked <0x000000076adddbb8> (a com.qisi.simple.Jvisualvm.JStackExample)
    at com.qisi.simple.Jvisualvm.JStackExample$$Lambda$1/777874839.run(Unknown Source)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:750)


2024-07-28 21:52:26
Full thread dump OpenJDK 64-Bit Server VM (25.392-b08 mixed mode):

"pool-1-thread-2" #12 prio=5 os_prio=31 tid=0x0000000159921000 nid=0x5a03 waiting on condition [0x0000000172c56000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
    at java.lang.Thread.sleep(Native Method)
    at com.qisi.simple.Jvisualvm.JStackExample.test2(JStackExample.java:33)
    - locked <0x000000076adddbb8> (a com.qisi.simple.Jvisualvm.JStackExample)
    at com.qisi.simple.Jvisualvm.JStackExample$$Lambda$2/930990596.run(Unknown Source)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:750)

"pool-1-thread-1" #11 prio=5 os_prio=31 tid=0x00000001598dc800 nid=0x5903 in Object.wait() [0x0000000172a4a000]
   java.lang.Thread.State: BLOCKED (on object monitor)
    at java.lang.Object.wait(Native Method)
    - waiting on <0x000000076adddbb8> (a com.qisi.simple.Jvisualvm.JStackExample)
    at java.lang.Object.wait(Object.java:502)
    at com.qisi.simple.Jvisualvm.JStackExample.test1(JStackExample.java:19)
    - locked <0x000000076adddbb8> (a com.qisi.simple.Jvisualvm.JStackExample)
    at com.qisi.simple.Jvisualvm.JStackExample$$Lambda$1/777874839.run(Unknown Source)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:750)
```

 先看21:52:21时的信息，

由于是thread1先执行producer，获取到锁，表现为-locked <0x000000076adddbb8>，当执行到sleep时，状态变为TIMED_WAITING

当执行到wait时，thread1会释放锁，所以在2024-07-28 21:52:26，新增了一条- waiting on <0x000000076adddbb8>，同时状态变为了BLOCKED，表示在等待锁。此时thread2获取到锁,thread2新增- locked <0x000000076adddbb8>，状态为TIMED_WAITING

当thread的sleep醒来执行了notify时，后面的堆栈信息虽然没有抓取到，但可以推测出来，thread1被唤醒，因此，thread1会再次获取到锁，从堆栈信息来看则是waiting on会变为lock，因为此时只有thread1需要锁，如果有多个线程都需要锁，有可能会变成waiting to lock

## 死循环cpu打满问题

### 测试代码：

```java
/**
 * @author : qisi
 * @date: 2024/7/28
 * @description: cpu打满测试用例
 */
public class CpuFullExample {
    public static void main(String[] args) {
        Runnable empty = new Runnable() {
            @Override
            public void run() {
                int count=0;
                while (count<1000){
                    count++;
                }
                System.out.println(count);
            }
        };
        Runnable full = new Runnable() {
            @Override
            public void run() {
                int count=0;
                while(true){
                    count++;
                }
            }
        };
        new Thread(empty).start();
        new Thread(full).start();
    }
}
```

根据top -pid [应用Id]找到cpu最高的tid，将10进制tid转换为16进制

根据jstack pid 生成tdump文件，根据转换出来的16进制，搜索对应的nid，此时的线程大概率是runnbale，不然怎么可能死循环，然后根据调用链路去找项目中对应的位置。

## 死锁问题

测试代码

```java
public class DeadLockExample {
    public Object resourceA = new Object();
    public Object resourceB = new Object();
    public static void main(String[] args) {
        DeadLockExample deadLockExample = new DeadLockExample();
        Runnable runnableA = new Runnable() {
            @Override
            public void run() {
                synchronized(deadLockExample.resourceA) {
                    System.out.printf(
                        "[INFO]: %s get resourceA" + System.lineSeparator(),
                        Thread.currentThread().getName()
                    );
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.printf(
                        "[INFO]: %s trying to get resourceB" + System.lineSeparator(),
                        Thread.currentThread().getName()
                    );
                    synchronized(deadLockExample.resourceB) {
                        System.out.printf(
                            "[INFO]: %s get resourceB" + System.lineSeparator(),
                            Thread.currentThread().getName()
                        );
                    }
                    System.out.printf(
                        "[INFO]: %s has done" + System.lineSeparator(),
                        Thread.currentThread().getName()
                    );
                }
            }
        };
        Runnable runnableB = new Runnable() {
            @Override
            public void run() {
                synchronized(deadLockExample.resourceB) {
                    System.out.printf(
                        "[INFO]: %s get resourceB" + System.lineSeparator(),
                        Thread.currentThread().getName()
                    );
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.printf(
                        "[INFO]: %s trying to get resourceA" + System.lineSeparator(),
                        Thread.currentThread().getName()
                    );
                    synchronized(deadLockExample.resourceA) {
                        System.out.printf(
                            "[INFO]: %s get resourceA" + System.lineSeparator(),
                            Thread.currentThread().getName()
                        );
                    }
                    System.out.printf(
                        "[INFO]: %s has done" + System.lineSeparator(),
                        Thread.currentThread().getName()
                    );
                }
            }
        };
        new Thread(runnableA).start();
        new Thread(runnableB).start();
    }
}
```

这段程序执行后，runnableA在持有resourceA后会再尝试获取resourceB锁，但此时resourceB锁已经被runnableB获取，释放的条件则是runnableB执行完，但是runnableB执行过程又在等待resourceA锁，这样就陷入了死锁。

使用jps获取pid，使用jstack pid生成文件。文件内容大致就是下面这样，其中大部分都是如同第一段一般，只有在出现死锁的时候，才会出现第二段（Found one Java-level deadlock），并清楚的告诉是哪些线程陷入死锁。

```java
"Thread-1" #14 prio=5 os_prio=31 tid=0x0000000125809000 nid=0x7a03 waiting for monitor entry [0x000000016efba000]
 java.lang.Thread.State: BLOCKED (on object monitor)
 at com.qisi.simple.Jvisualvm.DeadLockExample$2.run(DeadLockExample.java:59)
 - waiting to lock <0x000000076b5dee40> (a java.lang.Object)
 - locked <0x000000076b5dee50> (a java.lang.Object)
 at java.lang.Thread.run(Thread.java:750)

......

Found one Java-level deadlock:

"Thread-1":
 waiting to lock monitor 0x0000000124055770 (object 0x000000076b5dee40, a java.lang.Object),
 which is held by "Thread-0"
"Thread-0":
 waiting to lock monitor 0x0000000124058000 (object 0x000000076b5dee50, a java.lang.Object),
 which is held by "Thread-1"

......

Found 1 deadlock.
```

## 参考文档：

https://zihengcat.github.io/2019/08/09/java-tutorial-for-language-adavanced-deadlock-example-and-solution/

[图解 Java 线程生命周期-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1683376)

[JVM系列--jstack工具详解 | 奚新灿的博客-Chronos](https://xixincan.github.io/2020/05/20/Java/JVM/jstack%E5%B7%A5%E5%85%B7%E8%AF%A6%E8%A7%A3/)

[jstack 命令的使用和问题排查分析思路_jstack pid-CSDN博客](https://blog.csdn.net/CoreyXuu/article/details/110624151)

[Monitor对象全解析 - 张高峰的博客 | Peakiz Blog](https://peakzz.github.io/2021/11/12/Monitor%E5%AF%B9%E8%B1%A1%E5%85%A8%E8%A7%A3%E6%9E%90/)
