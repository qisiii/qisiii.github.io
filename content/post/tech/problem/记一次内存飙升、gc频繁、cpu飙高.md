+++
title = "记一次内存飙升、gc频繁、cpu飙高"
description = ""
date = 2023-08-19
image = ""
draft = false
slug = "task_fullgc"
tags = ['线上异常']
categories = ['tech']
+++

## 紧急排查：

一个普通工作日的晚上8点，突然收到大量接口超时的报警，最高甚至有超100s，于是开始紧急排查。

首先是看监控面板里的内存监控和cpu监控，说到这里不得不吐槽一句公司的内存监控没什么软用（当时并不会熟练使用arthas)，监控的是pod节点内存情况，也就是说一直展示的都是历史最高的，因为jvm申请内存后是不会返还给实例的，所以并不能看出来什么。倒是cpu确实彪的很高。

直觉判断，肯定是内存上去了，频繁gc，导致的cpu飙高，那么是哪里的问题呢？

下载内存dump文件，5个G，公司那小水管，每秒2兆，下载的贼慢，很绝望。

而且观察发现，只有蓝组的一台机器是这样子的，其他的机器都没有问题，所以为了不影响用户使用，先把蓝组流量全部关了，慢慢查。

通过命令`jmap -histo 73066 | head -n 30`查看，以前分析dump的时候，排名前几的都是char、Int等基本类型的对象居多，根本看不出什么问题，而这个虽然也是char 和int排名靠前，但是排在第5名的是一个普通的po类，去查这个类调用的地方，最终定位到是一个定时任务出现了问题，这也解释了为什么只影响了蓝组的一台实例，其他的没有受影响，加上这个任务并不是太重要，就手动先停了，今天先看看什么问题，第二天进行上线修补。

定时任务的核心代码是为了维护一个列表，下面是个demo

```java
//这段代码逻辑大概是从某个接口查询一个列表，然后和数据库的进行比较
//最新列表比数据库多的要新增落库，少的要删掉，已经在的要更新
List<String> list = client.getEnableList();
List<Memory> memories = memoryMapper.selectList(new QueryWrapper<>());
ArrayList<Memory> addList = new ArrayList<>();
ArrayList<Memory> updateList = new ArrayList<>();
for (Memory memory : memories) {
    if (!list.contains(memory.getName())){
        addList.add(memory);
    }else{
        updateList.add(memory);
    }
}
for (Memory memory : updateList) {
    memoryMapper.insert(memory);
}
```

观察上面这段逻辑，发现!list.contains的时候应该删除才对，而不是添加，应该是接口返回结果有，数据库没有才要添加，那仅仅如此为什么会导致异常呢？当时查了一下这个表的数据，有780万条数据，按照业务idgroupby了一下发现某个id就有780万重复数据，其他的数据都是1条。破案了，就是这条数据重复插入导致的，插入了780万次。

但是仔细看现在的代码，是表里有，但是接口没有，才会进行插入。

那表里这条数据为什么有呢？第一次是什么时候插入的呢？看了一下插入时间是几天前，也就是说几天前接口还返回了这条数据，但是今天突然不返回了，导致了这个问题

而没有立即暴露出来的原因是，假设第一次查出来是1条数据，然后新增了一条；下一次查出来就是2条，然后会新增两条；数量低的时候内存不会有压力，加上定时任务是半小时执行一次，所以是每半小时翻一番，在翻到第23番的时候终于出现问题了；2->4->8->16->.....->2^23(8388608)，

所以总结一下出问题的原因：

诱因是今天返回的数据比之前少了一条，少的这条因为代码逻辑问题会重复插入（没有加唯一索引）

# 复现：

还是这段代码，为了加快速度和方便测试，我们做了一下调整

```Dockerfile
//mock接口返回数据就只有一个“first”
List<String> first = Arrays.asList("first");
List<Memory> memories = memoryMapper.selectList(new QueryWrapper<>());
System.out.println("数据库List占用内存"+ObjectSizeCalculator.getObjectSize(memories)/1024/1024+"M");
ArrayList<Memory> addList = new ArrayList<>();
ArrayList<Memory> updateList = new ArrayList<>();
for (Memory memory : memories) {
    if (!first.contains(memory.getName())){
        addList.add(memory);
    }else{
        updateList.add(memory);
    }
}
//这里不再单个插入，而是批量插入，5万一次
Lists.partition(addList,50000).forEach(t->{
    System.out.println("大List占用内存"+ObjectSizeCalculator.getObjectSize(addList)/1024/1024+"M");
    memoryMapper.insertBatchSomeColumn(t);
});
```

数据库初始就两条数据，根据上面的代码，second这条会以2的幂级插入

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MjY1MjJiNWM0ZTJjODE5Nzk4ODUyNzA2OTNjZjIzNjhfczJmU2JPcVpyMzZLZkRkUzlmZVJGN3QwWElRNnZkWVBfVG9rZW46TmlqNmIzYWdob1A3S1V4RWVqT2NRMDRlbm9oXzE3MjMzNjE2MjQ6MTcyMzM2NTIyNF9WNA)

而且为了尽快达到上限，jvm参数得调整一下，最大堆200m，年轻代40m

```Dockerfile
-Xms200m
-Xmx200m
-Xmn40m
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
```

好了，接下来使用jconsole监控这个进程，然后一次次的触发这段程序，进行观察
这是启动时，没有触发的状态

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=YjFiOThiMzgyMWRjN2I0Njk3MDlhZWE5NWZkZTIyYjlfZXI1ejhsa21BTEEyMTl6VmtYaUppakNuYkdEUFdoZHBfVG9rZW46Q25TVWJHb0JJbzlFR2x4QXJhOWNRWEVXbnVjXzE3MjMzNjE2MjQ6MTcyMzM2NTIyNF9WNA)

在2^15次时，对象终于达到1M，然后是3M,6M,13M,27M,54M

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=OTFlMmMyODhiYzYyMWZmNzhhZTRiNjY5ZjNlZDQ5ZDNfb1RRVWxjSmlxSG5YSThsV0RJODRCN2pBRXZxSHNjMFFfVG9rZW46TmlxYWJlOUZib3phM294YXp1bWNUcm4zbnpnXzE3MjMzNjE2MjQ6MTcyMzM2NTIyNF9WNA)

最终报了OOM，而这个时候数据库是1048577条数据，20次的时候是54M，而且由于两个list（memories和addList)，所以代码实际使用100M，在下一次执行就得申请200M了，总共就200M，于是OOM，我们简单估算一下线上当时会申请多少M？21->100M,22->200M,23->400M，所以在第23次的时候就得申请两个400M，也就是800M了。

```log
2023-08-19 20:22:23.521 ERROR 57861 --- [nio-8080-exec-5] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Handler dispatch failed; nested exception is java.lang.OutOfMemoryError: GC overhead limit exceeded] with root cause

java.lang.OutOfMemoryError: GC overhead limit exceeded
```

## 参考文档：

[面试官：一个线程OOM，进程里其他线程还能运行么？](https://zhuanlan.zhihu.com/p/61650387)

[面试官问：平时碰到系统CPU飙高和频繁GC，你会怎么排查？](https://www.jianshu.com/p/cf3d157e245f)

[java面试oom问题及答案_java面试中必问的oom问题_卜奕的博客-CSDN博客](https://blog.csdn.net/weixin_34409887/article/details/114746993)
