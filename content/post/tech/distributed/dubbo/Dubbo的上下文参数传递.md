+++
title = "Dubbo的上下文参数传递"
description = ""
date = 2024-07-03
image = ""
draft = true
slug = "Dubbo的上下文参数传递"
tags = ['中间件','Dubbo']
categories = ['tech']
+++

Dubbo是支持在每个rpc调用前后传递上下文的，如常见的traceId，或者dubbo预置的一些参数（group,version）等都是通过上下文传递的。具体使用可以参考[调用链路传递隐式参数](https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/advanced-features-and-usage/service/attachment/)，本文就来分析一下这个Context。

```java
public class RpcContext {

    private static final RpcContext AGENT = new RpcContext();

    /**
     * use internal thread local to improve performance
     */
    private static final InternalThreadLocal<RpcContextAttachment> SERVER_LOCAL = new InternalThreadLocal<RpcContextAttachment>() {
        @Override
        protected RpcContextAttachment initialValue() {
            return new RpcContextAttachment();
        }
    };

    private static final InternalThreadLocal<RpcContextAttachment> CLIENT_ATTACHMENT = new InternalThreadLocal<RpcContextAttachment>() {
        @Override
        protected RpcContextAttachment initialValue() {
            return new RpcContextAttachment();
        }
    };

    private static final InternalThreadLocal<RpcContextAttachment> SERVER_ATTACHMENT = new InternalThreadLocal<RpcContextAttachment>() {
        @Override
        protected RpcContextAttachment initialValue() {
            return new RpcContextAttachment();
        }
    };

    private static final InternalThreadLocal<RpcServiceContext> SERVICE_CONTEXT = new InternalThreadLocal<RpcServiceContext>() {
        @Override
        protected RpcServiceContext initialValue() {
            return new RpcServiceContext();
        }
    };

    /**
     * use by cancel call
     */
    private static final InternalThreadLocal<CancellationContext> CANCELLATION_CONTEXT = new InternalThreadLocal<CancellationContext>() {
        @Override
        protected CancellationContext initialValue() {
            return new CancellationContext();
        }
    };
```

观察RpcContext，发现分为了好几种Context，这里以*CLIENT_ATTACHMENT、SERVER_ATTACHMENT、SERVER_LOCAL这三种来串联这个链路。*

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NmE1MjYwNjUzY2ZjYjcxMTRjZGM0ODhlZTRlMzQ3ZjlfRVpzR3ZYOERIYVNVQzJJOGxwcVBUeEFNbXNtMlBEWVBfVG9rZW46QUZON2JiVGlJbzZaMXp4VlRGYWNJQjVBbkllXzE3MjMzNjk1MzY6MTcyMzM3MzEzNl9WNA)

## CLIENT_ATTACHMENT的设置

CLIENT_ATTACHMENT的作用是在consumer端调用provider的方法之前，进行设置上下文信息，对应方法是RpcContext.getClientAttachment().setAttachment("xxx","xxx");

日常开发的时候可以直接在调用provider之前设置，也可以在filter中统一设置，其中在filter中设置的时候还可以直接通过invocation.setObjectAttachment方法来设置。之所以可以这样设置是因为通过RpcContext.getClientAttachment().setAttachment设置的kv值，最终都会设置到invocation中去。

```Java
public class DubboConsumerService {
    @DubboReference
    private TestProvider testProvider;
    public void consumer(){
        RpcContext.getClientAttachment().setAttachment("key","value");
        testProvider.beat("consumer");
    }
}
public class DubboConsumerFilter implements Filter {
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        RpcContext.getClientAttachment().setAttachment("key","value");
        invocation.setObjectAttachmentIfAbsent("key","value");
        Result invoke = invoker.invoke(invocation);
        return invoke;
    }
}
```

而且，在consumer端，还需要过一个过滤器ConsumerContextFilter，order为min_value，所以会最先执行，这里主要逻辑是三部分，一部分是自定义了一些attachment；第二则是是否要保留之前客户端的attachment，比如A-B，B-C，在B-C之前是否需要保留A传给B的一些attachment，默认保留，如果不想保留就按SPI机制实现PenetrateAttachmentSelector；第三则是清除ServerContext，即返回给A的一些attachment。

之后，就会执行到Invoker。

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=Y2QzNzI4MTYxMjI2NWNhOWU4NTA3NTE0MzRmNjk3YWVfa1BsVU1Hcno1elpwQXpiZm5MWWRLZUZVMDNFc0Z3WElfVG9rZW46U3dNOGJPRkZLb1hzeWJ4NHFSYWNSQWlBbnVkXzE3MjMzNjk1Nzg6MTcyMzM3MzE3OF9WNA)

Dubbo调用方法的核心类DubboInvoker持有一个attachment的类，在执行AbstractInvoker的invoke之前会调用prepareInvocation之前将ClientAttachment的值复制到attachment对象中。

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDMxOTRjOTJhM2I3ZmI5NDU5ZDcxODJhMzU5YmVlMWJfUW1kSm5maUtaT29YUUdoakprM2NsQXFGU1V3WEFvdU9fVG9rZW46SDNRUGJjSzhZb2xvcU14UnhRZGN1VWZDbmRjXzE3MjMzNjk1Nzg6MTcyMzM3MzE3OF9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGJmOTNiMTg5NGM0N2E1ZmZiZWU0OTVhMWVjMDIwNjNfYnFET3pXUUI5cDR3T2dBY2lvekpGMWVucHBCUnhZajRfVG9rZW46UnRzSGJOZEZkbzRMUGJ4UEo1VWNIZkk3bkJkXzE3MjMzNjk1Nzg6MTcyMzM3MzE3OF9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjQ0ODNmMmU1ZWI5ZDlhY2UwMTczZDFiNDMxNjJlZDVfenBKT1dsRXNGcEp4WHl5Mm10VHhFS3VrOE9YcFpZSEtfVG9rZW46RFRGR2JlTkUwb0NkTml4ZUd0cGNiUnpUbnljXzE3MjMzNjk1Nzg6MTcyMzM3MzE3OF9WNA)

## *SERVER_ATTACHMENT的设置*

从[Consumer]({{<ref "Consumer源码分析">}})中我们知道，在consumer端一个rpc的调用本质上是将一个invoker序列化之后通过socket进行传递。因此，上下文会作为一个属性值传递到服务端，然后被服务端反序列化之后放入到*SERVER_ATTACHMENT中。*

在org.apache.dubbo.rpc.filter.ContextFilter的invoker方法中。

```Java
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        //从invocation获取到getObjectAttachments
        Map<String, Object> attachments = invocation.getObjectAttachments();
        //这里不太懂为什么要new新的对象
        if (attachments != null) {
            Map<String, Object> newAttach = new HashMap<>(attachments.size());
            for (Map.Entry<String, Object> entry : attachments.entrySet()) {
                String key = entry.getKey();
                if (!UNLOADING_KEYS.contains(key)) {
                    newAttach.put(key, entry.getValue());
                }
            }
            attachments = newAttach;
        }

        RpcContext.getServiceContext().setInvoker(invoker)
                .setInvocation(invocation);
        //获取到ServerAttachment的引用
        RpcContext context = RpcContext.getServerAttachment();
//                .setAttachments(attachments)  // merged from dubbox
        context.setLocalAddress(invoker.getUrl().getHost(), invoker.getUrl().getPort());

        String remoteApplication = invocation.getAttachment(REMOTE_APPLICATION_KEY);
        if (StringUtils.isNotEmpty(remoteApplication)) {
            RpcContext.getServiceContext().setRemoteApplicationName(remoteApplication);
        } else {
            RpcContext.getServiceContext().setRemoteApplicationName(context.getAttachment(REMOTE_APPLICATION_KEY));
        }

        long timeout = RpcUtils.getTimeout(invocation, -1);
        if (timeout != -1) {
            // pass to next hop
            RpcContext.getClientAttachment().setObjectAttachment(TIME_COUNTDOWN_KEY, TimeoutCountDown.newCountDown(timeout, TimeUnit.MILLISECONDS));
        }

        // merged from dubbox
        // we may already add some attachments into RpcContext before this filter (e.g. in rest protocol)
        //将来自client的attachment放入ServerAttachment中
        if (attachments != null) {
            if (context.getObjectAttachments().size() > 0) {
                context.getObjectAttachments().putAll(attachments);
            } else {
                context.setObjectAttachments(attachments);
            }
        }
```

## SERVER_CONTEXT的设置

provider端的SERVER_CONTEXT的设置和consumer端的client很像，都是在方法中或者filter中可以设置值，同时也可以通过invocation直接设置值。只不过不同于client的地方是，client是在consumerContextFilter中设置到invoke中的。而Server_context是在ContextFilter的onResponse中设置的，同时还清除了其他的上下文。这样子也表明服务端是不需要手动清除ServerAttachment的

```Java
@Override
public void onResponse(Result appResponse, Invoker<?> invoker, Invocation invocation) {
    // pass attachments to result
    appResponse.addObjectAttachments(RpcContext.getServerContext().getObjectAttachments());
    removeContext();
}
private void removeContext() {
    RpcContext.removeServerAttachment();
    RpcContext.removeClientAttachment();
    RpcContext.removeServiceContext();
    RpcContext.removeServerContext();
}
```

## SERVER_CONTEXT的获取

当consumer端执行到onResponse的时候，就会将appResponse中的objectAttachment复制到serverContext，同时会会清除掉clientAttachment，因此在response之后再次get时，会重新生成一个对象

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=OTJkOTMzY2JjYTlmZmNmMDQyNzIzNzllNGZjNDMyNjJfdW5nazRsaFF6U0dEWHNUT0VraEZySzREd0NUa1p2WkNfVG9rZW46V1p1bWJlcXRhb3NVaHd4R3d5WWNzaW42bm9BXzE3MjMzNjk1Nzg6MTcyMzM3MzE3OF9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MDA2NDY5OTlmY2U5MTMxZTg5ZmY3MjJhNzBiMmMwN2NfUGFSQjBSZ0xKeFY2azVsb0p2Q1IxVFFpQ3lDSkpuaURfVG9rZW46TGRtVWJxRzhCbzV5ZHZ4NUdYQmNWYWIybldlXzE3MjMzNjk1Nzg6MTcyMzM3MzE3OF9WNA)

但是要注意一点，由于onResponse是在回调中执行的，本身是异步的，所以在自定义的Filter中可能获取不到值，即当执行到自定义filter时，其response还没有执行到。或者说，只要能获取到clientAttachment，那就表明ServerContext还没有赋值，两者只能同时存在一个。
