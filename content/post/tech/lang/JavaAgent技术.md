+++
title = 'JavaAgent技术'
date = 2023-09-10
draft = false
tags = ['java','agent']
categories = ['tech']
slug = "javaagent"
+++

> 在定位公司问题的时候，需要了解一下skywalking的相关知识，而agent就提上了日程。

[官网文档](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html "官网文档")

Agent技术是Jdk在1.5版本之后，所提供的一个在jvm启动前后对部分java类代理加强的机制。由于是直接修改字节码，并不会对业务代码有注入，所以可以很好的应用于监控或者热部署等场景。

正常所提到的Agent一般都是部署成jar包的样子，比如agent-1.0-SNAPSHOT.jar。

在这个jar包中，要添加一个MANIFEST.MF文件，在文件中指定jar包的代理类，比如下面代码中的Premain-Class。

在对应的代理类，要实现一个permain方法或者agentmain方法，这样jvm可以通过MANIFEST找到类，通过类再找到对应的方法，从而进行加强，所以加强逻辑是在permain方法或者agentmain方法内部实现的。

```webmanifest
Manifest-Version: 1.0
Built-By: qisi
Premain-Class: com.qisi.agent.InterviewAgent
Agent-Class: com.qisi.agent.InterviewAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Class-Path: byte-buddy-1.10.22.jar
Created-By: Apache Maven 3.8.1
Build-Jdk: 1.8.0_332
```

```java
public class InterviewAgent {
    public static void premain(String agentArgs, Instrumentation instrumentation) {
    }
    public static void agentmain(String agentArgs, Instrumentation instrumentation) {
    }
}
```

而如果在permain或者agentmain方法打上debug可以发现，执行时是通过sun.instrument.InstrumentationImpl#loadClassAndCallPremain和sun.instrument.InstrumentationImpl#loadClassAndCallAgentmain两个方法通过反射来执行到我们指定的类的。

Agent技术有两种场景，一种是在jvm启动之前，通过-javaagent:path来指定jar包，像是skywalking就是采用的这种方式；另一种则是在jvm启动之后，通过attach指定的进程，对jvm中的类进行加强，arthas就是采用的这种方式。

在具体介绍这两种方式之前，需要先讲一下Instrumentation相关类和接口

## java.lang.Instrumentation

### Instrumentation

Instrumentation相关的类都在java.lang.Instrumentation包下，两个异常，两个接口，一个类。
![](http://picgo.qisiii.asia/post/202407299e1128f6310d99cc3d00227b4a2055db.png) 

两个异常在这里不做介绍，功能就像类名一样。核心的其实是Instrumentation接口，本文仅关注红框内的几个方法。这几个方法都是通过permain和agentmain获取到的instrumentation实例进行的操作。

![](http://picgo.qisiii.asia/post/2024072959dadc1a00d2567d75bced09ca36b3f0.png)

从时间发展来看，其中jdk1.5开始支持的是下面几个方法，也就是说在jdk5的时候，仅支持添加和移除类转换器，且添加的类转换器只能在加载和重定义的时候使用。就是说如果类没有加载，那么通过addTransformer方法注册的ClassFileTransformer就可以对这个类进行增强，否则一旦类已经加载完毕，则只能通过redefineClasses，完全替换类定义再次触发loadClass来增强

```
addTransformer(ClassFileTransformer transformer)
removeTransformer(ClassFileTransformer transformer)
isRedefineClassesSupported();//依赖于MANIFEST中的Can-Redefine-Classes值
redefineClasses(ClassDefinition... definitions)
```

而从jdk1.6开始，增加了一个retransformClasses的概念。retransform和redefine的区别，前者是在原有类的基础上进行修改，后者则是完全重定义，不使用原有类做任何参考。
需要注意的事，只有在首次调用addTransformer时，将canRetransform设置为true的类，才可以被重新转换。

```
addTransformer(ClassFileTransformer transformer, boolean canRetransform);
isRetransformClassesSupported();
retransformClasses(Class<?>... classes)//依赖于MANIFEST中的Can-Retransform-Classes值
isModifiableClass(Class<?> theClass);
```

### ClassFileTransformer、ClassDefinition

这两个类其实都是Instrumentation接口方法的入参，其中用的比较多的应该是ClassFileTransformer。这个类只有一个transform，jvm类加载的时候都会调用一遍这个方法。如果需要加强，那么就利用给定的参数，进行字节码的改动，将改动后的字节码作为返回值返回；如果无需增强，则直接返回null即可。

```
byte[]
transform(  ClassLoader         loader,
            String              className,
            Class<?>            classBeingRedefined,
            ProtectionDomain    protectionDomain,
            byte[]              classfileBuffer)
```

ClassDefinition也类似，不过是在对象里重新绑定class和byte的关系

```
public final class ClassDefinition {
    /**
     *  The class to redefine
     */
    private final Class<?> mClass;

    /**
     *  The replacement class file bytes
     */
    private final byte[]   mClassFile;
```

## 实践

### MANIFEST.MF配置

在pom文件中添加下面的代码，根据需要修改参数值

```
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <configuration>
                <archive>
                    <addMavenDescriptor>false</addMavenDescriptor>
                    <manifest>
                        <addClasspath>true</addClasspath>
                    </manifest>
                    <manifestEntries>
                        <Premain-Class>
                            com.qisi.agent.InterviewByteButtyAgent
                        </Premain-Class>
                        <Agent-Class>
                            com.qisi.agent.InterviewByteButtyAgent
                        </Agent-Class>
                        <Can-Redefine-Classes>
                            true
                        </Can-Redefine-Classes>
                        <Can-Retransform-Classes>
                            true
                        </Can-Retransform-Classes>
                        <Built-By>
                            qisi
                        </Built-By>
                    </manifestEntries>
                </archive>
            </configuration>
        </plugin>
    </plugins>
</build>
```

#### -javaagent:

在这种方式下，起作用的是permain，也就是说-javaagent和permain方法是配套使用的。
核心就是添加一个自定义的ClassFileTransformer，可以另起一个类，也可以这样匿名类。
如果只是熟悉流程可以像下面一样，直接打印一些日志，不去修改类；

```
    public static void premain(String agentArgs, Instrumentation instrumentation) {
        System.out.println("enhance by premain,params:"+agentArgs);
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

如果要真实修改，需要引入asm javassist bytebuddy等修改字节码的框架。下面这部分就是使用了bytebuddy，作用是让任何类的testAgent方法，都返回固定值transformed

```
public static void premain(String agentArgs, Instrumentation instrumentation) throws ClassNotFoundException {
    System.out.println("enhance by permain InterviewByteButtyAgent,params:"+agentArgs);
    new AgentBuilder.Default().type(any()).transform(new AgentBuilder.Transformer() {
        @Override
        public DynamicType.Builder<?> transform(DynamicType.Builder<?> builder, TypeDescription typeDescription, ClassLoader classLoader, JavaModule module) {
            return builder.method(named("testAgent"))
                    .intercept(FixedValue.value("transformed"));
        }
    }).installOn(instrumentation);
}
```

编写完之后，就可以在任意项目添加一个存在testAgent方法的进行尝试了，比如
` java -javaagent:/xxxx/path/agent-1.0-SNAPSHOT.jar=key1:value1,key2:value2 -jar AppDemo.jar`

#### attach

##### agentmain

这种方式需要实现agentmain方法，和permian不太一样的地方是需要在addTransformer之后触发需要retransformClasses想要加强的类。

```java
public static void agentmain(String agentArgs, Instrumentation instrumentation) {
    System.out.println("enhance by agentmain,params:"+agentArgs);
    instrumentation.addTransformer(new ClassFileTransformer() {
        @Override
        public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
                                ProtectionDomain protectionDomain, byte[] classfileBuffer)
                throws IllegalClassFormatException {
            System.out.println("agentmain load Class     :" + className);
            return classfileBuffer;
        }
    }, true);
    try {
        instrumentation.retransformClasses(Class.forName("com.qisi.mybatis.app.controller.FirstRequestController"));
    } catch (UnmodifiableClassException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
}
```

同样，提供一个bytebuddy的例子，下面这个则是指定修改FirstRequestController的testAgent方法的返回值为transformed

```java
public static void agentmain(String agentArgs, Instrumentation instrumentation) throws ClassNotFoundException {
    System.out.println("enhance by agentmain InterviewByteButtyAgent,params:"+agentArgs);
    //这里RedefinitionStrategy必须注意，默认的DISABLED是不支持retransform
    new AgentBuilder.Default().with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION).type(new AgentBuilder.RawMatcher() {
        @Override
        public boolean matches(TypeDescription typeDescription, ClassLoader classLoader, JavaModule module, Class<?> classBeingRedefined, ProtectionDomain protectionDomain) {
            return typeDescription.getName().contains("FirstRequestController");
        }
    }).transform(new AgentBuilder.Transformer() {
        @Override
        public DynamicType.Builder<?> transform(DynamicType.Builder<?> builder, TypeDescription typeDescription, ClassLoader classLoader, JavaModule module) {
            System.out.println("enhance"+typeDescription.getName());
            return builder.method(named("testAgent"))
                    .intercept(FixedValue.value("transformed"));
        }
        //这里采用disableClassFormatChanges的方案，好像还可以使用advice
    }).disableClassFormatChanges().installOn(instrumentation);
    try {
        instrumentation.retransformClasses(Class.forName("com.qisi.mybatis.app.controller.FirstRequestController"));
    } catch (UnmodifiableClassException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
}
```

##### VirtualMachine

不同于-javaagent命令，这里需要使用自jdk6开始提供的VirtualMachine类，在tool.jar包里
下面的方法是我参考arthas写的一个attach的流程，选择我们想要attach的进程，然后加载我们上面写好的jar包就好了。

```java
public class AgentTest {
    public static void main(String[] args) throws IOException, AttachNotSupportedException {
        String pid = null;
        try {
            Process jps = Runtime.getRuntime().exec("jps");
            InputStream inputStream = jps.getInputStream();
            BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
            String line;
            while ((line = bufferedReader.readLine()) != null) {
                System.out.println(line);
            }
            System.out.println("选择要attach的进程");
            pid= new Scanner(System.in).nextLine();
            System.out.println("选择的pid是"+pid);
        } catch (IOException e) {
            e.printStackTrace();
        }
        for (VirtualMachineDescriptor virtualMachineDescriptor : VirtualMachine.list()) {
            if (virtualMachineDescriptor.id().equals(pid)){
                VirtualMachine attach = VirtualMachine.attach(virtualMachineDescriptor);
                try {
                    attach.loadAgent("/xxxxx/agent/target/agent-1.0-SNAPSHOT.jar","参数1，参数2");
                } catch (AgentLoadException e) {
                    e.printStackTrace();
                } catch (AgentInitializationException e) {
                    e.printStackTrace();
                } finally {
                    attach.detach();
                }
                break;
            }
        }
    }
}
```

# 参考文档：

[探秘 Java  热部署二（Java agent premain）](https://www.jianshu.com/p/0bbd79661080 "探秘 Java  热部署二（Java agent premain）")

[JAVA热更新1:Agent方式热更 | 花隐间-JAVA游戏技术解决方案](https://yeas.fun/archives/hotswap-agent "JAVA热更新1:Agent方式热更 | 花隐间-JAVA游戏技术解决方案")

[ByteBuddy入门教程](https://zhuanlan.zhihu.com/p/151843984 "ByteBuddy入门教程")
