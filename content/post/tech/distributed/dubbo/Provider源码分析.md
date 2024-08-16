+++
title = "Provider源码分析"
description = ""
date = 2024-03-17
image = ""
draft = true
slug = "Provider源码分析"
tags = ['中间件','Dubbo']
categories = ['tech']
+++

在3.2版本中，dubbo-demo-api-provider中有这样一个例子

```Java
private static void startWithBootstrap() {
    ServiceConfig<DemoServiceImpl> service = new ServiceConfig<>();
    service.setInterface(DemoService.class);
    service.setRef(new DemoServiceImpl());

    DubboBootstrap bootstrap = DubboBootstrap.getInstance();
    bootstrap
            .application(new ApplicationConfig("dubbo-demo-api-provider"))
            .registry(new RegistryConfig("zookeeper://127.0.0.1:2181"))
            .protocol(new ProtocolConfig(CommonConstants.DUBBO, -1))
            .service(service)
            .start()
            .await();
}
```

其中，DemoService和DemoServiceImpl是业务接口和其实现，它们被封装到了一个叫做ServiceConfig的泛型类的实例中。

而从书上了解到流程是由ServiceConfig->Invoker->Exporter，那我们就必须先知道springboot如何变为ServiceConfig。

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MWVkOTY3NmJjZDkwMDg2NDNkZWZlNjNhYzE2ZTExYjFfMDlsRDR5bUdET0wzR1BDZkhtRkVJMjFUTU9kSzNhTHRfVG9rZW46UGtENmJYWDRvb2s2ZGJ4ZElJcmN5TWpRbm1iXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

## springboot中如何变为serviceConfig

在Springboot中都是使用的注解@Service或者@DubboService标注在一个类的实现上，那么这个类就可以作为provider了，由此推断，只能是dubbo-spring之类的组件提前做了处理，将标有@DubboService注解的类进行了解析加工为ServiceConfig。

在@EnableDubbo注解中@DubboComponentScan注解，其内import了DubboComponentScanRegistrar类，这个类的功能是初始化了dubbo的上下文，注册了各个关键的configbean

比如ServiceAnnotationPostProcessor就是负责Provider的bean的注册

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=YzU4OGUyOGFmMzI2NGM5NmRhYzk1ZWFiZmQ3ZTQ2M2JfM0x5VlhYM2hVN0FwamN0SE9oc01sb3pmMm9KcUlPYlZfVG9rZW46Q2I4OGJnRmtJbzlqTmN4RjFxR2NxMWcyblNiXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=YTFmMDQ1M2NmZjg3ODkxZmMzYjBmMGExZmQyMDFmNzJfZUZBTnZwYnNHYTZZZ290UXo2b2d0N3d2WVFmUUdxRDZfVG9rZW46Q3FQSGJLWmpvb3dTa2J4QmpwYWMyd2tSbmJlXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

如上图所示，会从指定路径扫描标有@dubboService、还有dubbo和alibaba的两个service注解的类，然后将其注册到spring。

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NGYxMTUzZGE0YTcxNWRjMTZlOTBjNTM0YTYxNTc2MDdfVzBhS3B4eWp6SlFsSVZUYkN3cGJrRVMwUWJYWFVZWWJfVG9rZW46THNiMWJQQld6b01EVUF4MGFoQ2NPTUNsbnRmXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

而在processScannedBeanDefinition中则是解析注解属性，然后封装为另一个bean，以{ServiceBean:刚才的bean接口名}为名，ServiceBean（继承了 ServiceConfig）为类。其bean定义自定义了一个属性是ref，ref的value是原来的bean。即对于每一个dubbo的provider，都有两个bean，一个是本身的bean，另一个则是ServiceBean，这个ServiceBean持有原有bean 的引用。

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjkwNDkxZmFkODZjY2ZhYjdhOTJmOThlMTgxYjRiMTJfRmhjeFBsY3pxS1lXZ2I1MFZXQ29tZmhCUFFqOG9wYm9fVG9rZW46RUVieGJ0ZXExb0Vodnp4SlI3QWNyQVB4bkVjXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=OTEyNGY5ZDZhMDU2MWVlOTc1MGYwMTc1M2RjNmEwMTFfTTRzbzl3Z0VleEI1bXJEVFNDdmthUXhYVTFxNGdrWDVfVG9rZW46VXJUemJQWFVFb0VDbTR4YjlDc2NSWXpGbkxnXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

## export

上面提到，所有的providerBean都被定义为ServiceBean的类型，所以在实例化之后，会调用afterPropertiesSet，将bean加入到configManager中去

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NTkwMjFiOTk1NDc2ZWNjMDlmN2JlNTNjYTE2OGVmNjJfc1BncmdtN0RlUngyWTB3YXl6cEZCOVEzQ3R2eWxrSThfVG9rZW46Q0RqamJ3M3NUb3M5MG14TUpFdGNLVHhTbnhiXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=Y2JlYzYyNDcyYjYxMmUwNzhkMzM3ODZhMThiZjhjZDNfckszNFRIUTdjcEc4cm5Yb2pDSVhZOEJuU0dsbWRqaWRfVG9rZW46SkFhaGIwNDJRb2xJczR4bEtlaWMxUDBnbkRlXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

到此刻为止，bean只是在应用的本地内存中，还没有上报到注册中心去，那么是什么时候上报的呢？

DubboDeployApplicationListener，这个类在spring上下文处理发事件的时候会监听到进而执行deployer的start

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MTAzYjAzZjE3Y2MzZjA3M2UxMTEyZDNjZWZlMDJkNThfWDBiem9QNVh5QWt5SG4yQ0VGS1hObWhIWkxtUEt6bTFfVG9rZW46UHpaZmJ6amhlbzhQQnV4cW9sZGNIZGFmblRiXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

在starter方法中，exportServices中进行sc.export();由下面的时序图可知，doExportUrlsFor1Protocol是关键方法，之后则是exportLocal和exportRemote两种情况。

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MmZlNDI5NWY3YjAzNGQ4NzE2ZDY3MTYxYzg3ZmNiMDVfUHo1R0RNZ3ZEbnYxZmkzYjNvb05DWVFkbFhHa0lKNEdfVG9rZW46WXFwOWI5cFRGb2VIZmp4UTFhcGNPclFRbnJiXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

### Invoker

这里以exportRemote为例，exportRemote的核心方法是doExportUrl，其核心是将url，接口，实现类的引用封装为了一个invoker

```Java
@SuppressWarnings({"unchecked", "rawtypes"})
private void doExportUrl(URL url, boolean withMetaData) {
    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
    if (withMetaData) {
        invoker = new DelegateProviderMetaDataInvoker(invoker, this);
    }
    Exporter<?> exporter = protocolSPI.export(invoker);
    exporters.add(exporter);
}
```

其中，proxyFactory的初始化可知，其适配类的兜底方案是JavassistProxyFactory

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ODczOTc2Zjc2MWNlMjE0ZGNmNGY0Mjg1MGRkOWJjYWRfaTZ3cEp2SzdVRm5BQWhXeHB5bmhmTk1Zc1RXRFdnUHVfVG9rZW46TGxUdWJqSnUxb1Y4SUF4d0NJQmNtcnprbmVoXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGRjMjRiNDQ3ZmNiNDFiZmZhZjFhODliYWE5YjQ0YzJfeEJzdEpVYUl6aERrQkdNWGt3emFiUWI5R3lnMm5JMWpfVG9rZW46Wk5SMGJHWVdUb2hnSEh4eTF5QmNHUjRybjBlXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

因此，proxyFactory.getInvoker本质是调用JavassistProxyFactory.getInvoker

```Java
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    try {
        // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
```

而这里面又将实现类ref通过Wrapper进行了封装，为了避免多次反射调用，因此这里实际上是持有ref引用，假设实现类是TestProviderImpl，那么wrapper类就是TestProviderImplDubboWrap0，通过arthas可以获取其源码，这里主要关注invokeMethod方法。

```Java
package com.qisi.dubbo.provider.rpc;

import com.qisi.dubbo.provider.rpc.TestProviderImpl;
import java.lang.reflect.InvocationTargetException;
import java.util.Map;
import org.apache.dubbo.common.bytecode.ClassGenerator;
import org.apache.dubbo.common.bytecode.NoSuchMethodException;
import org.apache.dubbo.common.bytecode.NoSuchPropertyException;
import org.apache.dubbo.common.bytecode.Wrapper;

public class TestProviderImplDubboWrap0
extends Wrapper
implements ClassGenerator.DC {
    public static String[] pns;
    public static Map pts;
    public static String[] mns;
    public static String[] dmns;
    public static Class[] mts0;
    public static Class[] mts1;

    @Override
    public String[] getPropertyNames() {
        return pns;
    }

    @Override
    public boolean hasProperty(String string) {
        return pts.containsKey(string);
    }

    public Class getPropertyType(String string) {
        return (Class)pts.get(string);
    }

    @Override
    public String[] getMethodNames() {
        return mns;
    }

    @Override
    public String[] getDeclaredMethodNames() {
        return dmns;
    }

    @Override
    public void setPropertyValue(Object object, String string, Object object2) {
        try {
            TestProviderImpl testProviderImpl = (TestProviderImpl)object;
        }
        catch (Throwable throwable) {
            throw new IllegalArgumentException(throwable);
        }
        throw new NoSuchPropertyException(new StringBuffer().append("Not found property \"").append(string).append("\" field or setter method in class com.qisi.dubbo.provider.rpc.TestProviderImpl.").toString());
    }

    @Override
    public Object getPropertyValue(Object object, String string) {
        try {
            TestProviderImpl testProviderImpl = (TestProviderImpl)object;
        }
        catch (Throwable throwable) {
            throw new IllegalArgumentException(throwable);
        }
        throw new NoSuchPropertyException(new StringBuffer().append("Not found property \"").append(string).append("\" field or getter method in class com.qisi.dubbo.provider.rpc.TestProviderImpl.").toString());
    }

    public Object invokeMethod(Object object, String string, Class[] classArray, Object[] objectArray) throws InvocationTargetException {
        TestProviderImpl testProviderImpl;
        try {
            testProviderImpl = (TestProviderImpl)object;
        }
        catch (Throwable throwable) {
            throw new IllegalArgumentException(throwable);
        }
        try {
            if ("register".equals(string) && classArray.length == 1) {
                testProviderImpl.register((String)objectArray[0]);
                return null;
            }
            if ("beat".equals(string) && classArray.length == 1) {
                testProviderImpl.beat((String)objectArray[0]);
                return null;
            }
        }
        catch (Throwable throwable) {
            throw new InvocationTargetException(throwable);
        }
        throw new NoSuchMethodException(new StringBuffer().append("Not found method \"").append(string).append("\" in class com.qisi.dubbo.provider.rpc.TestProviderImpl.").toString());
    }
}
```

从invokeMethod方法可以看出，当调用provider方法走到了调用invoker的doInvoker时，wrapper.invokeMethod实际上就是调用ref的本身的方法，而不是通过反射。

### exporter

上面已经获取了invoker，接下来就是转成exporter了，虽然代码只有一行，但背后走的是哪些类需要具体分析。

```JavaScript
Exporter<?> exporter = protocolSPI.export(invoker);
```

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NDA4ZTA1ZmI5OGM2NDZkMGUwZWU4MWQ5OTkyOGUyZjJfMkR2dlNIS2NIOFlYZUNhQk9GWVlpVXJWRnV3YUVJTEhfVG9rZW46S0wzc2JuNURlb0plTXp4a3N2WWN3ZHpIbm1nXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=Y2QyMjRkOTBhYWU2ODNmYzg2MjQ3YjczMzViZjQzZDJfdUtJcG9GRkxsZVdCdmRZZVZDRlJreVRrY1hkVlNXUVRfVG9rZW46UVNSa2JqQURlb0N3TE54WEw1SWMyTUtubmRoXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

理论上应该走dubbo，但是实际上发现既走了registry，又走了dubbo

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MzI1MWIwNTE4NTFhYzYwYjAzNWZhZTBiZTIzOTBiMmJfRVZSejFjQ29XM3lORTlnejBpV3ZmYk9rU2RqUDNrQ1RfVG9rZW46VXU1YWJlVXdIb2FoMDd4aWQ2Y2NSbEpubjVnXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

观察Protocol$Adaptive的源码，发现优先取的是invoker的url的protocol

```Java
    public Exporter export(Invoker invoker) throws RpcException {
        String string;
        if (invoker == null) {
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        }
        if (invoker.getUrl() == null) {
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        }
        URL uRL = invoker.getUrl();
        String string2 = string = uRL.getProtocol() == null ? "dubbo" : uRL.getProtocol();
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (").append(uRL.toString()).append(") use keys([protocol])").toString());
        }
        ScopeModel scopeModel = ScopeModelUtil.getOrDefault(uRL.getScopeModel(), Protocol.class);
        Protocol protocol = scopeModel.getExtensionLoader(Protocol.class).getExtension(string);
        return protocol.export(invoker);
    }
```

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=OWE4ZjlmOTM3MzRlNzJmYmVlMDE4NWVlODM4ZWRjMjRfcWZyOG9FQk81NTQ4YUtqZGhKR0J2Q3FSTVVGaWwwZnFfVG9rZW46U3cwOGJxMmI4b3FoTGN4UnJJRGN6RzJIbkpoXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ZThiY2E3ZmQ5MjI0MjEwYzA4OWE5NTg0MGI3NjBlYzdfdmtCWjNYS1ZQZEo5UHo1bUR1TDFGNExZdEtxTlc5OFdfVG9rZW46QXdtUGJwUmpyb3FOTWx4ekNIS2NQS0JCbjhnXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

因此，实际上是先走到registry中，从url的名为export的属性中获取到providerUrl，然后调用doLocalExport才会走到dubboProtocol中

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDA0MjI4NTg5ZmRlZTEwY2NjNWU5MDlhNmJmNWU2MzlfQU9kWHM5cXNSeTM5alJyTlo5Z2U1ekZXSUNiWWRBaGxfVG9rZW46TW5aMWJhVmdEbzhRd3N4TDFld2NWYzhvbkdoXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

#### RegistryProtocol

这个类的一部分职责就是将provider的url注册到注册中心去

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NDZiZDJkYWU0Y2QxODRiMjk5MDE4Mjg3YTFkNDhhNzJfS0pkVGVvMkRhaUNaSXgwemc3NFpUODNwcW1JZWRZblRfVG9rZW46WXJLZ2Jib1hEb3lNZ0R4SGhVNGNRMmMwblRjXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

```Java
public void doRegister(URL url) {
    try {
        checkDestroyed();
        zkClient.create(toUrlPath(url), url.getParameter(DYNAMIC_KEY, true));
    } catch (Throwable e) {
        throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}

dubbo://192.168.31.16:20880/com.qisi.dubbo.api.TestProvider?anyhost=true&application=dubbo-provider-boot&background=false&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.qisi.dubbo.api.TestProvider&methods=beat,register&pid=69078&release=3.0.7&revision=1.0.0&service-name-mapping=true&service.filter=provider&side=provider&timestamp=1710828709106&version=1.0.0
```

#### DubboProtocol

职责是构造了exporter，key为（端口+接口+版本+组），value为exporter

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ZmQ2NThhZjY2ZmM0NWUzZDYzYzY2ZDk3YzFjMWRlYTFfT0tJSWpUZkhWREZhTWdJQWpHSGVYN3Iwand4a285RUxfVG9rZW46S05LQWJpWVJyb3NYS3R4RXlnSWM0R0tGbmllXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=OWQ0MGMyMjNhMDkwZTNjNDYwNjNkYjI1NTc2YmY0OTZfYmJKdVV2TzRaUURPRGc1WGRyc08wZHgwa09uUTZuOXRfVG9rZW46RzVXN2IyRFFZbzM0Sk94R1lkSWNDeHVKblFjXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)

当服务端接收到请求时，会从exporterMap中取出exporter，然后取出invoker

org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol#getInvoker

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NGZlMzAxYjg1OGY4MGQ5NWE3YTRjZDJjZjc2MGUyNjRfaVIwbDg5dXpvcnVxNzJwSmh1eUVLU2RjamtvaEM4VDFfVG9rZW46U29ZcGJJaHJWb3N2cVh4RjBmMmNwRFlVbkloXzE3MjMzNjk4NjA6MTcyMzM3MzQ2MF9WNA)
