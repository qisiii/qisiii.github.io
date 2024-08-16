+++
title = "Spring注册BeanDefinition的方式"
description = ""
date = 2023-12-25
image = ""
draft = false
slug = "BeanDefinitionRegistion"
tags = ['Spring']
categories = ['tech']
+++

通过在org.springframework.beans.factory.support.DefaultListableBeanFactory#registerBeanDefinition这个方法打断点可以知晓所有的注册途径

## xml注册bean

```Java
ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("/note/simpleApplication.xml");
assertThat(ctx.containsBean("noteFirst")).isTrue();
assertThat(ctx.containsBean("noteBean")).isTrue();ctx.close();
```

![](http://picgo.qisiii.asia/post/w39dhfwj7v37x15wz_psyqmc0000gn/T/202408156df18ebc5eba0a98d330d5f2bb3556e9.jpg)

需要在配置文件中指定bean或者指定扫描的包

```Java
<bean id="noteFirst" name="noteFirst"
     class="org.springframework.note.NoteFirst"/>
<context:component-scan base-package="org.springframework.note"></context:component-scan>
```

在refresh方法的obtainFreshBeanFactory->refreshBeanFactory->loadBeanDefinitions处理beanDefinition

对于bean的处理，最终是在org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#processBeanDefinition这个方法处理的

对于context:component-scan，则会走到org.springframework.context.annotation.ComponentScanBeanDefinitionParser#parse，通过scan扫描到@Component、@ManagedBean、@Named的bean，通过org.springframework.beans.factory.support.BeanDefinitionReaderUtils#registerBeanDefinition方法注册

## 注解注册

### @Component

所有的@Component标注的类需要被扫描到才可以，不然不会加载，因此，也可以确定@Component标注的类是通过doScan来处理的

对于这种的bean是需要通过beanDefinition的后处理器来加载bean，也就是refresh的invokeBeanFactoryPostProcessors方法

![](http://picgo.qisiii.asia/post/w39dhfwj7v37x15wz_psyqmc0000gn/T/bb61d2d5d8f0e2abc92e789908810daf.jpg)

![](http://picgo.qisiii.asia/post/w39dhfwj7v37x15wz_psyqmc0000gn/T/616f982c76f4e936b2863824969fe82d.jpg)

这里的ConfigurationClassPostProcessor是通过内部注册到beanFactory的

会调用ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry->org.springframework.context.annotation.ConfigurationClassPostProcessor#processConfigBeanDefinitions

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NGI1N2VlOWQwMTJkNGJkMjIwZGFmNzc3NzUxMzc3YjJfRVE1OVMzOWZ6b25mZFpMZHJuSHh4akZsS0laVmdoazBfVG9rZW46QjJNUmIxdWpnb1JWOWF4RGVZVWNhZ2R1blZjXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

如图，postProcessBeanDefinitionRegistry方法中处理ComponentBean的地方是在ConfigurationClassParser.parse，其最终执行的是**ConfigurationClassParser#doProcessConfigurationClass**

但candidates必须有值，也就是说为了扫描别的bean，至少有一个bean已经注册且已经指定了ComponentScan的范围，一般来说如果是AnnotationConfigApplicationContext，那就是启动时指定的类

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MTdmMjIwODVmN2RlODdjYjM2NzM3ZWJiYjM2YTE2ODhfZWQ5Tjhod2R5Slo3amhIbFBOaUtCWEFtNDBCWlhtSnpfVG9rZW46VVVEM2JlYkpob0w1eWl4Nm1tNmN6UnQybmxoXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

而如果是springboot，那就是@SpringBootApplication标注的类

最终会在下面这段代码扫描所有的Component

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=YzI3N2Y0MTM0N2MxY2M4ODBiODc5MDVmYmE1MGRlMzhfYWl3eWVGQWJ0MTlHUFVQUlhVNFJCSk45SDdjZE9tOTNfVG9rZW46UndGeGJZdzhJb0xOQUZ4eDBQTGN2UldlbmVnXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

### @Bean

@Bean是在加载@Component的bean之后进行处理的，虽然在bean定义阶段是一样的，但是@Configuration类中的@Bean是被动态代理了的

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MTJmZTJiOGQzMzZjMzczZWM1MjQwMjg1M2FlZmEyM2JfSU9pTEJIaExIczVBYTgzN1k4WEF4SlNrMFdvQnBocFdfVG9rZW46RDZCNGJzZkJ4b2ZBVDF4YmVubmNQMGNHbnBlXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

当处理完所有的Component类之后，就可以通过ConfigClass处理里面的@bean所生产的bean，所以还是在ConfigurationClassPostProcessor#processConfigBeanDefinitions方法内

最终走到的也是loadBeanDefinitionsForBeanMethod，然后调用了registerBeanDefinition

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NGRmYWViZThhNGM5MWUyN2EyNDI5OTg3YjdjYWY0ZGJfRkJodElhbmZWSVhSY1haSXlxMDRIdUw2SllUR1VacXBfVG9rZW46SlJiVmJ2TUZjb3d6TXd4b25xTWNUSDhzbk5iXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

## @Import注册

分为两步

### processImports

第一步是将@Import内的类注册为ConfigClass，也是在**ConfigurationClassParser#doProcessConfigurationClass** 方法中，处理完ComponentScan就是@Import了

同理，也必须是在某个bean类上的Import才有作用

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=YjQxZGI2ODhjZjlkNGI3YjEyOGExNzY1MzE4ODMzMzJfVG44SEs4NmdLVmk4a1FTZVV6cHozN2lmbjZOVVh1cHBfVG9rZW46Q3A0TWJvaGk2b0g3d1N4YTlUemNBQUZkbndnXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

而processImport方法中，主要是有三部分，

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MDU2YTBkZGYxYTcxNmYwOWY3NjQ5NDdmMTk2OGJjMTdfQTRJOEJIeGJteW4wNjRUZnlyQlRhUkF4MzhqRm14TG1fVG9rZW46UzhlTmI5QktIb0tuYXF4bHY4UmNGd2NTbkhlXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

#### ImportSelector

我们以事务为例，@EnableTransactionManagement注解，引入了TransactionManagementConfigurationSelector

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NjZlNWM5NWEwYTFhYmU5NzEyMTAxNDFhNTc5MjUyNzRfeVZ3VHlXWHlVZXk4NExrTG1YN0FGUzF0bDN3aXRWVUtfVG9rZW46SUhsS2IwRHAwb2JZMU14M3hKZ2N1aTVnbnhiXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

在处理到有@EnableTransactionManagement的任意一个bean的时候，就会走到上面的第一种情况

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NDY5ZTY0ZTNmZGE4NjIxMjg4YWUyMDgyZDBjYzkzMDRfeWhNTnFUV1g2RHFLOUZRN3lJeDdwbDlrQ0h4UUtVUXVfVG9rZW46WExTS2JOOEkxb2xhNmR4Z0J2SWNwNTFGbnpkXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

selector字如其名，是通过某种选项来选择要构造的config类，

下面的是TransactionManagementConfigurationSelector的实现，由于他是继承的aop的selector，所以是由adviceMdoe决定的，两种返回的bean是不同的；比如默认是PROXY，则返回的是两个bean，{AutoProxyRegistrar,ProxyTransactionManagementConfiguration}，之后会递归再次调用processImport方法

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=OTg0YjMyMzhjMWU2ODI1ZmM0MjFkZTgwODU1OTc5YWVfWGozWHFWZFRvazJPSXJuQVRCamQwbU9WR1dIaWNZTkFfVG9rZW46VDluNmJvVlRGb2h6cHF4U01ETmNmbFpBbmxlXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

因此这种情况只是为了提供一种选择，最终落地的还是二三种情况

#### ImportBeanDefinitionRegistrar

事务的selector提供的AutoProxyRegistrar会在递归之后执行到第二种情况，最终添加到了BeanDefinitionRegister中去，实现了ImportBeanDefinitionRegistrar接口的类主要工作是实现registerBeanDefinitions方法，用于自定义beanDefinition，这个方法后面才会调到

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=OWRlOWNmNmU1ZjQ1MDBkYzdkZjg5NWJkZDliYjk0MmNfWHpDb3FveE1kRzlQT2VqV0dBcHAzZjlteDRLTTRIclRfVG9rZW46T0J2Q2I1c3I2b0tjWXp4Sm5QeGN0ZG1kbnhlXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

#### ConfigurationClass

事务的selector提供的第二个bean就是一个普通bean，最终会走到这种情况

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=OTU0MzhlZDc0NDQxNWEwMTgxM2Q3ZWI3YzgxMWRjYmFfWXBtVmFBZEl2empsWTJVN1ZYajBWSXA4SXg0c1hRcndfVG9rZW46WUlWNWJTeHJPb01xYTB4VTNZZmNadFF3blNnXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

在这种情况下，会立即构建这个类为普通的config类

importStack.registerImport会在实例化之前设置一个元数据，除此之外不知道有啥用

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NmJkYTQzM2RlNjFiMGQ1YWEyZmZkZTljZDEzY2FhMzdfelF1UUdJWXlsVFJXQ3ZLVHpUam9QQlZ3VmR1Rm9qSlZfVG9rZW46QjJTTmJmdDYwb2M2cjZ4TzhwTGN2dzJMbkFoXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

### 执行registerBeanDefinitions

这个方法的入口是在之前@bean处理的方法后面，具体工作就是注册自定义的bean

比如MapperScan，就是注册指定包下所有的mapper和service

具体实现具体分析

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=NmY1YzIxZDgzZWRiZWQ5NTJkODg5MjY0M2YwMWE4NTVfcHR1RHFFNklzQVBoeVY1cEJvZG9maDluWVVJVHd4SnJfVG9rZW46WEdzYWJES3FIbzNqSjN4c3VGTmNqdktabjhlXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTgyMWY0Yzc5NmY2OTdmZWFjNWIyYmYxZjY0YjJlYTlfbGppWUxWbU1MdWtEN3Z3R3FtMjlvcnlJNUdqVlRKU2ZfVG9rZW46U1RrZGJ3YVRzb0FkTUV4WmdzWmNqbkFQbnVnXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)

## 内部注册

内部注册是指spring自己注册一些关键的bean，比如ConfigurationClassPostProcessor，通常是调用的AnnotationConfigUtils.*registerAnnotationConfigProcessors方法注册的，一般是在refresh之前*

![](https://l8ut65fgfc.feishu.cn/space/api/box/stream/download/asynccode/?code=MWQ5YTBlNzBiZGE4NWRhZDY5Njg3ZTkzZGFhNWFiM2Vfa1dZak54U21qRW5iNEhPc2NMd0h6WDdqSGlMWGhBMmxfVG9rZW46UjdGNmJiOFRmb0wwSnV4aFJMTmNxOWVLbndiXzE3MjMzNTkwNTI6MTcyMzM2MjY1Ml9WNA)
