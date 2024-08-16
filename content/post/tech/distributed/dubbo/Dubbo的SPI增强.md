+++
title = "Dubbo的SPI增强"
description = ""
date = 2024-03-17
image = ""
draft = false
slug = "Dubbo_SPI"
tags = ['SPI','Dubbo','中间件']
categories = ['tech']
+++

[自定义扩展](https://cn.dubbo.apache.org/zh-cn/overview/tasks/extensibility/)

Jdk的SPI缺点在于需要遍历所有的扩展类并实例化，dubbo对此做了优化。

1. 使其支持按需实例化，其本质是将所有的扩展类Class存到map，通过getExtension(key)来实例化。

2. 支持IOC，使得任意扩展可以任意组合

3. 支持AOP，使得扩展可以被前后加强

## @SPI

```Java
ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension();

public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
    }
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type (" + type +
                ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
    }

    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```

dubbo中，所有扩展的接口都标有@SPI注解，使用ExtensionLoader来进行获取，ExtensionLoader被设计为不可被外部实例化，只能通过getExtensionLoader来获取并存储到了map中，因此每个类型都是不同的ExtensionLoader实力对象。

### getExtensionClasses

扫描所有的扩展类本质上是扫描META-INF/services/，META-INF/dubbo/internal，META-INF/dubbo/三个目录下以org.apache或者com.alibaba为前缀的配置文件，并将class信息存储到cachedClasses持有的map中

```Java
private Map<String, Class<?>> loadExtensionClasses() {
    cacheDefaultExtensionName();

    Map<String, Class<?>> extensionClasses = new HashMap<>();
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    return extensionClasses;
}
```

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MjJjZGYwZjI2NGZlODlmNjYzZGE5OWY3YjRjMGExZDJfWVJKcUliRWEyQUtBRlkxdjdJVjhKT0ZWZHdYWmR3MjFfVG9rZW46QmIxNWJIRzZMbzZhZ254d0Q4SWMwaXhObnFoXzE3MjMzNjkyNDA6MTcyMzM3Mjg0MF9WNA)

### @Adapter

SPI可以直接使用导入的扩展类，也可以使用适配器来进行切换。

```JavaScript
ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=OGQ4ZGZjYjFlZDlmNzZlNzY0MTQwMTZhN2QyZDgyMzlfbVhyb1QyWVRmQVA3bVE1VnBRbUVMVEZ1TTdjMkt1WGNfVG9rZW46UXdPb2JJV0Nab1lSN0N4TFZkNWNyREZmbll5XzE3MjMzNjkyNDA6MTcyMzM3Mjg0MF9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NjdmOWM3NTcxZWViM2E1YTAzYTE1ZGVkNzRmZjQ3OGRfa3ZBU1BvbWphcFhwOENDT296b0dYeUZvT1JCdjdvUUtfVG9rZW46TXNObGIzYm9ob2ZsN0t4Rzh1VGNsTzdhbmNnXzE3MjMzNjkyNDA6MTcyMzM3Mjg0MF9WNA)

以Protocol为例，其默认协议为dubbo，但使用的时候就是转换为了适配器。在通过AdaptiveClassCodeGenerator生成类的字符串后，通过Comiler编译为实际的class，然后实例化，并进行IOC装配。其Adaptive的代码如下所示，主要是对方法上有@Adapter的方法进行编写，其优先取用url中的协议，然后使用dubbo来进行兜底。

```Java
package com.books.dubbo.demo.provider;

import org.apache.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException("The method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("The method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }

    public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }
}
```

对于API层来说，provider进行暴露的时候调用protocol.export实际上就是调用Protocol$Adaptive的export方法，如果是dubbo，那么就是DubboProtocol;如果是thrift，那么就是ThriftProtocol。

在Protocol$Adaptive中会再次通过ExtensionLoader.*getExtensionLoader*(Protocol.class).getExtension(extName)来找到对应的实现类然后进行调用。

```Java
private T createExtension(String name) {
    //在map中找到对应的类
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
        //实例化
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        //进行IOC装配
        injectExtension(instance);
        //进行AOP增强
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}
```

### IOC

ioc的本质实际上给扩展类中的set方法进行赋值

```Java
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            for (Method method : instance.getClass().getMethods()) {
                if (isSetter(method)) {
                    /**
                     * Check {@link DisableInject} to see if we need auto injection for this property
                     */
                    if (method.getAnnotation(DisableInject.class) != null) {
                        continue;
                    }
                    //如果是setProtocol，那么入参肯定是Protocol.class
                    Class<?> pt = method.getParameterTypes()[0];
                    if (ReflectUtils.isPrimitives(pt)) {
                        continue;
                    }
                    try {
                        //setProtocol的话就是protocol
                        String property = getSetterProperty(method);
                        //所以实际上是在扩展中寻找类型为Protocol.class和name为protocol的实例
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
```

### AOP

AOP的典型应用场景是Filter，在invoke的前后进行一些业务操作。

在扩展类中，有一类扩展是Wrapper类，在loadResource中的loadClass

```Java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + " is not subtype of interface.");
    }
    //适配类
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        cacheAdaptiveClass(clazz);
    //包装类
    } else if (isWrapperClass(clazz)) {
        cacheWrapperClass(clazz);
    //普通扩展类
    } else {
        if (ArrayUtils.isNotEmpty(names)) {
        //如果有@Activate注解进行记录
        cacheActivateClass(clazz, names[0]);
        for (String n : names) {
            cacheName(clazz, n);
            saveInExtensionClass(extensionClasses, clazz, name);
        }
    }
    //决定是否是包装类的条件是，存在构造函数的入参为指定的类型
    private boolean isWrapperClass(Class<?> clazz) {
    try {
        clazz.getConstructor(type);
        return true;
    } catch (NoSuchMethodException e) {
        return false;
    }

    //如果是的话就添加到了cachedWrapperClasses这个map中
    private void cacheWrapperClass(Class<?> clazz) {
    if (cachedWrapperClasses == null) {
        cachedWrapperClasses = new ConcurrentHashSet<>();
    }
    cachedWrapperClasses.add(clazz);
}
```

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NDA5YzMwMTZhNGE5OWU0MTY0NzRiMmE4MjkzZjdlY2RfQ2pvSHR3UXJVMHVoQWk5NFNOSkdTd0FoQWRqMlFBQmZfVG9rZW46TzhaYmJKcHFjb0t2dDJ4UzkxbmNZM0ZXbkNkXzE3MjMzNjkyNDA6MTcyMzM3Mjg0MF9WNA)

像这种情况就是，ProtocolFilterWrapper是Protocol的包装类，回到之前的createExtension方法，就发现是实例化的包装类

```Java
Set<Class<?>> wrapperClasses = cachedWrapperClasses;
if (CollectionUtils.isNotEmpty(wrapperClasses)) {
    for (Class<?> wrapperClass : wrapperClasses) {
        instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
    }
}
```

### 激活多个扩展

上述的情况在使用的时候都是使用单个扩展，通过名字来指定。但是有些场景下是会同时实例化好多扩展的，这种场景下一般都是标为@Activate注解的扩展类。

会根据dubbo对应的url进行分析，需要激活那些扩展。

```Java
public List<T> getActivateExtension(URL url, String[] values, String group) {
    List<T> exts = new ArrayList<>();
    List<String> names = values == null ? new ArrayList<>(0) : Arrays.asList(values);
    if (!names.contains(Constants.REMOVE_VALUE_PREFIX + Constants.DEFAULT_KEY)) {
        getExtensionClasses();
        for (Map.Entry<String, Object> entry : cachedActivates.entrySet()) {
            String name = entry.getKey();
            Object activate = entry.getValue();

            String[] activateGroup, activateValue;

            if (activate instanceof Activate) {
                activateGroup = ((Activate) activate).group();
                activateValue = ((Activate) activate).value();
            } else if (activate instanceof com.alibaba.dubbo.common.extension.Activate) {
                activateGroup = ((com.alibaba.dubbo.common.extension.Activate) activate).group();
                activateValue = ((com.alibaba.dubbo.common.extension.Activate) activate).value();
            } else {
                continue;
            }
            //根据url进行匹配，只有实际需要的才会添加
            if (isMatchGroup(group, activateGroup)) {
                T ext = getExtension(name);
                if (!names.contains(name)
                        && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)
                        && isActive(activateValue, url)) {
                    exts.add(ext);
                }
            }
        }
        exts.sort(ActivateComparator.COMPARATOR);
    }
    List<T> usrs = new ArrayList<>();
    for (int i = 0; i < names.size(); i++) {
        String name = names.get(i);
        if (!name.startsWith(Constants.REMOVE_VALUE_PREFIX)
                && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)) {
            if (Constants.DEFAULT_KEY.equals(name)) {
                if (!usrs.isEmpty()) {
                    exts.addAll(0, usrs);
                    usrs.clear();
                }
            } else {
                T ext = getExtension(name);
                usrs.add(ext);
            }
        }
    }
    if (!usrs.isEmpty()) {
        exts.addAll(usrs);
    }
    return exts;
}
```
