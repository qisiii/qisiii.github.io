+++
title = "Aop执行逻辑"
description = ""
date = 2024-04-27
image = ""
draft = false
slug = "Aop"
tags = ['Spring']
categories = ['tech']
+++

## AopAutoConfiguration

在spring-boot-autoconfiguration的spring.factories中指定了org.springframework.boot.autoconfigure.aop.AopAutoConfiguration这个bean，因此会进行执行

```java
@Configuration(proxyBeanMethods = false)
//默认开启，如果不配或者为true都是开启，为false则是关闭
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {
   //如果项目中引入aspectjweaver包，则实例化AspectJAutoProxyingConfiguration否则实例化ClassProxyingConfiguration
   @Configuration(proxyBeanMethods = false)
   @ConditionalOnClass(Advice.class)
   static class AspectJAutoProxyingConfiguration {

      @Configuration(proxyBeanMethods = false)
      @EnableAspectJAutoProxy(proxyTargetClass = false)
      @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false")
      static class JdkDynamicAutoProxyConfiguration {

      }
      //默认启用的是cglib代理，通过@EnableAspectJAutoProxy引入其他类（EnableAspectJAutoProxy）的配置
      @Configuration(proxyBeanMethods = false)
      @EnableAspectJAutoProxy(proxyTargetClass = true)
      @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true",
            matchIfMissing = true)
      static class CglibAutoProxyConfiguration {

      }

   }

   @Configuration(proxyBeanMethods = false)
   @ConditionalOnMissingClass("org.aspectj.weaver.Advice")
   @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true",
         matchIfMissing = true)
   static class ClassProxyingConfiguration {

      @Bean
      static BeanFactoryPostProcessor forceAutoProxyCreatorToUseClassProxying() {
         return (beanFactory) -> {
            if (beanFactory instanceof BeanDefinitionRegistry) {
               BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
               AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
               //默认也是cglib
               AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
            }
         };
      }

   }

}
```

AspectJAutoProxyingConfiguration和ClassProxyingConfiguration的区别在于

ClassProxyingConfiguration想要注册的是InfrastructureAdvisorAutoProxyCreator

AspectJAutoProxyingConfiguration注册的则是AnnotationAwareAspectJAutoProxyCreator

这里是不同级别的creator，总共存在三个，如果存在低级则替换成高级

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=YjFhM2NkMmE0MTNiZDFkOTAzOGFhNzNlZTZjZGY3ZDVfTXd1NmJCWWRtVThTRGlpVkZDNDhyWTVhYjhQaUZmaDZfVG9rZW46TElLb2JMaHNHb1RMMjh4bzk0amNxZ3JUbkxoXzE3MjMzNjc4NDI6MTcyMzM3MTQ0Ml9WNA)

```java
private static BeanDefinition registerOrEscalateApcAsRequired(
      Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");

   if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
      BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
      if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
         int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
         int requiredPriority = findPriorityForClass(cls);
         //如果当前等级小于需要的等级，则改为高等级
         if (currentPriority < requiredPriority) {
            apcDefinition.setBeanClassName(cls.getName());
         }
      }
      return null;
   }

   RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
   beanDefinition.setSource(source);
   //这个顺序有啥用？
   beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
   beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
   registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
   return beanDefinition;
}
```

这里先不熟悉三个creator的区别，有时间了搞，由于事务这里注册的是最高级别-AnnotationAwareAspectJAutoProxyCreator，所以先分析这个类

## AnnotationAwareAspectJAutoProxyCreator

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MTM4ZjUyOGI5MTUyZTg3ODRkMDgzZGQyMzFiNTJiN2FfUTZLWTNwNTlReEFiMjVuT0FpcWR5bFZ2eGtNZHNWSXVfVG9rZW46S3dXQWJnOWxRb1Z0bXh4UVM1bmNtcUdWbnRnXzE3MjMzNjc4ODc6MTcyMzM3MTQ4N19WNA)

观察类图

这个类实现了BeanFactoryAware接口来获取beanFacotry

实现了SmartInstantiationAwareBeanPostProcessor接口，主要重写了四个方法：predictBeanType、postProcessBeforeInstantiation、getEarlyBeanReference和postProcessAfterInstantiation，

postProcessBeforeInstantiation主要是用于自定义了TargetSource的情况，但是没见在哪用过；

getEarlyBeanReference则适用于处理循环依赖的时候，核心方法是wrapIfNecessary

postProcessAfterInstantiation则是正常的bean初始化结束后对bean进行aop增强的逻辑

```java
@Override
public Object getEarlyBeanReference(Object bean, String beanName) {
   Object cacheKey = getCacheKey(bean.getClass(), beanName);
   this.earlyProxyReferences.put(cacheKey, bean);
   return wrapIfNecessary(bean, beanName, cacheKey);
}
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      if (this.earlyProxyReferences.remove(cacheKey) != bean) {
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}
```

### 增强流程

![](http://picgo.qisiii.asia/post/11-17-20-10-image.png)

### wrapIfNecessary

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
   if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
      return bean;
   }
   //已经存在表示这个bean处理过了
   if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
      return bean;
   }
   //Advice、Pointcut、Advisor、AopInfrastructureBean为基础架构类，永远不可被代理
   //Q&A 2024/4/27
   // Q:shouldSkip里逻辑不懂,而且isInfrastructureClass排除了Advisor.class.isAssignableFrom(beanClass)，但实际上查询Advicors的时候又找的这个类型的bean，这怎么搞
   // A:

   if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return bean;
   }

   // Create proxy if we have advice.
   // async最大的问题就是这里没有扫描到
   //获取所有匹配的Advisor,是排好序的
   Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
   if (specificInterceptors != DO_NOT_PROXY) {
      this.advisedBeans.put(cacheKey, Boolean.TRUE);
      //进行增强
      Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   this.advisedBeans.put(cacheKey, Boolean.FALSE);
   return bean;
}

protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
   //先查找所有的候选Advisor
   List<Advisor> candidateAdvisors = findCandidateAdvisors();
   //进行匹配
   List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
   //追加了一个ExposeInvocationInterceptor.ADVISOR
   extendAdvisors(eligibleAdvisors);
   if (!eligibleAdvisors.isEmpty()) {
      //进行排序
      //默认是AnnotationAwareOrderComparator.sort(advisors)；但是AspectJAwareAdvisorAutoProxyCreator重写了排序逻辑
      eligibleAdvisors = sortAdvisors(eligibleAdvisors);
   }
   return eligibleAdvisors;
}
```

### findCandidateAdvisors

findCandidateAdvisors有两种，一种是继承了Advisor接口的bean（通过beanFactory直接查询），一种是标有@Aspect的（需要扫描所有的bean，然后反射获取切面所有的方法，每个方法单独封装为一个Advisor

```java
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
   Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
   String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
   validate(aspectClass);

   // We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
   // so that it will only instantiate once.
   MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
         new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

   List<Advisor> advisors = new ArrayList<>();
   //获取所有类的方法并排序，按照注解排序Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class)->按照方法名排序
   for (Method method : getAdvisorMethods(aspectClass)) {
      //获取切面-将pointCut和Advice组装起来
      Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, 0, aspectName);
      if (advisor != null) {
         advisors.add(advisor);
      }
   }
   //这个是introduction机制，需要使用@DeclareParents注解
    for (Field field : aspectClass.getDeclaredFields()) {
       Advisor advisor = getDeclareParentsAdvisor(field);
       if (advisor != null) {
          advisors.add(advisor);
       }
    }
   ...
 }


 private List<Method> getAdvisorMethods(Class<?> aspectClass) {
   List<Method> methods = new ArrayList<>();
   ReflectionUtils.doWithMethods(aspectClass, methods::add, adviceMethodFilter);
   if (methods.size() > 1) {
      methods.sort(adviceMethodComparator);
   }
   return methods;
}

//先按照注解排序，再按照方法名称排序
Comparator<Method> adviceKindComparator = new ConvertingComparator<>(
      new InstanceComparator<>(
            Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class),
      (Converter<Method, Annotation>) method -> {
         AspectJAnnotation<?> ann = AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(method);
         return (ann != null ? ann.getAnnotation() : null);
      });
Comparator<Method> methodNameComparator = new ConvertingComparator<>(Method::getName);
adviceMethodComparator = adviceKindComparator.thenComparing(methodNameComparator);
```

#### 获取Advisor

```java
 public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
      int declarationOrderInAspect, String aspectName) {

   validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
   //切点其实封装的就是表达式和切面类
   AspectJExpressionPointcut expressionPointcut = getPointcut(
         candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
   if (expressionPointcut == null) {
      return null;
   }
   //封装切点和Advice，advice其实就是那几种注解@Before、@After之类的
   return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
         this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}
```

#### 获取切点

```java
private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
   AspectJAnnotation<?> aspectJAnnotation =
         AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
   if (aspectJAnnotation == null) {
      return null;
   }

   AspectJExpressionPointcut ajexp =
         new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
   //将方法上的注解的value值放到expression属性
   ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
   if (this.beanFactory != null) {
      ajexp.setBeanFactory(this.beanFactory);
   }
   return ajexp;
}
```

#### 获取Advice

```java
public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
      MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
....
//主要就这几种场景的advice
switch (aspectJAnnotation.getAnnotationType()) {
   case AtPointcut:
      if (logger.isDebugEnabled()) {
         logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
      }
      return null;
   case AtAround:
      springAdvice = new AspectJAroundAdvice(
            candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
      break;
   case AtBefore:
      springAdvice = new AspectJMethodBeforeAdvice(
            candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
      break;
   case AtAfter:
      springAdvice = new AspectJAfterAdvice(
            candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
      break;
   case AtAfterReturning:
      springAdvice = new AspectJAfterReturningAdvice(
            candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
      AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
      if (StringUtils.hasText(afterReturningAnnotation.returning())) {
         springAdvice.setReturningName(afterReturningAnnotation.returning());
      }
      break;
   case AtAfterThrowing:
      springAdvice = new AspectJAfterThrowingAdvice(
            candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
      AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
      if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
         springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
      }
      break;
   default:
      throw new UnsupportedOperationException(
            "Unsupported advice type on method: " + candidateAdviceMethod);
}
```

### Advisor排序逻辑

默认是AnnotationAwareOrderComparator.sort(advisors)；但是AspectJAwareAdvisorAutoProxyCreator重写了排序逻辑

#### AnnotationAwareOrderComparator

这个类实现了OrderComparator，正常对于OrderComparator来说，其compare调用doCompare，里面的顺序以getOrder方法为主，getOrder调用的又是findOrder，及根据接口Ordered.getOrder获取顺序值

```java
private int doCompare(@Nullable Object o1, @Nullable Object o2, @Nullable OrderSourceProvider sourceProvider) {
   boolean p1 = (o1 instanceof PriorityOrdered);
   boolean p2 = (o2 instanceof PriorityOrdered);
   if (p1 && !p2) {
      return -1;
   }
   else if (p2 && !p1) {
      return 1;
   }

   int i1 = getOrder(o1, sourceProvider);
   int i2 = getOrder(o2, sourceProvider);
   return Integer.compare(i1, i2);
}
protected Integer findOrder(Object obj) {
   return (obj instanceof Ordered ? ((Ordered) obj).getOrder() : null);
}
```

而AnnotationAwareOrderComparator则进行了扩展，对于使用了@Order注解的也可以做排序，主要是重写了findOrder方法

```java
protected Integer findOrder(Object obj) {
   Integer order = super.findOrder(obj);
   if (order != null) {
      return order;
   }
   return findOrderFromAnnotation(obj);
}

@Nullable
private Integer findOrderFromAnnotation(Object obj) {
   AnnotatedElement element = (obj instanceof AnnotatedElement ? (AnnotatedElement) obj : obj.getClass());
   MergedAnnotations annotations = MergedAnnotations.from(element, SearchStrategy.TYPE_HIERARCHY);
   Integer order = OrderUtils.getOrderFromAnnotations(element, annotations);
   if (order == null && obj instanceof DecoratingProxy) {
      return findOrderFromAnnotation(((DecoratingProxy) obj).getDecoratedClass());
   }
   return order;
}

org.springframework.core.annotation.OrderUtils#findOrder
private static Integer findOrder(MergedAnnotations annotations) {
   MergedAnnotation<Order> orderAnnotation = annotations.get(Order.class);
   if (orderAnnotation.isPresent()) {
      return orderAnnotation.getInt(MergedAnnotation.VALUE);
   }
   MergedAnnotation<?> priorityAnnotation = annotations.get(JAVAX_PRIORITY_ANNOTATION);
   if (priorityAnnotation.isPresent()) {
      return priorityAnnotation.getInt(MergedAnnotation.VALUE);
   }
   return null;
}
```

#### AspectJAwareAdvisorAutoProxyCreator.sort

```java
protected List<Advisor> sortAdvisors(List<Advisor> advisors) {
   List<PartiallyComparableAdvisorHolder> partiallyComparableAdvisors = new ArrayList<>(advisors.size());
   for (Advisor advisor : advisors) {
      partiallyComparableAdvisors.add(
            new PartiallyComparableAdvisorHolder(advisor, DEFAULT_PRECEDENCE_COMPARATOR));
   }
   List<PartiallyComparableAdvisorHolder> sorted = PartialOrder.sort(partiallyComparableAdvisors);
   if (sorted != null) {
      List<Advisor> result = new ArrayList<>(advisors.size());
      for (PartiallyComparableAdvisorHolder pcAdvisor : sorted) {
         result.add(pcAdvisor.getAdvisor());
      }
      return result;
   }
   else {
      return super.sortAdvisors(advisors);
   }
}

private static final Comparator<Advisor> DEFAULT_PRECEDENCE_COMPARATOR = new AspectJPrecedenceComparator();
```

先梳理一下陌生的类

PartiallyComparableAdvisorHolder：持有Advisor和一个Comparator，重写了compareTo和fallbackCompareTo

```java
private static class PartiallyComparableAdvisorHolder implements PartialComparable {

   private final Advisor advisor;

   private final Comparator<Advisor> comparator;

   public PartiallyComparableAdvisorHolder(Advisor advisor, Comparator<Advisor> comparator) {
      this.advisor = advisor;
      this.comparator = comparator;
   }

   @Override
   public int compareTo(Object obj) {
      Advisor otherAdvisor = ((PartiallyComparableAdvisorHolder) obj).advisor;
      return this.comparator.compare(this.advisor, otherAdvisor);
   }

   @Override
   public int fallbackCompareTo(Object obj) {
      return 0;
   }
```

AspectJPrecedenceComparator：默认还是AnnotationAwareOrderComparator，但是对于同一个切面的advisor做了自己的处理。但是在spring 5.2.7 之前是有用的，5.2.7之后，declarationOrderInAspect全部强制改为0了

> // Prior to Spring Framework 5.2.7, advisors.size() was supplied as the declarationOrderInAspect
> 
> // to getAdvisor(...) to represent the "current position" in the declared methods list.
> 
> // However, since Java 7 the "current position" is not valid since the JDK no longer
> 
> // returns declared methods in the order in which they are declared in the source code.
> 
> // Thus, we now hard code the declarationOrderInAspect to 0 for all advice methods
> 
> // discovered via reflection in order to support reliable advice ordering across JVM launches.
> 
> // Specifically, a value of 0 aligns with the default value used in
> 
> // AspectJPrecedenceComparator.getAspectDeclarationOrder(Advisor).

```SQL
public AspectJPrecedenceComparator() {
   this.advisorComparator = AnnotationAwareOrderComparator.INSTANCE;
}
public int compare(Advisor o1, Advisor o2) {
//即先拿order比较
   int advisorPrecedence = this.advisorComparator.compare(o1, o2);
   if (advisorPrecedence == SAME_PRECEDENCE && declaredInSameAspect(o1, o2)) {
      advisorPrecedence = comparePrecedenceWithinAspect(o1, o2);
   }
   return advisorPrecedence;
}
//拿声明顺序比较
private int comparePrecedenceWithinAspect(Advisor advisor1, Advisor advisor2) {
   boolean oneOrOtherIsAfterAdvice =
         (AspectJAopUtils.isAfterAdvice(advisor1) || AspectJAopUtils.isAfterAdvice(advisor2));
   int adviceDeclarationOrderDelta = getAspectDeclarationOrder(advisor1) - getAspectDeclarationOrder(advisor2);
    //对于是否存在after，是两种逻辑，存在after的话，最后声明的有最高优先级，否则，最先声明的有最高优先级
    //Q&A 2024/4/27
    // Q: 完全不理解
    // A:
   if (oneOrOtherIsAfterAdvice) {
      // the advice declared last has higher precedence
      if (adviceDeclarationOrderDelta < 0) {
         // advice1 was declared before advice2
         // so advice1 has lower precedence
         return LOWER_PRECEDENCE;
      }
      else if (adviceDeclarationOrderDelta == 0) {
         return SAME_PRECEDENCE;
      }
      else {
         return HIGHER_PRECEDENCE;
      }
   }
   else {
      // the advice declared first has higher precedence
      if (adviceDeclarationOrderDelta < 0) {
         // advice1 was declared before advice2
         // so advice1 has higher precedence
         return HIGHER_PRECEDENCE;
      }
      else if (adviceDeclarationOrderDelta == 0) {
         return SAME_PRECEDENCE;
      }
      else {
         return LOWER_PRECEDENCE;
      }
   }
}
```

此外，这里使用的不是TimSort，而是PartialOrder(离散里的偏序）
