+++
title = "RefershScope源码分析"
description = ""
date = 2024-05-26
image = ""
draft = false
slug = "RefershScope"
tags = ['Spring']
categories = ['tech']
+++

Spring中是存在多个Scope的，比如大部分的bean都是单例(Singleton)，还有一些不常用的比如原型(Prototype)、request、session、thread、servlet以及refresh。除了单例之外，其他的都是表示在某一个限定的范围内，是相同的bean，比如原型表示每次都是全新的bean，request表示每一次request是全新的bean。

RefreshScope，则表示每次Refresh的时候，都会获取全新的bean。这个注解的核心在于Scope注解，且value是refresh。

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Scope("refresh")
@Documented
public @interface RefreshScope {

    /**
     * @see Scope#proxyMode()
     * @return proxy mode
     */
     //默认是cglib代理
    ScopedProxyMode proxyMode() default ScopedProxyMode.TARGET_CLASS;

}
```

## 注册代理bean

标有Scope的类，无论是那种加载beanDefinition的情况，在注册beanDefinition的时候会注册成代理bean并持有scope范围的bean。例如，bean名字是simpleBean,那么在spring容器中会注册两个bean，scopedTarget.simpleBean(非单例）和simpleBean（单例）。所有的依赖，如@Autowired和@Value类的属性，都只会注入到simpleBean中

这里以ClassPathBeanDefinitionScanner.doScan为例。

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=OTQ3MzNjYzZhOGU2ZmRlOTRhODcyMzI4ODQ5NTAzMjJfc2JWM2JsVGlUb1NQeHl1RGR6RUxrcE5UTVVIaVo5UWtfVG9rZW46V0pIMmIyelZsb3V3Y0J4YWsydGNiYnlZbm5lXzE3MjMzNjcxNDU6MTcyMzM3MDc0NV9WNA)

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
       Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
       for (BeanDefinition candidate : candidates) {
       //解析Scope注解的元数据信息
          ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
          candidate.setScope(scopeMetadata.getScopeName());
          String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
          if (candidate instanceof AbstractBeanDefinition) {
             postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
          }
          if (candidate instanceof AnnotatedBeanDefinition) {
             AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
          }
          if (checkCandidate(beanName, candidate)) {
             BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
             //额外处理
             definitionHolder =
                   AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
             beanDefinitions.add(definitionHolder);
             registerBeanDefinition(definitionHolder, this.registry);
          }
       }
    }
    return beanDefinitions;
}
```

需要关注的就两个方法resolveScopeMetadata和AnnotationConfigUtils.*applyScopedProxyMode*

```java
//就是组装注解元数据信息
public ScopeMetadata resolveScopeMetadata(BeanDefinition definition) {
    ScopeMetadata metadata = new ScopeMetadata();
    if (definition instanceof AnnotatedBeanDefinition) {
       AnnotatedBeanDefinition annDef = (AnnotatedBeanDefinition) definition;
       AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(
             annDef.getMetadata(), this.scopeAnnotationType);
       if (attributes != null) {
          metadata.setScopeName(attributes.getString("value"));
          ScopedProxyMode proxyMode = attributes.getEnum("proxyMode");
          if (proxyMode == ScopedProxyMode.DEFAULT) {
             proxyMode = this.defaultProxyMode;
          }
          metadata.setScopedProxyMode(proxyMode);
       }
    }
    return metadata;
}
static BeanDefinitionHolder applyScopedProxyMode(
       ScopeMetadata metadata, BeanDefinitionHolder definition, BeanDefinitionRegistry registry) {

    ScopedProxyMode scopedProxyMode = metadata.getScopedProxyMode();
    //为no表示不代理
    if (scopedProxyMode.equals(ScopedProxyMode.NO)) {
       return definition;
    }
    //是否是cglib代理
    boolean proxyTargetClass = scopedProxyMode.equals(ScopedProxyMode.TARGET_CLASS);
    return ScopedProxyCreator.createScopedProxy(definition, registry, proxyTargetClass);
}
//这段代码的主要作用就是将bean本身注册为了一个代理bean，且这个代理bean持有一个名字为scopedTarget.+原名字的内部bean
public static BeanDefinitionHolder createScopedProxy(BeanDefinitionHolder definition,
       BeanDefinitionRegistry registry, boolean proxyTargetClass) {

    String originalBeanName = definition.getBeanName();
    BeanDefinition targetDefinition = definition.getBeanDefinition();
    //代理bean的名字是scopedTarget.+原名字
    String targetBeanName = getTargetBeanName(originalBeanName);

    // Create a scoped proxy definition for the original bean name,
    // "hiding" the target bean in an internal target definition.
    RootBeanDefinition proxyDefinition = new RootBeanDefinition(ScopedProxyFactoryBean.class);
    proxyDefinition.setDecoratedDefinition(new BeanDefinitionHolder(targetDefinition, targetBeanName));
    //持有targetBean
    proxyDefinition.setOriginatingBeanDefinition(targetDefinition);
    proxyDefinition.setSource(definition.getSource());
    proxyDefinition.setRole(targetDefinition.getRole());

    proxyDefinition.getPropertyValues().add("targetBeanName", targetBeanName);
    if (proxyTargetClass) {
       targetDefinition.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
       // ScopedProxyFactoryBean's "proxyTargetClass" default is TRUE, so we don't need to set it explicitly here.
    }
    else {
       proxyDefinition.getPropertyValues().add("proxyTargetClass", Boolean.FALSE);
    }

    // Copy autowire settings from original bean definition.
    proxyDefinition.setAutowireCandidate(targetDefinition.isAutowireCandidate());
    proxyDefinition.setPrimary(targetDefinition.isPrimary());
    if (targetDefinition instanceof AbstractBeanDefinition) {
       proxyDefinition.copyQualifiersFrom((AbstractBeanDefinition) targetDefinition);
    }

    // The target bean should be ignored in favor of the scoped proxy.
    targetDefinition.setAutowireCandidate(false);
    targetDefinition.setPrimary(false);

    // Register the target bean as separate bean in the factory.
    //这里注册了targetBean，bean名字是scopedTarget.+原名字
    registry.registerBeanDefinition(targetBeanName, targetDefinition);

    // Return the scoped proxy definition as primary bean definition
    // (potentially an inner bean).
    //这里返回代理bean的信息，但是bean名字还是原来的bean
    return new BeanDefinitionHolder(proxyDefinition, originalBeanName, definition.getAliases());
}
```

## 实例化scopeBean

由于scopeBean都不是单例，所以不是在熟知的refresh遍历beanNames进行实例化的。

而是在finishRefresh方法中，对于ContextRefreshedEvent事件进行处理时，在RefresScope中进行的getBean

```java
@Override
public void onApplicationEvent(ContextRefreshedEvent event) {
    start(event);
}

public void start(ContextRefreshedEvent event) {
    if (event.getApplicationContext() == this.context && this.eager
          && this.registry != null) {
       eagerlyInitialize();
    }
}

private void eagerlyInitialize() {
    for (String name : this.context.getBeanDefinitionNames()) {
       BeanDefinition definition = this.registry.getBeanDefinition(name);
       if (this.getName().equals(definition.getScope())
             && !definition.isLazyInit()) {
          Object bean = this.context.getBean(name);
          if (bean != null) {
             bean.getClass();
          }
       }
    }
}
```

在doGetBean时，对于Scope类型，是要走特殊的逻辑的，会从scopes中获取对应的scope。

```java
if (mbd.isSingleton()) {...}
else if (mbd.isPrototype()) {...}
else {
    String scopeName = mbd.getScope();
    final Scope scope = this.scopes.get(scopeName);
    if (scope == null) {
       throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
    }
    try {
    //会调到RefreshScope的get方法，将其添加到cache中
       Object scopedInstance = scope.get(beanName, () -> {
          beforePrototypeCreation(beanName);
          try {
             return createBean(beanName, mbd, args);
          }
          finally {
             afterPrototypeCreation(beanName);
          }
       });
       bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
    }
    catch (IllegalStateException ex) {
       throw new BeanCreationException(beanName,
             "Scope '" + scopeName + "' is not active for the current thread; consider " +
             "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
             ex);
    }
}
```

### RefreshScope

这里需要先了解下scopes里面的内容从何而来，以及认识一下ScopeRefresh类。

RefreshScope是实现了BeanFactoryPostProcessor接口的类，因此在执行到postProcessBeanFactory时，就会添加到scopes中去。

```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
       throws BeansException {
    this.beanFactory = beanFactory;
    beanFactory.registerScope(this.name, this);
    setSerializationId(beanFactory);
}

public void registerScope(String scopeName, Scope scope) {
    Assert.notNull(scopeName, "Scope identifier must not be null");
    Assert.notNull(scope, "Scope must not be null");
    if (SCOPE_SINGLETON.equals(scopeName) || SCOPE_PROTOTYPE.equals(scopeName)) {
       throw new IllegalArgumentException("Cannot replace existing scopes 'singleton' and 'prototype'");
    }
    //scopeName为refresh，value是this
    Scope previous = this.scopes.put(scopeName, scope);
    if (previous != null && previous != scope) {
       if (logger.isDebugEnabled()) {
          logger.debug("Replacing scope '" + scopeName + "' from [" + previous + "] to [" + scope + "]");
       }
    }
    else {
       if (logger.isTraceEnabled()) {
          logger.trace("Registering scope '" + scopeName + "' with implementation [" + scope + "]");
       }
    }
}
```

同时RefreshScope继承了GenericScope，其大部分核心的逻辑都在GenericScope，GenericScope持有一个cache的对象，就是当前scope对应的所有的bean，cache的类型是ConcurrentHashMap。

而在doCreateBean方法，scope.get会真正调到这里。

```java
@Override
public Object get(String name, ObjectFactory<?> objectFactory) {
    BeanLifecycleWrapper value = this.cache.put(name,
          new BeanLifecycleWrapper(name, objectFactory));
    this.locks.putIfAbsent(name, new ReentrantReadWriteLock());
    try {
    //实际是调用objectFactory.getObject();
       return value.getBean();
    }
    catch (RuntimeException e) {
       this.errors.put(name, e);
       throw e;
    }
}


public Object getBean() {
    if (this.bean == null) {
       synchronized (this.name) {
          if (this.bean == null) {
             this.bean = this.objectFactory.getObject();
          }
       }
    }
    return this.bean;
}
```

## 刷新ScopeBean

在RefreshScope类中，表示刷新的方法有两个，分别是刷新单个bean或者刷新全部的bean，本质是调用的cache的remove和clear。

当在缓存中删除之后，下一次再调用的时候，就会重新调用getBean来创造新的bean，这也是scope为refresh的本质。

```Java
public boolean refresh(String name) {
    if (!name.startsWith(SCOPED_TARGET_PREFIX)) {
       // User wants to refresh the bean with this name but that isn't the one in the
       // cache...
       name = SCOPED_TARGET_PREFIX + name;
    }
    // Ensure lifecycle is finished if bean was disposable
    if (super.destroy(name)) {
       this.context.publishEvent(new RefreshScopeRefreshedEvent(name));
       return true;
    }
    return false;
}

@ManagedOperation(description = "Dispose of the current instance of all beans "
       + "in this scope and force a refresh on next method execution.")
public void refreshAll() {
    super.destroy();
    this.context.publishEvent(new RefreshScopeRefreshedEvent());
}

protected boolean destroy(String name) {
//移除指定bean
    BeanLifecycleWrapper wrapper = this.cache.remove(name);
    if (wrapper != null) {
       Lock lock = this.locks.get(wrapper.getName()).writeLock();
       lock.lock();
       try {
          wrapper.destroy();
       }
       finally {
          lock.unlock();
       }
       this.errors.remove(name);
       return true;
    }
    return false;
}
public void destroy() {
    List<Throwable> errors = new ArrayList<Throwable>();
    //移除所有的bean
    Collection<BeanLifecycleWrapper> wrappers = this.cache.clear();
    for (BeanLifecycleWrapper wrapper : wrappers) {
       try {
          Lock lock = this.locks.get(wrapper.getName()).writeLock();
          lock.lock();
          try {
             wrapper.destroy();
          }
          finally {
             lock.unlock();
          }
       }
       catch (RuntimeException e) {
          errors.add(e);
       }
    }
    if (!errors.isEmpty()) {
       throw wrapIfNecessary(errors.get(0));
    }
    this.errors.clear();
}
```
