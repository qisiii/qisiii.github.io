+++
title = "Consumer源码分析"
description = ""
date = 2024-04-29
image = ""
draft = true
slug = "Consumer源码分析"
tags = ['Dubbo','中间件']
categories = ['tech']
+++

# 启动流程

## ReferenceBean

### 注册beanDefinition

同provider类似，reference的相关机制中比较核心的是一个叫做ReferenceBean的东西。

首先，在ReferenceAnnotationBeanPostProcessor中，会扫描所有的标有DubboReference.class, Reference.class, com.alibaba.dubbo.config.annotation.Reference.class的依赖注入的bean，如果有的话，就进入到下面的注册bean里

org.apache.dubbo.config.spring.beans.factory.annotation.ReferenceAnnotationBeanPostProcessor#registerReferenceBean

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=OGVhNTE1Yjg2NDQ2ODYzNzA3NjUwYWRhNzIwNmJhZGJfcUh3ckRsUVJhcnpNV0xuR3FDbU5sdDZabXVhWllDZEJfVG9rZW46V3NHWWJiN0Jhb3V2bGt4NFhodWNmQmwwbkNnXzE3MjMzNzAwODU6MTcyMzM3MzY4NV9WNA)

有了beanDefinition之后，在实例化bean的时候就会进行实例为一个类型为ReferenceBean的bean，在实例化过程中，有两个地方需要关注。

一是ReferenceBean重写了afterPropertiesSet，所以在实例化过程中会调用afterPropertiesSet方法，主要逻辑是将这个bean添加到了ReferenceBeanManager中

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjA3MTk4YTI1NjYwNzczYTdiNmQxMzFkZTA3MWQ1OWFfenBieFB0QnkwVlpkOEZyZHNNYVA0QnV2Zmw2b2UxRmpfVG9rZW46WG91WWI5b0xjb2toc3F4OGtRVGNXbm5WbmZlXzE3MjMzNzAwODU6MTcyMzM3MzY4NV9WNA)

二是ReferenceBean是一个FactoryBean，所以在createBean之后，最终会执行到ReferenceBean的getObject方法

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=N2RmNjNjYmQ5YWZlY2Q4Yzg1MDQ3ZTFiN2M1NWE5ZTRfQUtzaG1SdXg1bnJIYmlBem9JUWFvY3ZEQ1BQdHNMTUdfVG9rZW46VVh2S2JjeFdUb21oT3l4U2sxQWNpTjFzbjBjXzE3MjMzNzAwODU6MTcyMzM3MzY4NV9WNA)

这里的逻辑是将bean进行了代理，其targetSource是DubboReferenceLazyInitTargetSource，等同于是延迟了加载，在真正调用ReferenceBean方法的时候，才会执行到DubboReferenceLazyInitTargetSource的getCallProxy，最终执行到referenceConfig.get()，因此，referenceConfig.get()得到的对象才是真正的dubbo服务的bean。

```Java
private void createLazyProxy() {

    //set proxy interfaces
    //see also: org.apache.dubbo.rpc.proxy.AbstractProxyFactory.getProxy(org.apache.dubbo.rpc.Invoker<T>, boolean)
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.setTargetSource(new DubboReferenceLazyInitTargetSource());
    proxyFactory.addInterface(interfaceClass);
    Class<?>[] internalInterfaces = AbstractProxyFactory.getInternalInterfaces();
    for (Class<?> anInterface : internalInterfaces) {
        proxyFactory.addInterface(anInterface);
    }
    if (!StringUtils.isEquals(interfaceClass.getName(), interfaceName)) {
        //add service interface
        try {
            Class<?> serviceInterface = ClassUtils.forName(interfaceName, beanClassLoader);
            proxyFactory.addInterface(serviceInterface);
        } catch (ClassNotFoundException e) {
            // generic call maybe without service interface class locally
        }
    }

    this.lazyProxy = proxyFactory.getProxy(this.beanClassLoader);
}

private Object getCallProxy() throws Exception {
    if (referenceConfig == null) {
        throw new IllegalStateException("ReferenceBean is not ready yet, please make sure to call reference interface method after dubbo is started.");
    }
    //get reference proxy
    return referenceConfig.get();
}

private class DubboReferenceLazyInitTargetSource extends AbstractLazyCreationTargetSource {

    @Override
    protected Object createObject() throws Exception {
        return getCallProxy();
    }

    @Override
    public synchronized Class<?> getTargetClass() {
        return getInterfaceClass();
    }
}
```

## referenceConfig.get()

上面说到在首次调用dubbo方法的时候会调用referenceConfig.get()方法，其实在正常启动的时候也是会调用的

启动同provider一样，也是在DefaultModuleDeployer

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NmM5NmM3MmNiNjBhMTg5ZjhkNzllZTE0NjQ3NzA4NzNfVjZqek0wWEJ4MVlNaTJVbGFxS0hZZnB3ZDNsVjhiRkVfVG9rZW46UlB0SmJ2eDJwb3Y1SXZ4MFQ5RWNwcVVUbndiXzE3MjMzNzAwODU6MTcyMzM3MzY4NV9WNA)

由于在afterPropertiesSet中已经添加了ReferenceBean，这里可以发现在forEach里面关键代码是referenceCache.get，再往里跟会发现核心还是ReferenceConfig.get()

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MjE5OGJiMzNmYTJhY2U1OWEyNjNiNDg5NzBhODk1N2RfOWZRZlFwcUlpU0FZdno5YXg2cUV5RktaQXU3MGVrNVZfVG9rZW46SUgyZWJrSER1b1dFdlB4eHNya2N4dWJBbmtmXzE3MjMzNzAwODU6MTcyMzM3MzY4NV9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MWNlNzRmZGZjNjVmNGNmYTkyODI4ZGQ5MzE2ZTk4MTdfbEtIenpwNGpEaGVHTUMyUEp2eXFzbWdSOFRicGx0V1hfVG9rZW46STFYN2IwZ2trb2o4RmF4RXRLZmNOZURwblliXzE3MjMzNzAwODU6MTcyMzM3MzY4NV9WNA)

一直往下跟

org.apache.dubbo.config.ReferenceConfig#init->

org.apache.dubbo.config.ReferenceConfig#createProxy->

org.apache.dubbo.config.ReferenceConfig#createInvokerForRemote->

org.apache.dubbo.rpc.Protocol#refer

由于refer方法存在@Adaptive，所以需要看一下适配类的源码

通过arthas反编译源码，可以看出url中的protocol>默认（dubbo）,url中为registry://qisiii.asia:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-consumer-boot&dubbo=2.0.2&pid=73268&qos.enable=false&registry=zookeeper&release=3.0.7&timestamp=1718124890217，所以会调用到RegistryProtocol的refer方法。

核心方法其实是doRefer，简单说就是将consumer的一个Reference的Url封装为了一个invoker。

```Java
    public Invoker refer(Class clazz, URL uRL) throws RpcException {
        String string;
        if (uRL == null) {
            throw new IllegalArgumentException("url == null");
        }
        URL uRL2 = uRL;
        String string2 = string = uRL2.getProtocol() == null ? "dubbo" : uRL2.getProtocol();
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (").append(uRL2.toString()).append(") use keys([protocol])").toString());
        }
        ScopeModel scopeModel = ScopeModelUtil.getOrDefault(uRL2.getScopeModel(), Protocol.class);
        Protocol protocol = scopeModel.getExtensionLoader(Protocol.class).getExtension(string);
        return protocol.refer(clazz, uRL);
    }
//org.apache.dubbo.registry.integration.RegistryProtocol
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    url = getRegistryUrl(url);
    Registry registry = getRegistry(url);
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }

    // group="a,b" or group="*"
    Map<String, String> qs = (Map<String, String>) url.getAttribute(REFER_KEY);
    String group = qs.get(GROUP_KEY);
    if (StringUtils.isNotEmpty(group)) {
        if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
            return doRefer(Cluster.getCluster(url.getScopeModel(), MergeableCluster.NAME), registry, type, url, qs);
        }
    }

    Cluster cluster = Cluster.getCluster(url.getScopeModel(), qs.get(CLUSTER_KEY));
    return doRefer(cluster, registry, type, url, qs);
}

protected <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url, Map<String, String> parameters) {
    Map<String, Object> consumerAttribute = new HashMap<>(url.getAttributes());
    consumerAttribute.remove(REFER_KEY);
    String p = isEmpty(parameters.get(PROTOCOL_KEY)) ? CONSUMER : parameters.get(PROTOCOL_KEY);
    URL consumerUrl = new ServiceConfigURL (
        p,
        null,
        null,
        parameters.get(REGISTER_IP_KEY),
        0, getPath(parameters, type),
        parameters,
        consumerAttribute
    );
    url = url.putAttribute(CONSUMER_URL_KEY, consumerUrl);
    ClusterInvoker<T> migrationInvoker = getMigrationInvoker(this, cluster, registry, type, url, consumerUrl);
    return interceptInvoker(migrationInvoker, url, consumerUrl);
}
```

这个migrationInvoker，到目前为止其实依然无法被真正调用，因为依然没有和provider建立连接，甚至都没有获取到provider的相关url。

## 订阅provider&建立连接

interceptInvoker方法代码的核心逻辑其实是通过listener对refer和export方法进行定制处理，而目前只有一个类MigrationRuleListener，关于MigrationRuleListener可以看[应用级地址发现迁移](https://l8ut65fgfc.feishu.cn/wiki/BpfJwgsq4iFCEyklGsscPIIJnVd)这个文档，这个的主要逻辑就是兼容dubbo2和dubbo3的接口级地址发现和应用级地址发现，在该文档中我们不关心这部分业务，直接从接口级服务发现（refreshInterfaceInvoker）开始继续分析源码

```Java
protected <T> Invoker<T> interceptInvoker(ClusterInvoker<T> invoker, URL url, URL consumerUrl) {
    List<RegistryProtocolListener> listeners = findRegistryProtocolListeners(url);
    if (CollectionUtils.isEmpty(listeners)) {
        return invoker;
    }

    for (RegistryProtocolListener listener : listeners) {
        listener.onRefer(this, invoker, consumerUrl, url);
    }
    return invoker;
}
```

### refreshInterfaceInvoker

在refreshInterfaceInvoker中定义了一个DynamicDirectory对象，这个对象主要干了三个事，关键代码就是下面三行

```Java
//建立路由链
directory.buildRouterChain(urlToRegistry);
//订阅provider并建立连接
directory.subscribe(toSubscribeUrl(urlToRegistry));
//使用容错机制对invoker进行装饰
(ClusterInvoker<T>) cluster.join(directory, true);
```

### subscribe

```Java
for (String path : toCategoriesPath(url)) {
    ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.computeIfAbsent(url, k -> new ConcurrentHashMap<>());
    ChildListener zkListener = listeners.computeIfAbsent(listener, k -> new RegistryChildListenerImpl(url, path, k, latch));
    if (zkListener instanceof RegistryChildListenerImpl) {
        ((RegistryChildListenerImpl) zkListener).setLatch(latch);
    }
    zkClient.create(path, false);
    List<String> children = zkClient.addChildListener(path, zkListener);
    if (children != null) {
        urls.addAll(toUrlsWithEmpty(url, path, children));
    }
}
notify(url, listener, urls);
```

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDUyZjM3MzQwZThjYjA2NTdkZWYzNmU5YjBhNGJhNTNfTGNHam0zN3dxN254ek92MGVlbFp3c3NKTXY0d0k0QVlfVG9rZW46UmU3emJNQTlNb1dpM0V4d05KWmNwaXRUbm5nXzE3MjMzNzAwODU6MTcyMzM3MzY4NV9WNA)

观察上面的截图和代码可以知道，通过zkClient，对当前接口在zk的provider节点进行了监听，并添加了listener，然后调用notify进行首次获取值。

### notify

在接收到订阅的provider的url后，会将其转换为invoker，核心是再次调用refer方法，比如url是dubbo://192.168.31.16:20880/com.qisi.dubbo.api.TestProvider?anyhost=true&application=dubbo-provider-boot&background=false&category=providers,configurators,routers&check=false&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.qisi.dubbo.api.TestProvider&methods=beat,register&pid=78286&qos.enable=false&reference.filter=consumer&release=3.0.7&service-name-mapping=true&service.filter=provider&side=provider&sticky=false，这一次的refer就会走dubbo协议，即DubboProtocol的refer方法，最终是构造成了dubboinvoker。

```Java
private Map<URL, Invoker<T>> toInvokers(Map<URL, Invoker<T>> oldUrlInvokerMap, List<URL> urls) {
    Map<URL, Invoker<T>> newUrlInvokerMap = new ConcurrentHashMap<>(urls == null ? 1 : (int) (urls.size() / 0.75f + 1));
    if (urls == null || urls.isEmpty()) {
        return newUrlInvokerMap;
    }
    String queryProtocols = this.queryMap.get(PROTOCOL_KEY);
    for (URL providerUrl : urls) {

        // Cache key is url that does not merge with consumer side parameters, regardless of how the consumer combines parameters, if the server url changes, then refer again
        Invoker<T> invoker = oldUrlInvokerMap == null ? null : oldUrlInvokerMap.remove(url);
        if (invoker == null) { // Not in the cache, refer again
            try {
            ...
                    invoker = protocol.refer(serviceType, url);
                }
            } catch (Throwable t) {
                logger.error("Failed to refer invoker for interface:" + serviceType + ",url:(" + url + ")" + t.getMessage(), t);
            }
            if (invoker != null) { // Put new invoker in cache
                newUrlInvokerMap.put(url, invoker);
            }
        } else {
            newUrlInvokerMap.put(url, invoker);
        }
    }
    return newUrlInvokerMap;
}

public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    checkDestroyed();
    return protocolBindingRefer(type, url);
}

public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
    checkDestroyed();
    optimizeSerialization(url);

    // create rpc invoker.
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);

    return invoker;
}
```

### getClients

在构造DubboInvoker的构造参数里，有一个参数是clients，这个就是当前consumer和provider保持的长连接。最终会生成一个netty的client

```Java
return getExchanger(url).connect(url, handler);


public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
    return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
}
//org.apache.dubbo.remoting.Transporters#connect(org.apache.dubbo.common.URL, org.apache.dubbo.remoting.ChannelHandler...)
public static Client connect(URL url, ChannelHandler... handlers) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    ChannelHandler handler;
    if (handlers == null || handlers.length == 0) {
        handler = new ChannelHandlerAdapter();
    } else if (handlers.length == 1) {
        handler = handlers[0];
    } else {
        handler = new ChannelHandlerDispatcher(handlers);
    }
    return getTransporter(url).connect(url, handler);
}

public class NettyTransporter implements Transporter {

    public static final String NAME = "netty3";

    @Override
    public RemotingServer bind(URL url, ChannelHandler handler) throws RemotingException {
        return new NettyServer(url, handler);
    }

    @Override
    public Client connect(URL url, ChannelHandler handler) throws RemotingException {
        return new NettyClient(url, handler);
    }

}
```

# consumer的调用

consumer的调用有两个关键的invoker，一个是FailoverClusterInvoker，另一个是dubboInvoker。前者负责集群容错和负载均衡，后者则是进行真正的netty请求

```Java
//org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    List<Invoker<T>> copyInvokers = invokers;
    checkInvokers(copyInvokers, invocation);
    String methodName = RpcUtils.getMethodName(invocation);
    int len = calculateInvokeTimes(methodName);
    // retry loop.
    RpcException le = null; // last exception.
    List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size()); // invoked invokers.
    Set<String> providers = new HashSet<String>(len);
    for (int i = 0; i < len; i++) {
        //Reselect before retry to avoid a change of candidate `invokers`.
        //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
        if (i > 0) {
            checkWhetherDestroyed();
            copyInvokers = list(invocation);
            // check again
            checkInvokers(copyInvokers, invocation);
        }
        Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
        invoked.add(invoker);
        RpcContext.getServiceContext().setInvokers((List) invoked);
        boolean success = false;
        try {
            Result result = invokeWithContext(invoker, invocation);
            success = true;
            return result;
     }
 //org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker#doInvoke
 protected Result doInvoke(final Invocation invocation) throws Throwable {
    RpcInvocation inv = (RpcInvocation) invocation;
    final String methodName = RpcUtils.getMethodName(invocation);
    inv.setAttachment(PATH_KEY, getUrl().getPath());
    inv.setAttachment(VERSION_KEY, version);

    ExchangeClient currentClient;
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        currentClient = clients[index.getAndIncrement() % clients.length];
    }
    try {
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        int timeout = calculateTimeout(invocation, methodName);
        invocation.setAttachment(TIMEOUT_KEY, timeout);
        if (isOneway) {
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            currentClient.send(inv, isSent);
            return AsyncRpcResult.newDefaultAsyncResult(invocation);
        } else {
            ExecutorService executor = getCallbackExecutor(getUrl(), inv);
            CompletableFuture<AppResponse> appResponseFuture =
                    currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);
            // save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter
            FutureContext.getContext().setCompatibleFuture(appResponseFuture);
            AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
            result.setExecutor(executor);
            return result;
        }
    } catch (TimeoutException e) {
        throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    } catch (RemotingException e) {
        throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

这里的currentClient.request(inv,...)，inv会被序列化写入到channel中去
