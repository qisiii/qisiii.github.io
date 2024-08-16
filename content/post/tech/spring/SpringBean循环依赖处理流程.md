+++
title = "SpringBean循环依赖处理流程"
description = ""
date = 2024-06-20
image = ""
draft = false
slug = "cycle_bean"
tags = ['Spring']
categories = ['tech']
+++

```java
@Service
public class Bean1 {
    @Autowired
    private Bean2 bean2;
//    @Async
    public void test(){
        System.out.println("Bean1");
    }

}
@Service
public class Bean2 {
    @Autowired
    private Bean1 bean1;
//    @Async
    public void test(){
        System.out.println("Bean2");
    }
}
```

![](http://picgo.qisiii.asia/post/11-17-02-57-image.png)

在doGetBean的第一个节点上，会调用到getSingleon这个方法，正常来说，Bean1和Bean2首次获取都会是空，只有在实例化后&属性注入前，才会将当前状态的Bean暴露到singleton工厂来获取。

```java
分为三级缓存
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // Quick check for existing instance without full singleton lock
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
       singletonObject = this.earlySingletonObjects.get(beanName);
       if (singletonObject == null && allowEarlyReference) {
          synchronized (this.singletonObjects) {
             // Consistent creation of early reference within full singleton lock
             singletonObject = this.singletonObjects.get(beanName);
             if (singletonObject == null) {
                singletonObject = this.earlySingletonObjects.get(beanName);
                if (singletonObject == null) {
                   ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                   if (singletonFactory != null) {
                      singletonObject = singletonFactory.getObject();
                      this.earlySingletonObjects.put(beanName, singletonObject);
                      this.singletonFactories.remove(beanName);
                   }
                }
             }
          }
       }
    }
    return singletonObject;
}
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
       for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {
          exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
       }
    }
    return exposedObject;
}
```

所以是在Bean2注入属性Bean1的时候，可以获取到Bean1的暴露对象，同时放到了二级缓存中。

为什么要有三级缓存，二级不够吗？

三级缓存主要是为了解决aop的，分两点，一点是保证所有的aop只代理一次，之后的bean则从二级缓存获取。

第二点这是，aop代理正常情况下是在bean初始化之后才进行的aop代理，只是在循环依赖的时候提前进行了代理。当一个bean本身不知道是否存在循环依赖的时候，不应该提前就进行代理，所以这里不管怎样都要放入二级缓存中去。

有没有循环依赖没有解决的场景？

有，对于都是构造器注入的bean，由于spring处理的循环依赖是在实例化后进行暴露的bean，而在构造器阶段，双方都需要对方的实力作为参数，就会报循环依赖。

还有一种情况，则是在bean已经被循环依赖处理后，自己通过其他方式返回了一个不是原来bean的bean，比如@Async的代理对象，通过beanpostprocessor进行了增强，导致Bean2所持有的bean1不对了，这个时候也算错误。
