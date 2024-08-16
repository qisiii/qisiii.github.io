+++
title = "Guava的RateLimiter"
description = ""
date = 2024-01-06
image = ""
draft = false
slug = "Guava_RateLimiter"
tags = ['限流']
categories = ['tech']
+++

单机的限流工具

# 使用

```Java
//普通的限流，不支持预热，使用的是SmoothBursty
RateLimiter rateLimiter=RateLimiter.create(10);
//获取令牌的时间
double acquire = rateLimiter.acquire();
//能否获取令牌
boolean b = rateLimiter.tryAcquire();
//支持预热的限流，使用的是Smoothwarmup
RateLimiter warmupLimiter=RateLimiter.create(10,10, TimeUnit.SECONDS);

//虽然创建了100个线程，但是每秒只有10个令牌，所以要花10秒
static void mock(){
    RateLimiter rateLimiter=RateLimiter.create(10);
    ExecutorService executorService = Executors.newFixedThreadPool(100);
    for (int i = 0; i < 100; i++) {
        executorService.submit(()->{
            System.out.println(Thread.currentThread().getName()+" "+rateLimiter.acquire()+"秒执行结束");
        });
    }
}
```

# 源码分析

![](http://picgo.qisiii.asia/post/1723663939248.jpg)

## RateLimiter

类图如图中所示，RateLimiter类只有两个属性，一个是计数器，一个是令牌.

### SleepingStopWatch

计数器的类型是SleepingStopwatch，是个抽象类，但其实现是通过匿名内部类实例化的。主要是持有了一个Stopwatch实例，同时重写了两个方法。

而Stopwatch是通过计算两个纳秒值的差值来进行使用的。

```Java
public static SleepingStopwatch createFromSystemTimer() {
  //匿名内部类
  return new SleepingStopwatch() {
    final Stopwatch stopwatch = Stopwatch.createStarted();

    @Override
    protected long readMicros() {
    //转换单位 微秒
      return stopwatch.elapsed(MICROSECONDS);
    }

    @Override
    protected void sleepMicrosUninterruptibly(long micros) {
      if (micros > 0) {
      //不可中断的睡眠
        Uninterruptibles.sleepUninterruptibly(micros, MICROSECONDS);
      }
    }
  };
}

public final class Stopwatch {
  //本质是System.nanoTime();
  private final Ticker ticker;
  private boolean isRunning;
  //经过的时间 elapsedNanos += tick - startTick;
  private long elapsedNanos;
  private long startTick;

  public long elapsed(TimeUnit desiredUnit) {
  //转换时间 纳秒转为指定单位（微秒）
  return desiredUnit.convert(elapsedNanos(), NANOSECONDS);
}
```

### mutexDoNotUseDirectly

令牌相对简单，使用了volatile和synchronized，并且使用了double-check，用于acquire方法和dosetrate等核心方法之前，因此，RateLimiter是线程安全的。

```Java
// Can't be initialized in the constructor because mocks don't call the constructor.
@CheckForNull private volatile Object mutexDoNotUseDirectly;

private Object mutex() {
  Object mutex = mutexDoNotUseDirectly;
  if (mutex == null) {
    synchronized (this) {
      mutex = mutexDoNotUseDirectly;
      if (mutex == null) {
        mutexDoNotUseDirectly = mutex = new Object();
      }
    }
  }
  return mutex;
}
```

## SmoothRateLimiter

SmoothRateLimiter是继承了RateLimiter，其本质是个基类，提供一些公共逻辑。

一个核心的逻辑是

当前线程来获取令牌时，只要存在令牌，如果令牌足够直接扣减令牌；

如果令牌不足够，那么先将令牌扣光，然后nextFreeTicketMicros会追加剩下需要新创建令牌所需的时间，但这个代价由下一个线程来承受，当前线程不会阻塞。

因此，在storedPermits不为空的情况下，acquire(1)和acquire(1000)对当前线程来说没有区别，但下一个线程要等待(1000-storedPermits)*stableIntervalMicros的时间

```Java
//当前存储的令牌数量double storedPermits;

//最大可存储的令牌数量
double maxPermits;

/**
 * 每间隔多少时间产生一个permit，比如1秒5个令牌，那就是200ms
 */
double stableIntervalMicros;

/**
 * The time when the next request (no matter its size) will be granted. After granting a request,
 * this is pushed further in the future. Large requests push this further than small requests.
 * 下一次可以获取permits的时间，很关键
 */
private long nextFreeTicketMicros = 0L;
```

## SmoothBursty

获取令牌的时候不用等待。

### 初始化部分

主要是要初始化成员变量

```Java
//创建的时候都是通过RateLimiter.create来创建的
//permitsPerSecond是指每秒创建多少令牌，用于设置速率
static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
  //SmoothBursty的突发指的意思是能满足瞬间的突发流量而无需等待
  // 他可以提前缓存一些令牌，当流量突发的时候可以瞬间处理
  //maxBurstSeconds固定为1，所以最多最多缓存1秒的令牌
  RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
  //设置速率
  rateLimiter.setRate(permitsPerSecond);
  return rateLimiter;
}

public final void setRate(double permitsPerSecond) {
  checkArgument(
      permitsPerSecond > 0.0 && !Double.isNaN(permitsPerSecond), "rate must be positive");
  synchronized (mutex()) {
    doSetRate(permitsPerSecond, stopwatch.readMicros());
  }
}

@Override
final void doSetRate(double permitsPerSecond, long nowMicros) {
  //同步更新库存和时间
  resync(nowMicros);
  double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
  //间隔在这里才赋值
  this.stableIntervalMicros = stableIntervalMicros;
  //设置速率，也可能是新速率
  doSetRate(permitsPerSecond, stableIntervalMicros);
}
//RateLimiter的令牌不是通过一个额外的线程来追加维护的，而是在关键的节点前计算出来的
//具体的实现就是resync方法，会在reserveEarliestAvailable和doSetRate两个方法前调用
void resync(long nowMicros) {
  // 如果nextFreeTicketMicros比当前时间还大，那表明有别的线程已经等到未来了,所以现在到未来的令牌都被占用了
  //如果nextFreeTicketMicros比当前时间小，那么表明这段时间应该产生了新的令牌
  if (nowMicros > nextFreeTicketMicros) {
    //coolDownIntervalMicros在SmoothBurste里面是固定间隔stableIntervalMicros
    //但是在初始化时，stableIntervalMicros还未赋值
    //浮点数0可以做除数，那这个Infinity（无限大）有啥意义呢？
    double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
    storedPermits = min(maxPermits, storedPermits + newPermits);
    nextFreeTicketMicros = nowMicros;
  }
}

//初始化时真正的复制逻辑在这里
@Override
void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
  double oldMaxPermits = this.maxPermits;
  //最大数量，maxPermits 为 1 秒产生的 permits
  maxPermits = maxBurstSeconds * permitsPerSecond;
  //oldMaxPermits什么情况才能是无穷大？
  if (oldMaxPermits == Double.POSITIVE_INFINITY) {
    // if we don't special-case this, we would get storedPermits == NaN, below
    storedPermits = maxPermits;
  } else {
  //初始化的时候是0
    storedPermits =
        (oldMaxPermits == 0.0)
            ? 0.0 // initial state
                //假如本来有10个，本来最大是100，现在每秒速率调整成50，那么就应该剩10/100*50
            : storedPermits * maxPermits / oldMaxPermits;
  }
}
```

### 获取令牌

有两种方式，

一种是acquire(n)，不传令牌数量默认为1，这种情况下，如果获取到令牌，会返回等待的时间；但是如果无法立即获取到令牌，那么会在睡眠一段时间后，再返回，等同于是阻塞住一段时间，类似于时间片

另一种tryAcquire，不传令牌数量默认为1，需要传一个等待时间，如果在这段等待时间内可以获取到令牌，那么返回true，否则返回false

```Java
public double acquire() {
  return acquire(1);
}
public boolean tryAcquire(Duration timeout) {
  return tryAcquire(1, toNanosSaturated(timeout), TimeUnit.NANOSECONDS);
}

public double acquire(int permits) {
  //计算时间
  long microsToWait = reserve(permits);
  //执行sleep
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  //返回sleep的时长（秒）
  return 1.0 * microsToWait / SECONDS.toMicros(1L);
}
//下面两个方法逻辑比较简单，核心的方法是reserveEarliestAvailable
final long reserve(int permits) {
  checkPermits(permits);
  synchronized (mutex()) {
    return reserveAndGetWaitLength(permits, stopwatch.readMicros());
  }
}
final long reserveAndGetWaitLength(int permits, long nowMicros) {
  long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
  return max(momentAvailable - nowMicros, 0);
}

final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
  //更新库存令牌和休眠时间，可以再回resync看看原逻辑
  resync(nowMicros);
  //老的等待时间，也是返回结果，但是会加上新的等待时间
  long returnValue = nextFreeTicketMicros;
  //返回可以返回的令牌
  double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
  //不够的令牌
  double freshPermits = requiredPermits - storedPermitsToSpend;
  //根据不够的令牌计算需要等待多久
  long waitMicros =
      //这个方法很重要，是和warmingup的重要区别
      storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)//SmoothBursty固定是0
          + (long) (freshPermits * stableIntervalMicros);
  //追加等待时间
  this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
  //减少库存
  this.storedPermits -= storedPermitsToSpend;
  return returnValue;
}

//try的部分
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {
  long timeoutMicros = max(unit.toMicros(timeout), 0);
  checkPermits(permits);
  long microsToWait;
  synchronized (mutex()) {
    long nowMicros = stopwatch.readMicros();
    //如果不能获取，立即返回false，否则和acquire一样的逻辑
    if (!canAcquire(nowMicros, timeoutMicros)) {
      return false;
    } else {
      microsToWait = reserveAndGetWaitLength(permits, nowMicros);
    }
  }
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  return true;
}

private boolean canAcquire(long nowMicros, long timeoutMicros) {
  //当前时间+可以忍受的等待时间是否大于目前下一次可以获取令牌的时间，queryEarliestAvailable是nextFreeTicketMicros
  return queryEarliestAvailable(nowMicros) - timeoutMicros <= nowMicros;
}
```

## SmoothWarmingUp

这个类主要是为了考虑资源需要预热的情况，不同于SmoothBursty，当并发量来的时候，会立刻返回存货，没有存货也是以固定的速率等待新的令牌产生；而这里，如果想要获取令牌，任何时候都是需要等待的，等待时间还是不固定的，当库存中令牌越多，等待时间要越久，这个时间就是为了预热其他资源的。

![](http://picgo.qisiii.asia/post/1723663976846.jpg)

观察这张图，x轴表示令牌，y轴表示获取令牌等待的时间。初始是stableIntervalMicros，和smoothBursty是一样的。但随着令牌增多，获取时等待的时间就越来越长。当桶中的令牌越多时，我们一般认为系统是越冷的，在这种情况下，获取令牌会越来越慢。

```Java
public static RateLimiter create(double permitsPerSecond, Duration warmupPeriod) {
  return create(permitsPerSecond, toNanosSaturated(warmupPeriod), TimeUnit.NANOSECONDS);
}
```

在创建的时候，除了permitsPerSecond，还有一个warmupPeriod，

### 图的理解

coldInterval=codeFactor*stableInterval,codeFactor恒定为3

变冷速率等于 maxPermits/warmupPeriod

从maxPermits到thresholdPermits是warmupPeriod

从0移动到thresholdPermits是warmupPeriod/2

The reason that this is warmupPeriod/2 is to maintain the behavior of the original implementation where coldFactor was hard coded as 3

为什么面积就是2倍呢？老版本的源码thresholdPermits就是halfPermits，如果halfPermits就是maxPermits的一半，那确实是两倍，那问题就是thresholdPermits为啥要是一半呢？

### 源码分析

计算属性

```Java
void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
  double oldMaxPermits = maxPermits;
  double coldIntervalMicros = stableIntervalMicros * coldFactor;
  //warmupPeriodMicros为梯形面积，warmupPeriodMicros=2*s*t，所以
  thresholdPermits = 0.5 * warmupPeriodMicros / stableIntervalMicros;
  //这里是因为梯形面积是warmupPeriod = 0.5 * (stableInterval + coldInterval) * (maxPermits - thresholdPermits)
  maxPermits =
      thresholdPermits + 2.0 * warmupPeriodMicros / (stableIntervalMicros + coldIntervalMicros);
  //对边比临边
  slope = (coldIntervalMicros - stableIntervalMicros) / (maxPermits - thresholdPermits);
  if (oldMaxPermits == Double.POSITIVE_INFINITY) {
    // if we don't special-case this, we would get storedPermits == NaN, below
    storedPermits = 0.0;
  } else {
    storedPermits =
        (oldMaxPermits == 0.0)
            ? maxPermits // initial state is cold
            : storedPermits * maxPermits / oldMaxPermits;
  }
}
```

冷却时间（每隔多久产生一个令牌）

```Java
double coolDownIntervalMicros() {
  return warmupPeriodMicros / maxPermits;
}
```

等待时间

对应的其实就是图中令牌k-x的面积

```Java
long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {
  double availablePermitsAboveThreshold = storedPermits - thresholdPermits;
  long micros = 0;
  // measuring the integral on the right part of the function (the climbing line)
  //如果梯形里面有令牌
  if (availablePermitsAboveThreshold > 0.0) {
    //梯形里面能取的令牌，也就是高
    double permitsAboveThresholdToTake = min(availablePermitsAboveThreshold, permitsToTake);
    // TODO(cpovirk): Figure out a good name for this variable.
    // 理论上应该是上底+下底
    double length =
        permitsToTime(availablePermitsAboveThreshold)
            + permitsToTime(availablePermitsAboveThreshold - permitsAboveThresholdToTake);
    micros = (long) (permitsAboveThresholdToTake * length / 2.0);
    permitsToTake -= permitsAboveThresholdToTake;
  }
  //permitsToTake是剩余的要取的令牌，所以这是长方形部分
  // measuring the integral on the left part of the function (the horizontal line)
  micros += (long) (stableIntervalMicros * permitsToTake);
  return micros;
}

private double permitsToTime(double permits) {
  //s高+临边*斜率（对边）
  return stableIntervalMicros + permits * slope;
}
```

# 参考文档:

[RateLimiter 源码分析(Guava 和 Sentinel 实现)_Javadoop](https://www.javadoop.com/post/rate-limiter)
