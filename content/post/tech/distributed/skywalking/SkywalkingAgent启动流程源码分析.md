+++
title = "SkywalkingAgent启动流程源码分析"
description = ""
date = 2023-08-29
image = ""
draft = false
slug = "SkywalkingAgent"
tags = ['中间件','Skywalking','监控']
categories = ['tech']
+++

Skywalking 从架构上分为三个大模块，分别是Agent、Oap（Backend&Storage）、UI。这篇文档主要是介绍Agent部分。这部分核心依赖的是java的agent技术，对这个技术陌生的可以先看一下[Agent]({{<ref "/post/tech/lang/JavaAgent技术">}}) 这篇文章。Skywalking在agent的基础上利用[ByteBuddy](http://bytebuddy.net/#/tutorial-cn)类库对类和方法等进行加强，实现无入侵的监测。

## SkyWalkingAgent类

在官方的skywalking-agent包中，通过MANIFEST.MF的Premain-Class值，我们可以确定入口类是org.apache.skywalking.apm.agent.SkyWalkingAgent，而核心的方法就是里面的premain方法，大概的内容是以下四部分。

1. 读取并整合配置

2. 加载插件

3. 通过AgentBuilder注入到jvm

4. 启动上报相关的服务等

### 读取并整合配置

```java
//将各个途径的配置都处理汇总到CONFIG类里，CONFIG类里面全是静态字段，所以直接赋值也不需要实例化对象
SnifferConfigInitializer.initializeCoreConfig(agentArgs);


public static void initializeCoreConfig(String agentOptions) {
    AGENT_SETTINGS = new Properties();
    //loadConfig方法中会优先从指定的系统属性skywalking_config中取值（即 java -Dskywalking_config=xxx/custom.config)
    //如果没有设置，则默认加载SKYWALKING_AGENT_PATH/config/agent.config
    try (final InputStreamReader configFileStream = loadConfig()) {
        AGENT_SETTINGS.load(configFileStream);
        for (String key : AGENT_SETTINGS.stringPropertyNames()) {
            String value = (String) AGENT_SETTINGS.get(key);
            //替换占位符的变量，依次从System.getProperty(key)->System.getenv(key)->properties.getProperty(key)->default取
            AGENT_SETTINGS.put(key, PropertyPlaceholderHelper.INSTANCE.replacePlaceholders(value, AGENT_SETTINGS));
        }

    } catch (Exception e) {
        LOGGER.error(e, "Failed to read the config file, skywalking is going to run in default config.");
    }

    try {
        //替换通过java -Dskywalking.指定的变量值
        //这里我觉得很奇怪，明明上面也是处理系统属性，这里为什么还要专门用skywalking前缀区分呢？
        overrideConfigBySystemProp();
    } catch (Exception e) {
        LOGGER.error(e, "Failed to read the system properties.");
    }

    agentOptions = StringUtil.trim(agentOptions, ',');
    if (!StringUtil.isEmpty(agentOptions)) {
        try {
            agentOptions = agentOptions.trim();
            LOGGER.info("Agent options is {}.", agentOptions);
            //替换在 java -agent:/***.jar=key:value里的值
            overrideConfigByAgentOptions(agentOptions);
        } catch (Exception e) {
            LOGGER.error(e, "Failed to parse the agent options, val is {}.", agentOptions);
        }
    }
    //通过反射将AGENT_SETTINGS注入到Config的变量中
    initializeConfig(Config.class);
    ...
}
```

### 加载插件

```java
//这一步实际上是加载了插件jar包里的增加类def里面描述的类，最终都传到了PluginFinder，存储为三种不同的对象
pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());


public List<AbstractClassEnhancePluginDefine> loadPlugins() throws AgentPackageNotFoundException {
    //可以通过启动一次看日志，了解这里是先装载jar包再识别出def
    //注意，这里自定义了一个类加载器,并且加载了activations和plugins/plugins下的jar包
    //Q&A 2023/8/27
    // Q:为什么要自定义类加载器
    // A:为了加载插件的那些jar包，继承自AppClassLodder
    AgentClassLoader.initDefaultLoader();

    PluginResourcesResolver resolver = new PluginResourcesResolver();
    //获取所有的skywalking-plugin.def，由于上面的jar包已经添加，所以实际就是那些jar包里的skywalking-plugin.def
    List<URL> resources = resolver.getResources();

    if (resources == null || resources.size() == 0) {
        LOGGER.info("no plugin files (skywalking-plugin.def) found, continue to start application.");
        return new ArrayList<AbstractClassEnhancePluginDefine>();
    }

    for (URL pluginUrl : resources) {
        try {
            //最终都加载到了List<PluginDefine> pluginClassList里
            PluginCfg.INSTANCE.load(pluginUrl.openStream());
        } catch (Throwable t) {
            LOGGER.error(t, "plugin file [{}] init failure.", pluginUrl);
        }
    }

    List<PluginDefine> pluginClassList = PluginCfg.INSTANCE.getPluginClassList();
    List<AbstractClassEnhancePluginDefine> plugins = new ArrayList<AbstractClassEnhancePluginDefine>();
    for (PluginDefine pluginDefine : pluginClassList) {
        try {
            LOGGER.debug("loading plugin class {}.", pluginDefine.getDefineClass());
            //Q&A 2023/8/22
            // Q:为什么这里所有的类都能强转为AbstractClassEnhancePluginDefine
            // A:因为插件继承的顶层最终都是这个抽象类,见下面类图
            AbstractClassEnhancePluginDefine plugin = (AbstractClassEnhancePluginDefine) Class.forName(pluginDefine.getDefineClass(), true, AgentClassLoader
                .getDefault()).newInstance();
            plugins.add(plugin);
        } catch (Throwable t) {
            LOGGER.error(t, "load plugin [{}] failure.", pluginDefine.getDefineClass());
        }
    }
    //这个是通过SPI用所有的InstrumentationLoader来添加AbstractClassEnhancePluginDefine，不知道用在哪
    plugins.addAll(DynamicPluginLoader.INSTANCE.load(AgentClassLoader.getDefault()));

    return plugins;

}
//这里面将插件分到了三个不同的集合中
public PluginFinder(List<AbstractClassEnhancePluginDefine> plugins) {
    for (AbstractClassEnhancePluginDefine plugin : plugins) {
        ClassMatch match = plugin.enhanceClass();

        if (match == null) {
            continue;
        }
        //重写enhanceClass，以restTemplate为例，enhanceClass是NameMatch.byName("org.springframework.web.client.RestTemplate")，所以对应map的key就是"org.springframework.web.client.RestTemplate"
        if (match instanceof NameMatch) {
            NameMatch nameMatch = (NameMatch) match;
            LinkedList<AbstractClassEnhancePluginDefine> pluginDefines = nameMatchDefine.get(nameMatch.getClassName());
            if (pluginDefines == null) {
                pluginDefines = new LinkedList<AbstractClassEnhancePluginDefine>();
                nameMatchDefine.put(nameMatch.getClassName(), pluginDefines);
            }
            pluginDefines.add(plugin);
        } else {
            signatureMatchDefine.add(plugin);
        }

        if (plugin.isBootstrapInstrumentation()) {
            bootstrapClassMatchDefine.add(plugin);
        }
    }
}
```

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=YjE3OGRlMGM3ZGVhNzA0MjJhYThlOWY2NTIyYTcwOTNfSURnVFAxZnlCWUxONDZuSHhEZDNnVUhOY1J0YWpmQnZfVG9rZW46VzdTU2JHUGNub01zVk94eGdZb2NOVWt6bkhnXzE3MjMzNzA1ODI6MTcyMzM3NDE4Ml9WNA)

### AgentBuilder

这中间有一部分是JDK9的模块相关的内容，由于我还没有深入了解过JDK9相关的内容，加上

```java
//这一步实际上是加载了插件jar包里的增加类def里面描述的类，最终都传到了PluginFinder，存储为三种不同的对象
pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());


public List<AbstractClassEnhancePluginDefine> loadPlugins() throws AgentPackageNotFoundException {
    //可以通过启动一次看日志，了解这里是先装载jar包再识别出def
    //注意，这里自定义了一个类加载器,并且加载了activations和plugins/plugins下的jar包
    //Q&A 2023/8/27
    // Q:为什么要自定义类加载器
    // A:为了加载插件的那些jar包，继承自AppClassLodder
    AgentClassLoader.initDefaultLoader();

    PluginResourcesResolver resolver = new PluginResourcesResolver();
    //获取所有的skywalking-plugin.def，由于上面的jar包已经添加，所以实际就是那些jar包里的skywalking-plugin.def
    List<URL> resources = resolver.getResources();

    if (resources == null || resources.size() == 0) {
        LOGGER.info("no plugin files (skywalking-plugin.def) found, continue to start application.");
        return new ArrayList<AbstractClassEnhancePluginDefine>();
    }

    for (URL pluginUrl : resources) {
        try {
            //最终都加载到了List<PluginDefine> pluginClassList里
            PluginCfg.INSTANCE.load(pluginUrl.openStream());
        } catch (Throwable t) {
            LOGGER.error(t, "plugin file [{}] init failure.", pluginUrl);
        }
    }

    List<PluginDefine> pluginClassList = PluginCfg.INSTANCE.getPluginClassList();
    List<AbstractClassEnhancePluginDefine> plugins = new ArrayList<AbstractClassEnhancePluginDefine>();
    for (PluginDefine pluginDefine : pluginClassList) {
        try {
            LOGGER.debug("loading plugin class {}.", pluginDefine.getDefineClass());
            //Q&A 2023/8/22
            // Q:为什么这里所有的类都能强转为AbstractClassEnhancePluginDefine
            // A:因为插件继承的顶层最终都是这个抽象类,见下面类图
            AbstractClassEnhancePluginDefine plugin = (AbstractClassEnhancePluginDefine) Class.forName(pluginDefine.getDefineClass(), true, AgentClassLoader
                .getDefault()).newInstance();
            plugins.add(plugin);
        } catch (Throwable t) {
            LOGGER.error(t, "load plugin [{}] failure.", pluginDefine.getDefineClass());
        }
    }
    //这个是通过SPI用所有的InstrumentationLoader来添加AbstractClassEnhancePluginDefine，不知道用在哪
    plugins.addAll(DynamicPluginLoader.INSTANCE.load(AgentClassLoader.getDefault()));

    return plugins;

}
//这里面将插件分到了三个不同的集合中
public PluginFinder(List<AbstractClassEnhancePluginDefine> plugins) {
    for (AbstractClassEnhancePluginDefine plugin : plugins) {
        ClassMatch match = plugin.enhanceClass();

        if (match == null) {
            continue;
        }
        //重写enhanceClass，以restTemplate为例，enhanceClass是NameMatch.byName("org.springframework.web.client.RestTemplate")，所以对应map的key就是"org.springframework.web.client.RestTemplate"
        if (match instanceof NameMatch) {
            NameMatch nameMatch = (NameMatch) match;
            LinkedList<AbstractClassEnhancePluginDefine> pluginDefines = nameMatchDefine.get(nameMatch.getClassName());
            if (pluginDefines == null) {
                pluginDefines = new LinkedList<AbstractClassEnhancePluginDefine>();
                nameMatchDefine.put(nameMatch.getClassName(), pluginDefines);
            }
            pluginDefines.add(plugin);
        } else {
            signatureMatchDefine.add(plugin);
        }

        if (plugin.isBootstrapInstrumentation()) {
            bootstrapClassMatchDefine.add(plugin);
        }
    }
}
```

这里开始就要用到ByteBuddy了，先简单了解下[AgentBuilder]() ，然后我们结合插件分析源码中的核心部分

比较关键的其实就是type、transform和installOn三个方法，其他的ignore、with感兴趣的可以自己研究，不影响主流程的理解

#### type()

type是命中要处理的类，对应的是插件里的enhanceClass，如果enhanceClass实现是指定的类名称，那么在之前已经加入到nameMatchDefine里了，如果不是则各自解析，比如Spring的插件就是通过注解来匹配

```java
protected ClassMatch enhanceClass() {
    return ClassAnnotationMatch.byClassAnnotationMatch(this.getEnhanceAnnotations());
}
protected String[] getEnhanceAnnotations() {
    return new String[]{"org.springframework.web.bind.annotation.RestController"};
}
```

```java
/**
 * 主要是通过插件里定义的NameMatch和IndirectMatch来匹配，IndirectMatch就是要更复杂，比如匹配的更多，前缀，后缀之类的，参考IndirectMatch的实现类
 * @return
 */
public ElementMatcher<? super TypeDescription> buildMatch() {
    ElementMatcher.Junction judge = new AbstractJunction<NamedElement>() {
        @Override
        public boolean matches(NamedElement target) {
            return nameMatchDefine.containsKey(target.getActualName());
        }
    };
    judge = judge.and(not(isInterface()));
    for (AbstractClassEnhancePluginDefine define : signatureMatchDefine) {
        ClassMatch match = define.enhanceClass();
        if (match instanceof IndirectMatch) {
            judge = judge.or(((IndirectMatch) match).buildJunction());
        }
    }
    return new ProtectiveShieldMatcher(judge);
```

#### transform()

```Java
public DynamicType.Builder<?> define(TypeDescription typeDescription, DynamicType.Builder<?> builder,
...
    WitnessFinder finder = WitnessFinder.INSTANCE;
    //witness机制，希望加强某类之前，必须有些类已经加载，用于区分版本，让插件和应用的版本所对应
    /**
     * find witness classes for enhance class
     */
    String[] witnessClasses = witnessClasses();
    if (witnessClasses != null) {
        for (String witnessClass : witnessClasses) {
            if (!finder.exist(witnessClass, classLoader)) {
                LOGGER.warn("enhance class {} by plugin {} is not activated. Witness class {} does not exist.", transformClassName, interceptorDefineClassName, witnessClass);
                return null;
            }
        }
    }
    List<WitnessMethod> witnessMethods = witnessMethods();
    if (!CollectionUtil.isEmpty(witnessMethods)) {
        for (WitnessMethod witnessMethod : witnessMethods) {
            if (!finder.exist(witnessMethod, classLoader)) {
                LOGGER.warn("enhance class {} by plugin {} is not activated. Witness method {} does not exist.", transformClassName, interceptorDefineClassName, witnessMethod);
                return null;
            }
        }
    }

    /**
     * find origin class source code for interceptor
     */
    DynamicType.Builder<?> newClassBuilder = this.enhance(typeDescription, builder, classLoader, context);
...
}
```

transform是增强命中的类，这里核心方法是org.apache.skywalking.apm.agent.SkyWalkingAgent.Transformer#transform->

org.apache.skywalking.apm.agent.core.plugin.AbstractClassEnhancePluginDefine#define

##### Witness机制

这里需要提及一个机制，即[Witness机制](https://skywalking.apache.org/docs/skywalking-java/next/en/setup/service-agent/java-agent/java-plugin-development-guide/#implement-plugin)，他的作用是让引入的插件必须和当前应用组件的版本必须相对应。举例来说，spring-mvc有三个版本的插件，如下面代码所示，不同的版本校验的类并不相同。(但是我去查了spring3456每个版本的类，好像并不太能对应到spring的版本，所以我也不知道skywalking官方的这个mvc插件对应的哪个版本)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MDQ4NWZhNmUxODk5ZWRjNjdlMTljMWY1ZWJmYjRjMGJfa1FsZDNTMUt0VFF2c0tKMnR2TDFGRDRGSG5XSzlRMXpfVG9rZW46QWxQUGI3dEk5bzd3Wmd4Z1NYdGM0OVBKbk1wXzE3MjMzNzA1ODI6MTcyMzM3NDE4Ml9WNA)

```java
public abstract class AbstractSpring3Instrumentation extends ClassInstanceMethodsEnhancePluginDefine {
    public static final String WITHNESS_CLASSES = "org.springframework.web.servlet.view.xslt.AbstractXsltView";

    public AbstractSpring3Instrumentation() {
    }

    protected final String[] witnessClasses() {
        return new String[]{"org.springframework.web.servlet.view.xslt.AbstractXsltView"};
    }
}
public abstract class AbstractSpring4Instrumentation extends ClassInstanceMethodsEnhancePluginDefine {
    public static final String WITHNESS_CLASSES = "org.springframework.cache.interceptor.SimpleKey";

    public AbstractSpring4Instrumentation() {
    }

    protected String[] witnessClasses() {
        return new String[]{"org.springframework.cache.interceptor.SimpleKey", "org.springframework.cache.interceptor.DefaultKeyGenerator"};
    }
}
public abstract class AbstractSpring5Instrumentation extends ClassInstanceMethodsEnhancePluginDefine {
    public static final String WITNESS_CLASSES = "org.springframework.beans.annotation.AnnotationBeanUtils";

    public AbstractSpring5Instrumentation() {
    }

    protected final String[] witnessClasses() {
        return new String[]{"org.springframework.beans.annotation.AnnotationBeanUtils"};
    }
}
```

##### 拦截器

enhance方法里面包含了两个部分，enhanceClass主要是加强类的静态方法，enhanceInstance则是加强类的构造方法和普通方法.

这样给插件留出更细粒度的扩展，只需要插件实现InstanceMethodsInterceptPoint等接口就可以了

```Java
protected DynamicType.Builder<?> enhance(TypeDescription typeDescription, DynamicType.Builder<?> newClassBuilder,
                                         ClassLoader classLoader, EnhanceContext context) throws PluginException {
    //这里是实际和插件作用的地方
    newClassBuilder = this.enhanceClass(typeDescription, newClassBuilder, classLoader);

    newClassBuilder = this.enhanceInstance(typeDescription, newClassBuilder, classLoader, context);

    return newClassBuilder;
}
protected DynamicType.Builder<?> enhanceInstance(TypeDescription typeDescription,
    DynamicType.Builder<?> newClassBuilder, ClassLoader classLoader,
    EnhanceContext context) throws PluginException {
...

    /**
     * Manipulate class source code.<br/>
     *
     * new class need:<br/>
     * 1.Add field, name {@link #CONTEXT_ATTR_NAME}.
     * 2.Add a field accessor for this field.
     *
     * And make sure the source codes manipulation only occurs once.
     *
     */
//生成一个新的类，该类继承EnhancedInstance接口，并且自定义了一个字段叫做_$EnhancedClassField_ws，同时生成了这个的set、get方法
//Q&A 2023/8/29
// Q: 为什么非要指定这样一个字段呢？
// A: 说是为了解决上下文问题，需要这样一个字段缓存，但是我不理解 https://github.com/apache/skywalking/issues/7146
    if (!typeDescription.isAssignableTo(EnhancedInstance.class)) {
        if (!context.isObjectExtended()) {
            newClassBuilder = newClassBuilder.defineField(
                CONTEXT_ATTR_NAME, Object.class, ACC_PRIVATE | ACC_VOLATILE)
                                             .implement(EnhancedInstance.class)
                                             .intercept(FieldAccessor.ofField(CONTEXT_ATTR_NAME));
            context.extendObjectCompleted();
        }
    }
    //下面的增强构造和实例方法其实很类似
    //核心点在于InstanceMethodsInterceptPoint，
    //通过getMethodsMatcher获取需要加强的方法
    //通过getMethodsInterceptor获取需要插件定义的拦截器，然后将其构造到agentBuilder里

    /**
     * 2. enhance constructors
     */
...
    /**
     * 3. enhance instance methods
     */

     ...
     newClassBuilder = newClassBuilder.method(junction)
                        .intercept(MethodDelegation.withDefaultConfiguration()
                        .to(new InstMethodsInter(interceptor, classLoader)));
    }

    return newClassBuilder;
}
```

这里不对agentBuidle的intercept等具体分析，感兴趣的可以自己看

#### installOn()

正常定义permain方法时，定义的transformer都是通过addTransformer添加到instrumentation的

```Java
public static void premain(String agentArgs, Instrumentation instrumentation) {
    System.out.println("enhance by agent,params:"+agentArgs);
    instrumentation.addTransformer(new ClassFileTransformer() {
        @Override
        public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
                                ProtectionDomain protectionDomain, byte[] classfileBuffer)
                throws IllegalClassFormatException {
            System.out.println("premain load Class     :" + className);
            return classfileBuffer;
        }
    }, true);
}
```

而这里，我们用的是agentBuilder,在transform()部分我们已经知道定义了transformer,其transform方法是遍历插件定义然后加强类，因此我们可以猜测最终肯定也是将transformer添加到instrumentation，最简单的映证方法就是在sun.instrument.InstrumentationImpl#addTransformer(java.lang.instrument.ClassFileTransformer, boolean)打个断点，启动项目

installon()->org.apache.skywalking.apm.dependencies.net.bytebuddy.agent.builder.AgentBuilder.Default#doInstall()

```Java
private ResettableClassFileTransformer doInstall(Instrumentation instrumentation, AgentBuilder.RawMatcher matcher, AgentBuilder.PatchMode.Handler handler) {
    if (!this.circularityLock.acquire()) {
        throw new IllegalStateException("Could not acquire the circularity lock upon installation.");
    } else {
        ResettableClassFileTransformer var13;
        try {
            AgentBuilder.RedefinitionStrategy.ResubmissionStrategy.Installation installation = this.redefinitionResubmissionStrategy.apply(instrumentation, this.poolStrategy, this.locationStrategy, this.descriptionStrategy, this.fallbackStrategy, this.listener, this.installationListener, this.circularityLock, new AgentBuilder.Default.Transformation.SimpleMatcher(this.ignoreMatcher, this.transformations), this.redefinitionStrategy, this.redefinitionBatchAllocator, this.redefinitionListener);
            ResettableClassFileTransformer classFileTransformer = this.transformerDecorator.decorate(this.makeRaw(installation.getListener(), installation.getInstallationListener(), installation.getResubmissionEnforcer()));
            installation.getInstallationListener().onBeforeInstall(instrumentation, classFileTransformer);

            try {
                this.warmupStrategy.apply(classFileTransformer, this.locationStrategy, this.redefinitionStrategy, this.circularityLock, installation.getInstallationListener());
                handler.onBeforeRegistration(instrumentation);
                if (this.redefinitionStrategy.isRetransforming()) {
                    DISPATCHER.addTransformer(instrumentation, classFileTransformer, true);
                } else {
                    instrumentation.addTransformer(classFileTransformer);
                }
```

启动上报相关的服务

暂未研究

## 参考文档:

[Skywalking8.9.1源码解析<六>-Skywalking-agent版本识别机制](https://zhuanlan.zhihu.com/p/563375170)
