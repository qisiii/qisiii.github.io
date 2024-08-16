+++
title = "Java的SPI机制"
description = ""
date = 2023-03-17
image = ""
draft = false
slug = "java_spi"
tags = ['Java','SPI']
categories = ['tech']
+++

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NjE1OWUxMGFjMWZiMGI3ZDkxYjk2M2IyYWRmYjFmZDZfZGFNQ3h4WU1oVFcyZUlYUVY3ZXhWUms3dVRjRzJScUhfVG9rZW46Qm1qRGJXU0RZb2l3VnV4SUJxTGNOTTJQbkZnXzE3MjMzMzkzNTk6MTcyMzM0Mjk1OV9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NjI3ODllMGU2NWFjYzNkYjg5NDE3MThhMTAzZjBhZTVfRFNoYkQ4ODlHdVJnUDBUUnZMNlB1ZEo0cUdGYVc5aDlfVG9rZW46VzlpV2JwOHVCb0pXdnN4M3YzeWNDUUVnblRlXzE3MjMzMzkzNTk6MTcyMzM0Mjk1OV9WNA)

本质是将接口和实现进行解耦，使得外部程序可以提供不同的实现；

实现机制为ServiceLoader通过迭代器进行

```Java
public Iterator<S> iterator() {
    return new Iterator<S>() {
        //之所以有knownProviders是为了处理多次迭代的情况，首次迭代的时候就将其缓存到了providers，
        //如果要重新加载所有的实现类，需要调用reload
        Iterator<Map.Entry<String,S>> knownProviders
            = providers.entrySet().iterator();
        //本质是lookupIterator的方法
        public boolean hasNext() {
            if (knownProviders.hasNext())
                return true;
            return lookupIterator.hasNext();
        }

        public S next() {
            if (knownProviders.hasNext())
                return knownProviders.next().getValue();
            return lookupIterator.next();
        }

        public void remove() {
            throw new UnsupportedOperationException();
        }

    };
}
```

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjEwOGNkYjkwOTU4NzgyZGFhMDIyNzliNzI1YTYzYmZfSTJwZ0ZuWmdzVEJCM0ZieUltV1c1TWFBa0gwek9NYmlfVG9rZW46S1NySWJnRjM3b2xkdm54UDFoM2M2aUVEblZlXzE3MjMzMzkzNTk6MTcyMzM0Mjk1OV9WNA)

从图中可以看出核心方法是hasNextService和nextService，acc是[Java安全机制之一——SecurityManager和AccessController]({{<ref "java安全机制">}})，可以忽略不看。

```Java
private boolean hasNextService() {
    if (nextName != null) {
        return true;
    }
    if (configs == null) {
        try {
            //PREFIX为META-INF/services/，service.getName是全限定名
            String fullName = PREFIX + service.getName();
            if (loader == null)
                configs = ClassLoader.getSystemResources(fullName);
            else
                configs = loader.getResources(fullName);
        } catch (IOException x) {
            fail(service, "Error locating configuration files", x);
        }
    }
    while ((pending == null) || !pending.hasNext()) {
        if (!configs.hasMoreElements()) {
            return false;
        }
        //将文件按行读取，每一行是list的一个元素
        pending = parse(service, configs.nextElement());
    }
    nextName = pending.next();
    return true;
}

private S nextService() {
    if (!hasNextService())
        throw new NoSuchElementException();
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
    //搞到类
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,
             "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service,
             "Provider " + cn  + " not a subtype");
    }
    try {
    //实例化，并强转为接口类型
        S p = service.cast(c.newInstance());
        //缓存方便以后使用
        providers.put(cn, p);
        return p;
    } catch (Throwable x) {
        fail(service,
             "Provider " + cn + " could not be instantiated",
             x);
    }
    throw new Error();          // This cannot happen
}
```

缺点是：不能按需加载，需要将所有的实现都遍历完，实例化完。

应用场景：比如spring参考SPI机制搞的spring.factories，dubbo的@SPI

## 参考文档：

[深入理解 Java 中 SPI 机制](https://zhuanlan.zhihu.com/p/84337883)

[Java SPI概念、实现原理、优缺点、应用场景、使用步骤、实战SPI案例-CSDN博客](https://blog.csdn.net/qq_52423918/article/details/130968307)
