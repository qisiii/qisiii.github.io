+++
title = "@Async源码分析"
description = ""
date = 2023-12-26
image = ""
draft = false
slug = "Async"
tags = ['Spring']
categories = ['tech']
+++

## @EnableAsync

```Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AsyncConfigurationSelector.class)
public @interface EnableAsync {
    //这里应该是可以自定义注解
   Class<? extends Annotation> annotation() default Annotation.class;

   boolean proxyTargetClass() default false;

   AdviceMode mode() default AdviceMode.PROXY;

   int order() default Ordered.LOWEST_PRECEDENCE;
```

由[Spring注册BeanDefinition的方式]({{<ref "Spring注册BeanDefinition的方式">}}) 可知，@Enable的@Import注解中的AsyncConfigurationSelector是为了注册指定的bean，加上default AdviceMode.*PROXY，结合下面代码，注册的是ProxyAsyncConfiguration，而里面最重要的是定义了AsyncAnnotationBeanPostProcessor这个Bean*

```Java
@Override
@Nullable
public String[] selectImports(AdviceMode adviceMode) {
   switch (adviceMode) {
      case PROXY:
         return new String[] {ProxyAsyncConfiguration.class.getName()};
      case ASPECTJ:
         return new String[] {ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME};
      default:
         return null;
   }
}

public class ProxyAsyncConfiguration extends AbstractAsyncConfiguration {
    @Bean（name="internalAsyncAnnotationProcessor")
    public AsyncAnnotationBeanPostProcessor asyncAdvisor() {
       Assert.notNull(this.enableAsync, "@EnableAsync annotation metadata was not injected");
       AsyncAnnotationBeanPostProcessor bpp = new AsyncAnnotationBeanPostProcessor();
       //这两个配置默认是空的，想要自定义需要实现AsyncConfigurer接口
       bpp.configure(this.executor, this.exceptionHandler);
       //这意味着可以自定义注解
       Class<? extends Annotation> customAsyncAnnotation = this.enableAsync.getClass("annotation");
       if (customAsyncAnnotation != AnnotationUtils.getDefaultValue(EnableAsync.class, "annotation")) {
          bpp.setAsyncAnnotationType(customAsyncAnnotation);
       }
       bpp.setProxyTargetClass(this.enableAsync.getBoolean("proxyTargetClass"));
       bpp.setOrder(this.enableAsync.<Integer>getNumber("order"));
       return bpp;
    }
```

AsyncAnnotationBeanPostProcessor实现了BeanFactoryAware接口，因此在initializeBean之前就先调用了aware接口的setBeanFactory，最终是制造了一个AsyncAnnotationAdvisor作为后处理器的一个属性

```Java
#AsyncAnnotationBeanPostProcessor
@Override
public void setBeanFactory(BeanFactory beanFactory) {
   super.setBeanFactory(beanFactory);

   AsyncAnnotationAdvisor advisor = new AsyncAnnotationAdvisor(this.executor, this.exceptionHandler);
   if (this.asyncAnnotationType != null) {
      advisor.setAsyncAnnotationType(this.asyncAnnotationType);
   }
   advisor.setBeanFactory(beanFactory);
   this.advisor = advisor;
}
```

```Java
#AsyncAnnotationAdvisor
public AsyncAnnotationAdvisor(
      @Nullable Supplier<Executor> executor, @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {
   ////默认应该是@Async和@Asynchronous注解，但可以调用setAsyncAnnotationType来覆盖
   Set<Class<? extends Annotation>> asyncAnnotationTypes = new LinkedHashSet<>(2);
   asyncAnnotationTypes.add(Async.class);
   try {
      asyncAnnotationTypes.add((Class<? extends Annotation>)
            ClassUtils.forName("javax.ejb.Asynchronous", AsyncAnnotationAdvisor.class.getClassLoader()));
   }
   catch (ClassNotFoundException ex) {
      // If EJB 3.1 API not present, simply ignore.
   }
   //实例化了一个拦截器---AnnotationAsyncExecutionInterceptor
   this.advice = buildAdvice(executor, exceptionHandler);
   //构造了两个切点，一个是类维度，一个是方法维度的
   this.pointcut = buildPointcut(asyncAnnotationTypes);
}
```

所以，当实例化对象执行到postProcessAfterInitialization的时候，就会执行AsyncAnnotationBeanPostProcessor的postProcessAfterInitialization

```Java
public Object postProcessAfterInitialization(Object bean, String beanName) {
   if (this.advisor == null || bean instanceof AopInfrastructureBean) {
      // Ignore AOP infrastructure such as scoped proxies.
      return bean;
   }

   if (bean instanceof Advised) {
      Advised advised = (Advised) bean;
      if (!advised.isFrozen() && isEligible(AopUtils.getTargetClass(bean))) {
         // Add our local Advisor to the existing proxy's Advisor chain...
         if (this.beforeExistingAdvisors) {
            advised.addAdvisor(0, this.advisor);
         }
         else {
            advised.addAdvisor(this.advisor);
         }
         return bean;
      }
   }
   //如果是aop代理的
   //通过getPointcut获取是否匹配
   if (isEligible(bean, beanName)) {
      ProxyFactory proxyFactory = prepareProxyFactory(bean, beanName);
      if (!proxyFactory.isProxyTargetClass()) {
         evaluateProxyInterfaces(bean.getClass(), proxyFactory);
      }
      proxyFactory.addAdvisor(this.advisor);
      customizeProxyFactory(proxyFactory);

      // Use original ClassLoader if bean class not locally loaded in overriding class loader
      ClassLoader classLoader = getProxyClassLoader();
      if (classLoader instanceof SmartClassLoader && classLoader != bean.getClass().getClassLoader()) {
         classLoader = ((SmartClassLoader) classLoader).getOriginalClassLoader();
      }
      //匹配的话会在这里对对象进行加强，通过调用getAdvice
      return proxyFactory.getProxy(classLoader);
   }

   // No proxy needed.
   return bean;
}


protected boolean isEligible(Class<?> targetClass) {
   Boolean eligible = this.eligibleBeans.get(targetClass);
   if (eligible != null) {
      return eligible;
   }
   if (this.advisor == null) {
      return false;
   }
   //底层是匹配getPointcut
   eligible = AopUtils.canApply(this.advisor, targetClass);
   this.eligibleBeans.put(targetClass, eligible);
   return eligible;
}
```

## @Async

从上面知道，最终起效的是一个拦截器AnnotationAsyncExecutionInterceptor，这样，当我们执行到被@Async注释的方法时，其实是执行的拦截器的invoke方法

```Java
protected Advice buildAdvice(
      @Nullable Supplier<Executor> executor, @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {

   AnnotationAsyncExecutionInterceptor interceptor = new AnnotationAsyncExecutionInterceptor(null);
   interceptor.configure(executor, exceptionHandler);
   return interceptor;
}
public void configure(@Nullable Supplier<Executor> defaultExecutor,
      @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {
    //没设置也有兜底的线程池,在容器查找TaskExecutor.class或者名为taskExecutor，类型是Executor.class的bean
   this.defaultExecutor = new SingletonSupplier<>(defaultExecutor, () -> getDefaultExecutor(this.beanFactory));
   this.exceptionHandler = new SingletonSupplier<>(exceptionHandler, SimpleAsyncUncaughtExceptionHandler::new);
}


public Object invoke(final MethodInvocation invocation) throws Throwable {
   Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
   Method specificMethod = ClassUtils.getMostSpecificMethod(invocation.getMethod(), targetClass);
   final Method userDeclaredMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
//获取线程池，@Async指定的线程池bean名称>AsyncConfigurer指定的线程池>beanFactory的线程池
   AsyncTaskExecutor executor = determineAsyncExecutor(userDeclaredMethod);
   if (executor == null) {
      throw new IllegalStateException(
            "No executor specified and no default executor set on AsyncExecutionInterceptor either");
   }

   Callable<Object> task = () -> {
      try {
         Object result = invocation.proceed();
         if (result instanceof Future) {
            return ((Future<?>) result).get();
         }
      }
      return null;
   };

   return doSubmit(task, executor, invocation.getMethod().getReturnType());
}
```

## @Async循环依赖问题

以下面的代码为demo启动时，

```Java
@Service
public class Bean1 {
    @Autowired
    private Bean2 bean2;
    @Async
    public void test(){
        System.out.println("Bean1");
    }

}
@Service
public class Bean2 {
    @Autowired
    private Bean1 bean1;
    public void test(){
        System.out.println("Bean2");
    }
}
//启动报错
Error creating bean with name 'bean1': Bean with name 'bean1' has been injected into other beans [bean2] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.
```

报错的地方是初始化完之后，再校验暴漏的bean和本来的bean是否一样时报的错

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=Yjg5OGRkZmVmYWRiZTE0ODJkZGM2N2U4ZWNjMDM3MDFfeXNEaVJlektPOWNrQ25VTktlOWx2WndzS2R2ekZic0RfVG9rZW46RjE1bmJuYWhvb3V0VVB4WDE3cGM3OGFzbnJmXzE3MjMzNjc0MTk6MTcyMzM3MTAxOV9WNA)

本来的bean通过getEarlyBeanReference获取到，是没有代理的bean

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MDRlYTVlODAwYTg5ZGNkNTVjMjI1ODgzZTQ1Nzc5NjdfN1h3MXIwQ0szZXA0WGpCMkVjWmszMFc0bmExTEwzZTBfVG9rZW46SmNLY2JTMzByb2xzdjd4dU54UGNnajY3bkZnXzE3MjMzNjc0MTk6MTcyMzM3MTAxOV9WNA)

但是由于使用了@Async，导致在AsyncAnnotationBeanPostProcessor.postProcessAfterInitialization方法中对bean进行了加强

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjQ4YjY0NTdmNWRiZWM2Mjk1OGM5OWE2NzkzNWZkYmZfVDZwU1pCVmhJTVBvdlZIQ0lJZVlrbTc4NkRRbWROUjFfVG9rZW46SEhPUmJGYW5Pb01TQnF4N3ROSWMwTHNobmxyXzE3MjMzNjc0MTk6MTcyMzM3MTAxOV9WNA)

---

但是，根据上面的源码研究，async其实也是属于使用了spring的aop机制，自定义了advisor，那为什么没有像aop一样使用二级缓存呢？

最核心的问题就在于，getAdvicesAndAdvisorsForBean并没有找到自定义的advisor，原因是AsyncAnnotationAdvisor没有注册为bean,AsyncAnnotationAdvisor只是作为asyncAnnotationBeanPostProcessor的一个属性

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MDAxNmIwN2Q5N2JlZGUwNTA1NDJiNmFlOWVjNjlmNzRfMElGMXlLZ2kySVJyTkFEZk9KTFQxaW5Nbk4xa3ZDck1fVG9rZW46RmsxMGJCcHJibzdjQ0t4OUQ2R2NxQW44bjFkXzE3MjMzNjc0MTk6MTcyMzM3MTAxOV9WNA)

解决方案

使用@lazy注解在任意一个bean上

原理是在递归处理的时候，生成一个虚假的代理，真正使用的时候再生成真的

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=OTJlZGNhNDc5MTUyMjU0Mjg5NGI2ZDZmMTI0MzZlNWJfQXdSZGlEc3owN081Z1FBYU9TZ1A3V1BvYUJxNzFiQ3JfVG9rZW46T2xJcmJ1Skczb0tXb1R4QzZ6dGNvRmF6bjBmXzE3MjMzNjc0MTk6MTcyMzM3MTAxOV9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=YmEwMzk0YzMyNDA2MWMxZGYzNjBiNWE1ZjVlNGQzMWRfTjhKaG51STB2eER1dUc3VmRqaUlJdXhPY3E5dTFLbFBfVG9rZW46UUJZTGI2elFLbzZ6YmJ4YzF1ZWN1YVB5bmZoXzE3MjMzNjc0MTk6MTcyMzM3MTAxOV9WNA)

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MDExYzMzYzIzMzk4YjY0ZGYxMmY1MThlNjY2MTA0NzZfVWFONzZGSU5zb3A1TnlFd3NOQWNYcGVsSmcyZDNDeHRfVG9rZW46SFN2VmJlcGlUb0pCeGp4VFp6TWNKcEVQbk9nXzE3MjMzNjc0MTk6MTcyMzM3MTAxOV9WNA)

---

补充，必须得是使用了@Async的提前暴漏了，才会抛这个异常；

先加载的如果有@Async，那么后加载的bean注入的有@Async的bean就是一个未加强的，导致后加载的这个bean不正确；

先加载的如果没有@Async，那么后加载的bean注入的本身就是原bean，所以后加载的没有问题，然后后加载的流程走完之后，由于其带有@async，所以是加强过的，而先加载的bean依赖的是加强过的bean，所以这个时候两个bean都正确

## 参考文档:

[Spring Boot 2.0.0 升级到2.4.1 循环依赖问题的解决](https://zhuanlan.zhihu.com/p/366440373)
