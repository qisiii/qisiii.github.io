+++
title = '单服务器Redis集群部署记录'
date = 2024-07-30T23:27:58+08:00
draft = false
tags = ['redis','中间件']
categories = ['tech']
slug ="redis_cluster_deploy"
+++

## 部署

参考[Redis 集群教程](https://redis.com.cn/topics/cluster-tutorial.html)使用单云服务器启动多个节点来当做redis集群，这里使用7000-7005这6个节点，三主三从

```shell
#准备好文件夹
mkdir redis-cluster
cd redis-cluster
mkdir 7000 7001 7002 7003 7004 7005
#下载redis
wget https://download.redis.io/redis-stable.tar.gz
tar -zxvf redis-stable.tar.gz
cd redis-stable
make
#回到redis-cluster &&新建公共配置文件
cd redis-cluster
touch redis.conf

##redis.conf 模板内容
port 7000 #单节点的端口号
cluster-enabled yes #开启集群模式
cluster-config-file "nodes.conf" #集群配置文件，启动后节点后会自动新建
cluster-node-timeout 5000 #节点能够失联的最大时间
appendonly yes #aof持久化
protected-mode no #方便其他机器访问云服务器
cluster-announce-ip 服务器公网IP #这是在通过spring连接集群时需要配置的，不配置的话会连接局域网ip


#复制模板redis.conf到6个端口号文件夹
echo 7000 7001 7002 7003 7004 7005 | xargs -n 1 cp -v redis.conf
修改每个文件夹下的redis.conf的port，和文件夹名保持一致


#启动6个redis实例
cd 7000
nohup ../redis-stable/src/redis-server ./redis.conf &
cd 7001
nohup ../redis-stable/src/redis-server ./redis.conf &
...
或者在redis-cluster目录直接启动所有
nohup ./redis-stable/src/redis-server 7000/redis.conf >> ./7000/nohup.out &;nohup ./redis-stable/src/redis-server 7001/redis.conf >> ./7001/nohup.out &;nohup ./redis-stable/src/redis-server 7002/redis.conf >> ./7002/nohup.out &;nohup ./redis-stable/src/redis-server 7003/redis.conf >> ./7003/nohup.out &;nohup ./redis-stable/src/redis-server 7004/redis.conf >> ./7004/nohup.out &;nohup ./redis-stable/src/redis-server 7005/redis.conf >> ./7005/nohup.out &
```

全部启动完之后，可以通过`ps -ef | grep redis`来确认正常启动，且是cluster模式

![](http://picgo.qisiii.asia/post/30-23-48-11-image.png)

使用以下命令新建集群

`redis-stable/src/redis-cli --cluster create 公网ip:7000 公网ip:7001 公网ip:7002 公网ip:7003 公网ip:7004 公网ip:7005 --cluster-公网ip --cluster-replicas 1`

当看到` [OK] All 16384 slots covered`表明启动成功

### 设置密码

```
依次访问每个节点，并进行设置
redis-cli -c -p 端口
config set requirepass 密码
config set masterauth 密码
config rewrite
```

设置完之后去看conf文件会发现改变了以下三行

```
user default on sanitize-payload #94eef65a0de53368e43fe0f363193bebcc0fb13784a3b23defa1e8616f0af ~* &* +@all
requirepass "密码"
masterauth "密码"
```

### 关闭&重启&删除

#### 关闭集群

依次连接每个节点`redis-cli -c -p [端口]`并输入shutdown命令或者直接用kill -15 掉对应的pid

鉴于我们启动的节点较多，可以通过`ps -ef | grep redis | cut -c 9-15 | xargs kill -15` 批量删除

#### 重启集群

重新启动节点，会自动加入到集群里

```
#启动6个redis实例
cd 7000
nohup ../redis-stable/src/redis-server ./redis.conf &
cd 7001
nohup ../redis-stable/src/redis-server ./redis.conf &
```

### 删除集群

删除集群需要先关闭集群的所有节点，然后删除每个节点的集群自动生成的文件，可以通过命令批量删除`rm -rf 700*/nodes.conf 700*/dump.rdb 700*/appendonlydir`

![](http://picgo.qisiii.asia/post/31-09-14-53-image.png)

## 使用

### redis-cli

通过redis-cli 连接任一节点，添加或者获取缓存时会计算key落到哪个节点（crc16(key)%16384)，然后重定向到该节点再去获取或者赋值。

![](http://picgo.qisiii.asia/post/31-08-48-25-image.png)

### redisinsight

创建新的db连接，Host中是公网ip，创建成功后会显示类别是OSS Cluster，普通的单机类型则是Standalone

![](http://picgo.qisiii.asia/post/31-08-53-25-image.png)

![](http://picgo.qisiii.asia/post/31-08-55-03-image.png)

里面会显示所有的节点的key，刚才在redis-cli设置值的时候，key【hello】是7000节点，key【hello】是7002节点，这里都会显示出来

![](http://picgo.qisiii.asia/post/31-08-55-46-image.png)同时，在Analysis Tool看板会显示集群的一些信息

![](http://picgo.qisiii.asia/post/31-08-57-45-image.png)

### Spring连接

引入依赖，设置配置文件，成功启动后会打印出world；

```
  #pom.xml
  <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
        <version>2.5.0</version>
  </dependency>
 #application.yml
  spring:
  redis:
    cluster:
      nodes: [公网ip]:7000,[公网ip]:7001,[公网ip]:7002,[公网ip]:7003,[公网ip]:7004,[公网ip]:7005
    password: 密码
@SpringBootApplication
public class RedisClusterApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(RedisClusterApplication.class);
        StringRedisTemplate template = context.getBean(StringRedisTemplate.class);
        template.opsForValue().set("hello","world");
        String value=template.opsForValue().get("hello");
        System.out.println(value);
    }
}
```

## 注意事项：

1. `redis-cli --cluster create`命令里公网ip如果是127.0.0.1或者内网ip的话，可以正常启动集群，也可以通过redis-cli 命令连接。但是无法通过spring来启动，且无法通过redis的客户端redisinsight进行连接。
   
   spring会报错`Unable to connect to [127.0.0.1:7001]: Connection refused: /127.0.0.1:7001`这样的错误
   
   redisinsight会报`Could not connect to [公网ip:端口号],please check the connection details`

2. 安全组和防火墙都需要开启7000-7005和17000-17005端口号，其中端口号+10000是集群总线的端口，不然的话通过`redis-cli --cluster create`命令创建集群的时候，`waiting for the cluster to join`后一直等待

3. 配置文件中需要指定cluster-announce-ip 为公网ip，不然通过spring去连接的时候，哪怕你创建命令指定的是公网ip，但依然会报`Unable to connect to [内网ip:7001]: Connection refused: /内网ip:7001`

4. 配置`protected-mode no`其实是远程连通redis的一种方案，在不设置且没有密码的情况下会报下面这个错误，正如NOTE中所述，选择其中任一一种就可以了。
   
   ```
   -DENIED Redis is running in protected mode because protected mode is enabled, no bind address was specified, no authentication password is requested to clients. In this mode connections are only accepted from the loopback interface. If you want to connect from external computers to Redis you may adopt one of the following solutions:
   
   1) Just disable protected mode sending the command 'CONFIG SET protected-mode no' from the loopback interface by connecting to Redis from the same host the server is running, however MAKE SURE Redis is not publicly accessible from internet if you do so. Use CONFIG REWRITE to make this change permanent.
   
   2) Alternatively you can just disable the protected mode by editing the Redis configuration file, and setting the protected mode option to 'no', and then restarting the server.
   
   3) If you started the server manually just for testing, restart it with the '--protected-mode no' option.
   
   4) Setup a bind address or an authentication password.
   
   NOTE: You only need to do one of the above things in order for the server to start accepting connections from the outside.
   ```

# 参考文档:

[Redis 集群教程](https://redis.com.cn/topics/cluster-tutorial.html)

[springboot lettcue连接redis cluster定时刷新拓扑和遇到的问题解决-CSDN博客](https://blog.csdn.net/qq_23747281/article/details/111882086)
