+++
title = "分析Skywalking的controller加强原理"
description = ""
date = 2023-09-02
image = ""
draft = false
slug = "SkywalkingController"
tags = ['中间件','Skywalking','监控']
categories = ['tech']
+++

在阅读本篇文章之前，建议先了解agent的启动流程，这对分析controller加强原理有很大的帮助，[Skywalking Agent启动流程源码分析]({{<ref "SkywalkingAgent启动流程源码分析">}})

项目启动时我指定skywalking[官方jar包](https://www.apache.org/dyn/closer.cgi/skywalking/java-agent/8.16.0/apache-skywalking-java-agent-8.16.0.tgz)

-javaagent:/xxxx/skywalking/skywalking-agent/skywalking-agent.jar

## InstMethodsInter

在agent启动原理那篇文章，我们知道Skywalking对于类的加强主要就是对静态方法、构造方法、实例方法各自的加强。虽然我们并没有深入研究AgentBuilder的每个方法，但通过附件的反编译可以了解到，对于要加强的类，

其本质是new出一个新的类，继承EnhancedInstance，然后创建了一堆的代理对象，以hello方法为例，我们实际调用的是InstMethodsInter类的一个实例对象的intercept方法；

intercept定义了三个钩子方法，分别是beforeMethod、handleMethodException和afterMethod。所以我们使用的插件一般来说也都是去实现定义的这三个方法。

```Java
private InstanceMethodsAroundInterceptor interceptor;
public Object intercept(@This Object obj, @AllArguments Object[] allArguments, @SuperCall Callable<?> zuper,
    @Origin Method method) throws Throwable {
    EnhancedInstance targetObject = (EnhancedInstance) obj;

    MethodInterceptResult result = new MethodInterceptResult();
    try {
        interceptor.beforeMethod(targetObject, method, allArguments, method.getParameterTypes(), result);
    } catch (Throwable t) {
        LOGGER.error(t, "class[{}] before method[{}] intercept failure", obj.getClass(), method.getName());
    }

    Object ret = null;
    try {
    //这里在beforeMethod中设定了isContinue为true的话，就直接返回result._ret()，等于是代理
        if (!result.isContinue()) {
            ret = result._ret();
        } else {
            ret = zuper.call();
        }
    } catch (Throwable t) {
        try {
            interceptor.handleMethodException(targetObject, method, allArguments, method.getParameterTypes(), t);
        } catch (Throwable t2) {
            LOGGER.error(t2, "class[{}] handle method[{}] exception failure", obj.getClass(), method.getName());
        }
        throw t;
    } finally {
        try {
            ret = interceptor.afterMethod(targetObject, method, allArguments, method.getParameterTypes(), ret);
        } catch (Throwable t) {
            LOGGER.error(t, "class[{}] after method[{}] intercept failure", obj.getClass(), method.getName());
        }
    }
    return ret;
}
```

## InstanceMethodsAroundInterceptor

在InstanceMethodsAroundInterceptor的before和after方法上都打上断点，请求hello接口，其核心链路如下图所示。

![](http://picgo.qisiii.asia/post/11-18-10-50-image.png)

TomcatInvokerInterceptor是在插件tomcat-7.x-8.x-plugin-8.15.0.jar中TomcatInstrumentation类的getMethodsInterceptor指定，对org.apache.catalina.core.StandardHostValve#invoke进行加强

GetBeanInterceptor是在插件apm-springmvc-annotation-5.x-plugin-8.15.0.jar中的HandlerMethodInstrumentation类getMethodsInterceptor指定，对org.springframework.web.method.HandlerMethod#getBean方法进行加强

RestMappingMethodInterceptor是在插件apm-springmvc-annotation-5.x-plugin-8.15.0.jar中的ControllerInstrumentation和RestControllerInstrumentation类所继承的AbstractControllerInstrumentation的getMethodsInterceptor指定（这里如果是@RequestMapping注解会走到RequestMappingMethodInterceptor，如果是@GetMapping、@PostMapping等rest的注解，则是会走到RestMappingMethodInterceptor，但其实这两个Interceptor都是继承的AbstractMethodInterceptor）

**GetBeanInterceptor、RestMappingMethodInterceptor都定义在插件apm-springmvc-annotation-commons-8.15.0.jar中**

### Interceptor源码分析

#### TomcatInvokerIntercetor

TomcatInvokerIntercetor相对比较独立，像是在容器层面的监听和统计，和mvc流程不掺和；逻辑也比较简单，就是将请求的url、method、head和响应状态封装到span里，上报到oap，这里就不贴源码分析了；

#### GetBeanInterceptor

GetBeanInterceptor的作用就是将request和response放到了thredlocal里面，方便RestMappingMethodInterceptor获取和处理

```Java
//GetBeanInterceptor
public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, Object ret) throws Throwable {
    if (ret instanceof EnhancedInstance) {
        ServletRequestAttributes requestAttributes = (ServletRequestAttributes)RequestContextHolder.getRequestAttributes();
        if (requestAttributes != null) {
            ContextManager.getRuntimeContext().put("SW_REQUEST", requestAttributes.getRequest());
            ContextManager.getRuntimeContext().put("SW_RESPONSE", requestAttributes.getResponse());
        }
    }

    return ret;
}
```

#### RestMappingMethodInterceptor

在分析这段代码之前，建议先看以下[官方文档](https://skywalking.apache.org/docs/skywalking-java/v8.16.0/en/setup/service-agent/java-agent/java-plugin-development-guide/#tracing-plugin)对于span的介绍，简单总结就是

Span 表示分布式系统中完成工作的单个单元，即一个组件的一次操作，可以被包含。

Skywalking 将span做了类型区分

EntrySpan 用来在服务间交互的时候记录数据

LocalSpan 用来在本地方法记录数据

ExitSpan 则表示endpoint的数据

```Java
public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, MethodInterceptResult result) throws Throwable {
    //SW_FORWARD_REQUEST_FLAG这个我没找到在哪里塞入
    Boolean forwardRequestFlag = (Boolean)ContextManager.getRuntimeContext().get("SW_FORWARD_REQUEST_FLAG");
    if (forwardRequestFlag == null || !forwardRequestFlag) {
    //依托于getBean方法的afterMethod塞到threadlocal的request
        Object request = ContextManager.getRuntimeContext().get("SW_REQUEST");
        if (request != null) {
        //字面意思，用来记录调用方法的深度
            StackDepth stackDepth = (StackDepth)ContextManager.getRuntimeContext().get("SW_CONTROLLER_METHOD_STACK_DEPTH");
            //这里是指进入controller之前，已经记录了一层，但我想象不到是哪种场景
            if (stackDepth != null) {
                AbstractSpan span = ContextManager.createLocalSpan(this.buildOperationName(objInst, method));
                span.setComponent(ComponentsDefine.SPRING_MVC_ANNOTATION);
            } else {
                ContextCarrier contextCarrier = new ContextCarrier();
                CarrierItem next;
                String operationName;
                AbstractSpan span;
                //区分request是HttpServletRequest、还是jakarta包的HttpServletRequest、或者是ServerHttpRequest
                if (IN_SERVLET_CONTAINER && IS_JAVAX && HttpServletRequest.class.isAssignableFrom(request.getClass())) {
                    HttpServletRequest httpServletRequest = (HttpServletRequest)request;
                    next = contextCarrier.items();
                    //从请求的header找有没有contextCarrier对象属性的值
                    while(next.hasNext()) {
                        next = next.next();
                        next.setHeadValue(httpServletRequest.getHeader(next.getHeadKey()));
                    }
                    //将请求url和方法提取并组合
                    operationName = this.buildOperationName(method, httpServletRequest.getMethod(), (EnhanceRequireObjectCache)objInst.getSkyWalkingDynamicField());
                    //创建EntrySpan，这里记录了开始时间，下面是记录URL、方法（Get、POST）、组件类型
                    span = ContextManager.createEntrySpan(operationName, contextCarrier);
                    Tags.URL.set(span, httpServletRequest.getRequestURL().toString());
                    HTTP.METHOD.set(span, httpServletRequest.getMethod());
                    span.setComponent(ComponentsDefine.SPRING_MVC_ANNOTATION);
                    SpanLayer.asHttp(span);
                    //默认是false，如果为true，则要记录入参请求
                    if (SpringMVC.COLLECT_HTTP_PARAMS) {
                        RequestUtil.collectHttpParam(httpServletRequest, span);
                    }
                    //指定要记录的header
                    if (!CollectionUtil.isEmpty(Http.INCLUDE_HTTP_HEADERS)) {
                        RequestUtil.collectHttpHeaders(httpServletRequest, span);
                    }
                } else if (IN_SERVLET_CONTAINER && IS_JAKARTA && jakarta.servlet.http.HttpServletRequest.class.isAssignableFrom(request.getClass())) {
                    jakarta.servlet.http.HttpServletRequest httpServletRequest = (jakarta.servlet.http.HttpServletRequest)request;
                    //和上面一样
                } else {
                    if (!ServerHttpRequest.class.isAssignableFrom(request.getClass())) {
                        throw new IllegalStateException("this line should not be reached");
                    }

                    ServerHttpRequest serverHttpRequest = (ServerHttpRequest)request;
                    //和上面一样
                }
                //初始化一个StackDepth，值为1
                stackDepth = new StackDepth();
                ContextManager.getRuntimeContext().put("SW_CONTROLLER_METHOD_STACK_DEPTH", stackDepth);
            }
            //每过一个before+1，每过一个after-1
            stackDepth.increment();
        }

    }
}

public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes, Object ret) throws Throwable {
    RuntimeContext runtimeContext = ContextManager.getRuntimeContext();
    Boolean forwardRequestFlag = (Boolean)runtimeContext.get("SW_FORWARD_REQUEST_FLAG");
    if (forwardRequestFlag != null && forwardRequestFlag) {
        return ret;
    } else {
        Object request = runtimeContext.get("SW_REQUEST");
        if (request != null) {
            try {
            //获取层深并减一
                StackDepth stackDepth = (StackDepth)runtimeContext.get("SW_CONTROLLER_METHOD_STACK_DEPTH");
                if (stackDepth == null) {
                    throw new IllegalMethodStackDepthException();
                }

                stackDepth.decrement();
                AbstractSpan span = ContextManager.activeSpan();
                if (stackDepth.depth() == 0) {
                    Object response = runtimeContext.get("SW_RESPONSE");
                    if (response == null) {
                        throw new ServletResponseNotFoundException();
                    }

                    Integer statusCode = null;
                    //记录请求response的code
                    if (IS_SERVLET_GET_STATUS_METHOD_EXIST && HttpServletResponse.class.isAssignableFrom(response.getClass())) {
                        statusCode = ((HttpServletResponse)response).getStatus();
                    } else if (IS_JAKARTA_SERVLET_GET_STATUS_METHOD_EXIST && jakarta.servlet.http.HttpServletResponse.class.isAssignableFrom(response.getClass())) {
                        statusCode = ((jakarta.servlet.http.HttpServletResponse)response).getStatus();
                    } else if (ServerHttpResponse.class.isAssignableFrom(response.getClass())) {
                        if (IS_SERVLET_GET_STATUS_METHOD_EXIST || IS_JAKARTA_SERVLET_GET_STATUS_METHOD_EXIST) {
                            statusCode = ((ServerHttpResponse)response).getRawStatusCode();
                        }

                        Object context = runtimeContext.get("SW_REACTIVE_RESPONSE_ASYNC_SPAN");
                        if (context != null) {
                        //响应是异步得专门看一下
                            ((AbstractSpan[])context)[0] = span.prepareForAsync();
                        }
                    }

                    if (statusCode != null) {
                        Tags.HTTP_RESPONSE_STATUS_CODE.set(span, statusCode);
                        if (statusCode >= 400) {
                            span.errorOccurred();
                        }
                    }

                    runtimeContext.remove("SW_REACTIVE_RESPONSE_ASYNC_SPAN");
                    runtimeContext.remove("SW_REQUEST");
                    runtimeContext.remove("SW_RESPONSE");
                    runtimeContext.remove("SW_CONTROLLER_METHOD_STACK_DEPTH");
                }
                //假如COLLECT_HTTP_PARAMS默认是false，那这里就会由ProfileStatus决定，而ProfileStatus由org.apache.skywalking.apm.agent.core.profile.ThreadProfiler#startProfilingIfNeed决定
                if (!SpringMVC.COLLECT_HTTP_PARAMS && span.isProfiling()) {
                    if (IS_JAVAX && HttpServletRequest.class.isAssignableFrom(request.getClass())) {
                        RequestUtil.collectHttpParam((HttpServletRequest)request, span);
                    } else if (IS_JAKARTA && jakarta.servlet.http.HttpServletRequest.class.isAssignableFrom(request.getClass())) {
                        RequestUtil.collectHttpParam((jakarta.servlet.http.HttpServletRequest)request, span);
                    } else if (ServerHttpRequest.class.isAssignableFrom(request.getClass())) {
                        RequestUtil.collectHttpParam((ServerHttpRequest)request, span);
                    }
                }
            } finally {
            //在这里进行的上报
                ContextManager.stopSpan();
            }
        }

        return ret;
    }
}
```

## 上报

到这里为止，可以发现其实就跟拦截器的思路是一样的，在执行前后做了一些记录操作。那么记录完之后，是如何上报到oap的呢?原理就在ContextManager.stopSpan()中

### 收集（生产）

```Java
public boolean stopSpan(AbstractSpan span) {
    AbstractSpan lastSpan = this.peek();
    if (lastSpan == span) {
        if (lastSpan instanceof AbstractTracingSpan) {
            AbstractTracingSpan toFinishSpan = (AbstractTracingSpan)lastSpan;
            //这个finish是AbstractTracingSpan的finish，
            if (toFinishSpan.finish(this.segment)) {
                this.pop();
            }
        } else {
            this.pop();
        }
        //这个finish是TraceContent的finish
        this.finish();
        return this.activeSpanStack.isEmpty();
    } else {
        throw new IllegalStateException("Stopping the unexpected span = " + span);
    }
}
```

org.apache.skywalking.apm.agent.core.context.ContextManager#stopSpan()->

org.apache.skywalking.apm.agent.core.context.trace.AbstractTracingSpan#finish，

从 stopSpan跟进去，会先走到AbstractTracingSpan#finish这个方法，在这里将本次请求的span放到了TraceSegment.spans集合中，等待被收集和消费

```Java
public boolean finish(TraceSegment owner) {
    this.endTime = System.currentTimeMillis();
    owner.archive(this);
    return true;
}
public void archive(AbstractTracingSpan finishedSpan) {
    this.spans.add(finishedSpan);
}
```

然后会走到TraceContent的finish，org.apache.skywalking.apm.agent.core.context.TracingContext.ListenerManager#notifyFinish->...->

#org.apache.skywalking.apm.commons.datacarrier.DataCarrier#produce

```Java
private void finish() {
...
//下面是调用了两次notifyFinish，但是通知的内容不一样，这里我们主要关注第二个，因为通知的是TraceSegment
        boolean isFinishedInMainThread = this.activeSpanStack.isEmpty() && this.running;
        if (isFinishedInMainThread) {
            TracingContext.TracingThreadListenerManager.notifyFinish(this);
        }

        if (isFinishedInMainThread && (!this.isRunningInAsyncMode || this.asyncSpanCounter == 0)) {
            TraceSegment finishedSegment = this.segment.finish(this.isLimitMechanismWorking());
            TracingContext.ListenerManager.notifyFinish(finishedSegment);
            this.running = false;
        }
        ...

}
//一层层跟下去
static void notifyFinish(TraceSegment finishedSegment) {
    Iterator var1 = LISTENERS.iterator();

    while(var1.hasNext()) {
        TracingContextListener listener = (TracingContextListener)var1.next();
        listener.afterFinished(finishedSegment);
    }

}
public void afterFinished(TraceSegment traceSegment) {
    if (!traceSegment.isIgnore()) {
        if (!this.carrier.produce(traceSegment) && LOGGER.isDebugEnable()) {
            LOGGER.debug("One trace segment has been abandoned, cause by buffer is full.");
        }

    }
}
#org.apache.skywalking.apm.commons.datacarrier.DataCarrier#produce
public boolean produce(T data) {
    return this.driver != null && !this.driver.isRunning(this.channels) ? false : this.channels.save(data);
}

public void save(data){
    while(条件){
        this.bufferChannels[index].save(data)
    }
）
```

直到这里为止，我们可以发现数据被存到了DataCarrier.channels中，而且从执行的方法是produce可以猜到，对应的consume方法就是消费逻辑。

### 消费

还是在DataCarrier中，通过下面的代码可以发现，实际一个ConsumeDriver有多个线程，每个线程对应一个channel，是以属性dataSource绑定到ConsumerThread上

```Java
public DataCarrier consume(Class<? extends IConsumer<T>> consumerClass, int num, long consumeCycle, Properties properties) {
    if (this.driver != null) {
        this.driver.close(this.channels);
    }
    //将channels放入ConsumeDriver
    this.driver = new ConsumeDriver(this.name, this.channels, consumerClass, num, consumeCycle, properties);
    this.driver.begin(this.channels);
    return this;
}

public void begin(Channels channels) {
...
//将chennel和ConsumerThread做绑定
            this.allocateBuffer2Thread();
            ConsumerThread[] var2 = this.consumerThreads;
            int var3 = var2.length;

            for(int var4 = 0; var4 < var3; ++var4) {
                ConsumerThread consumerThread = var2[var4];
                consumerThread.start();
            }
}
private void allocateBuffer2Thread() {
    int channelSize = this.channels.getChannelSize();

    for(int channelIndex = 0; channelIndex < channelSize; ++channelIndex) {
        int consumerIndex = channelIndex % this.consumerThreads.length;
        //将channel放到ConsumerThread的dataSource中，因为后面每个线程的主要工作就是从datasource获取要要发送的数据，然后发送
        this.consumerThreads[consumerIndex].addDataSource(this.channels.getBuffer(channelIndex));
    }

}
```

所以ConsumerThread持有datasource，然后datasource又有了数据,线程在死循环获取datasource中数据就可以了

```Java
#org.apache.skywalking.apm.commons.datacarrier.consumer.ConsumerThread#consume
private boolean consume(List<T> consumeList) {
    Iterator var2 = this.dataSources.iterator();
    while(var2.hasNext()) {
        ConsumerThread<T>.DataSource dataSource = (ConsumerThread.DataSource)var2.next();
        //将datasource数据放入consumeList
        dataSource.obtain(consumeList);
    }
    ...
    this.consumer.consume(consumeList);
    ...
}
#org.apache.skywalking.apm.agent.core.remote.TraceSegmentServiceClient#consume
public void consume(List<TraceSegment> data) {
    if (GRPCChannelStatus.CONNECTED.equals(this.status)) {
        final GRPCStreamServiceStatus status = new GRPCStreamServiceStatus(false);
        StreamObserver upstreamSegmentStreamObserver = ((TraceSegmentReportServiceStub)this.serviceStub.withDeadlineAfter((long)Collector.GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS)).collect(new StreamObserver<Commands>() {
            //这里的onNext是response的onNext，不是request，可以看一下collect的逻辑，所以下面直接调onNext并不是调到这个方法
            public void onNext(Commands commands) {
                ((CommandService)ServiceManager.INSTANCE.findService(CommandService.class)).receiveCommand(commands);
            }
            ...
        });
        ...
            while(var4.hasNext()) {
                TraceSegment segment = (TraceSegment)var4.next();
                //构建传输对象
                SegmentObject upstreamSegment = segment.transform();
                //进行传输
                upstreamSegmentStreamObserver.onNext(upstreamSegment);
            }
        ...
}
//走发送逻辑，由于GRPC我还没有研究过，所以暂不分析往下的源码
public void onNext(ReqT value) {
    Preconditions.checkState(!this.aborted, "Stream was terminated by error, no further calls are allowed");
    Preconditions.checkState(!this.completed, "Stream is already completed, no further calls are allowed");
    this.call.sendMessage(value);
}
```

## 代码

### Controller方法

```Java
@RestController
@RequestMapping("/first")
public class FirstRequestController {
    @GetMapping("/hello")
    public String hello() {
        System.out.println("hello,mybatis");
        System.out.println(testAgent());
        System.out.println(noAgent());;
        return "mybatis";
    }
    private String testAgent(){
        return "testAgent";
    }
    public String noAgent(){
        return "noAgent";
    }
}
```

### 反编译FirstRequestController

```Java
       @RestController
       @RequestMapping(value={"/first"})
       public class FirstRequestController
       implements EnhancedInstance {
           private volatile Object _$EnhancedClassField_ws;
           public static volatile /* synthetic */ InstMethodsInter delegate$fmp8mm0;
           public static volatile /* synthetic */ InstMethodsInter delegate$usgtuf1;
           public static volatile /* synthetic */ ConstructorInter delegate$ve3rcd0;
           public static volatile /* synthetic */ InstMethodsInter delegate$9q0b6r1;
           public static volatile /* synthetic */ InstMethodsInter delegate$mft2580;
           public static volatile /* synthetic */ ConstructorInter delegate$gcp4qa1;
           public static volatile /* synthetic */ InstMethodsInter delegate$4vl1o10;
           public static volatile /* synthetic */ InstMethodsInter delegate$vcjgr61;
           public static volatile /* synthetic */ ConstructorInter delegate$di6p0h0;
           public static volatile /* synthetic */ InstMethodsInter delegate$jfo0r51;
           public static volatile /* synthetic */ InstMethodsInter delegate$dg0j500;
           public static volatile /* synthetic */ ConstructorInter delegate$5c04f20;
           private static final /* synthetic */ Method cachedValue$aWq206nq$b4bilp1;
           public static volatile /* synthetic */ InstMethodsInter delegate$306a1i0;
           public static volatile /* synthetic */ InstMethodsInter delegate$3h5imi1;
           public static volatile /* synthetic */ ConstructorInter delegate$25tu2j0;
           public static volatile /* synthetic */ InstMethodsInter delegate$su4hhm0;
           public static volatile /* synthetic */ InstMethodsInter delegate$e5jt5f1;
           public static volatile /* synthetic */ ConstructorInter delegate$kc72041;
           public static volatile /* synthetic */ InstMethodsInter delegate$pf9hmj0;
           public static volatile /* synthetic */ InstMethodsInter delegate$kik32a0;
           public static volatile /* synthetic */ ConstructorInter delegate$os3qlu1;
           public static volatile /* synthetic */ InstMethodsInter delegate$0e263f0;
           public static volatile /* synthetic */ InstMethodsInter delegate$a5ji980;
           public static volatile /* synthetic */ ConstructorInter delegate$jc027h1;
           private static final /* synthetic */ Method cachedValue$niIMIXK0$b4bilp1;

           public FirstRequestController() {
               this(null);
               delegate$25tu2j0.intercept(this, new Object[0]);
           }

           private /* synthetic */ FirstRequestController(auxiliary.MVLCe3Gn mVLCe3Gn) {
           }

           @GetMapping(value={"/hello"})
           public String hello() {
               return (String)delegate$306a1i0.intercept(this, new Object[0], (Callable<?>)new auxiliary.4sHLGVLS(this), cachedValue$niIMIXK0$b4bilp1);
           }

           private /* synthetic */ String hello$original$UjAdUjOr() {
/*17*/         System.out.println("hello,mybatis");
/*18*/         System.out.println(this.testAgent());
/*19*/         System.out.println(this.noAgent());
/*20*/         return "mybatis";
           }

           private String testAgent() {
/*23*/         return "testAgent";
           }

           public String noAgent() {
/*26*/         return "noAgent";
           }

           static {
               ClassLoader.getSystemClassLoader().loadClass("org.apache.skywalking.apm.dependencies.net.bytebuddy.dynamic.Nexus").getMethod("initialize", Class.class, Integer.TYPE).invoke(null, FirstRequestController.class, 1021095365);
               cachedValue$niIMIXK0$b4bilp1 = FirstRequestController.class.getMethod("hello", new Class[0]);
           }

           final /* synthetic */ String hello$original$UjAdUjOr$accessor$niIMIXK0() {
               return this.hello$original$UjAdUjOr();
           }
       }
```

### 反编译HandlerMethod

```Java
public class HandlerMethod
        implements EnhancedInstance {
    private static final /* synthetic */ Method cachedValue$1RDdupwP$41avq00;
    public Object getBean() {
        return delegate$shr7st0.intercept(this, new Object[0], (Callable<?>) new auxiliary.ENGom7ym(this), cachedValue$1RDdupwP$41avq00);
    }
    /*
     * Enabled aggressive block sorting
     */
    static {
        ClassLoader.getSystemClassLoader().loadClass("org.apache.skywalking.apm.dependencies.net.bytebuddy.dynamic.Nexus").getMethod("initialize", Class.class, Integer.TYPE).invoke(null, HandlerMethod.class, -1058650667);
        cachedValue$1RDdupwP$41avq00 = HandlerMethod.class.getMethod("getBean", new Class[0]);
        /* 66*/
        logger = LogFactory.getLog(HandlerMethod.class);
    }
}
```
