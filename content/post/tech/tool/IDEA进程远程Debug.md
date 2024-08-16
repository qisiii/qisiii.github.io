+++
title = "IDEA进程远程Debug"
description = ""
date = 2023-07-10
image = ""
draft = false
slug = "remote_debug"
tags = ['tool','idea']
categories = ['tech']
+++

## 普通的应用启动

新建一个Remote_JVM_Debug配置

![](http://picgo.qisiii.asia/post/1723664070953.jpg)

填入自己的服务器的ip和任意一个没有被占用的端口号（非当前应用启动用的端口号），然后重点是下面这段代码

```java
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8100
```

![](http://picgo.qisiii.asia/post/1723664128443.jpg)

在服务器启动对应应用，上面提到的代码必须在 java -jar xxx.jar 的-jar前面，而我的启动命令是下面这样，绿色为新插入的

```C
nohup java -Xms128m -Xmx128m -Xmn40m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./logs/jvm -Xloggc:./logs/jvm/gc.log  -XX:+PrintGCDetails -XX:+PrintGCDateStamps -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8100  -jar callback-1.0-SNAPSHOT.jar --spring.profiles.active=prod &
```

在确认服务器启动之后，在本地点debug启动按钮，如果打印了下面这个语句，就证明连接成功

![](http://picgo.qisiii.asia/post/1723664107079.jpg)

之后在你需要排查的地方打上断点，然后触发接口就可以了，就会发现就进入了熟悉的debug的模式里

![](http://picgo.qisiii.asia/post/1723664149754.jpg)

## docker启动

IDEA的配置不变，整体的区别就是docker在打包时要暴露出来调试的接口，启动命令-p 映射端口也要有两个

dockerfile如下

```Dockerfile
FROM openjdk:8
LABEL maintainer="qisiii@example.com"
WORKDIR /app
#这里的路径不能使用父级的,即../
COPY callback-1.0-SNAPSHOT.jar /app/app.jar
#这里是服务应用的端口
EXPOSE 8101
#这里是debug端口
EXPOSE 8100
ENV JAVA_OPTS=''
ENV JAVA_OPTS2='--spring.profiles.active=prod'
ENTRYPOINT ["sh","-c","java ${JAVA_OPTS} -jar app.jar ${JAVA_OPTS2}"]
```

```Dockerfile
docker run -e "JAVA_OPTS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8100" -d -p 8101:8101 -p 8100:8100 --name open-platform  open-platform:1.0.0
```

## k8s启动

idea本身的配置和通过docker启动一样，只需要修改k8s里相关的配置就可以了

镜像依然使用docker那个镜像，我是在deployment对应的yml文件中添加的里面的环境变量

```Dockerfile
apiVersion: apps/v1  #kubectl api-versions 可以通过这条指令去看版本信息kind: Deployment # 指定资源类别metadata: #资源的一些元数据  name: open-platform #deloyment的名称  labels:    app: open-platform  #标签spec:  replicas: 1 #创建pod的个数  selector:    matchLabels:      app: open-platform #满足标签为这个的时候相关的pod才能被调度到  template:    metadata:      labels:        app: open-platform    spec:      imagePullSecrets:      - name: aliyun      containers:        - name: open-platform          image: registry.cn-hangzhou.aliyuncs.com/qisiii/open-platform:1.0.1          imagePullPolicy: IfNotPresent          ports:            - containerPort: 8101          env:            - name: JAVA_OPTS              value: "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8100"
```

然后我在service里面暴露了端口号

```Dockerfile
{                                           
"kind": "Service",                          
    "apiVersion": "v1",                     
    "metadata": {                           
        "name": "open-platform-service"     
    },                                      
    "spec": {                               
        "type":"NodePort",                  
        "selector": {                       
            "app": "open-platform"          
        },                                  
        "ports": [                          
            {                               
                "protocol": "TCP",          
                "port": 80,                 
                "targetPort": 8101,         
                "name": "open-platform"     
            },                              
            {                               
                "protocol": "TCP",          
                "port": 8100,               
                "targetPort": 8100,         
                "name": "debug"             
            }                               
        ]                                   
    }                                       
}                                           
```

并且在最后通过post-forward转发时同时转发了两个端口号

```Dockerfile
kubectl port-forward --address 0.0.0.0 service/open-platform-service 8101:80 8100:8100
```

这个时候本地idea启动debug，打上断点，远程触发访问

![](http://picgo.qisiii.asia/post/1723664166715.jpg)

# 参考文档：

https://zhuanlan.zhihu.com/p/128033093

https://www.cnblogs.com/gaoyuechen/p/11811180.html
