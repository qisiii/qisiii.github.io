+++
title = 'Java安全机制之一——SecurityManager和AccessController'
date = 2023-11-08
draft = false
tags = ['java']
categories = ['tech'] 
slug = "java_security"
+++

## 前言：

在看socket相关代码的时候，AbstractPlainSocketImpl中的一段代码吸引了我，其实之前见过很多次类似的代码，但一直不想去看，只知道肯定和权限什么的相关，这次既然又碰到了就研究一下，毕竟也不能对java基本代码一无所知。

```
static {
    java.security.AccessController.doPrivileged(
        new java.security.PrivilegedAction<Void>() {
            public Void run() {
                System.loadLibrary("net");
                return null;
            }
        });
}
```

## 一些概念:

在jdk1.0的时代，applet依然是前端的一种可用的技术方案，比如可以嵌入在网页里运行。那个时候jdk的设计者们认为本地代码是安全的、远端代码是有风险的，而applet就是属于远端代码。因此，为了保证用户主机的安全和隐私，设计者参考了沙箱的思想，依托于当时jdk的体量很小，使用SecurityManager来分隔本地代码和远程代码，一个有权限，一个没有权限。

当时还出现了签名相关的机制(本文不关心，所以没做了解），随着java发展，1.1的时候出现了JAVABEAN、JDBC、反射等新概念，于是有了更多的新权限。设计者发现完全授予本地代码所有权限变得不合理，在1.2的时候重构了SecurityManager，变成了现在这样以最小粒度控制权限。这个时候的SecurityManager有两个功能，一是防御远程代码、二是防御本地代码的漏洞。

不知道是什么时候起，安全机制引入了域（ProtectDomain）的概念，也可以视作将一个大沙箱拆分为多个小沙箱。一个域对应一个沙箱，不同的代码（Codesource）被划分到不同域中，不同的域有着不同的权限（Permission），就像下图一样。同时可以给不同的域配置不同的权限，静态和动态均可，这个配置被称为策略(Policy)。
![](http://picgo.qisiii.asia/post/20240729f745258652dc174547b669498d053847.png)

**注意！**

> 在JDK20和JDK21的security-guide中都提到了，和SecurityManager与之相关的api已被弃用，并将在未来的版本中删除。SecurityManager没有替代者。有关讨论和备选方案，请参阅[JEP 411: Deprecate the Security Manager for Removal。](https://openjdk.org/jeps/411 "JEP 411: Deprecate the Security Manager for Removal。")

## AccessController

AccessController主要有两个功能，对应的核心方法也是两类

### checkPermission(校验是否存在权限)

```
public static void checkPermission(Permission perm)
    throws AccessControlException
{
    AccessControlContext stack = getStackAccessControlContext();
    // if context is null, we had privileged system code on the stack.
    //...其他获取context方法
    AccessControlContext acc = stack.optimize();
    acc.checkPermission(perm);
}
```

调用该方法时，一般会new一个期望的权限，然后作为入参传入checkPermission方法。

```
FilePermission perm = new FilePermission("C:\\Users\\Administrator\\Desktop\\liveController.txt", "read");
AccessController.checkPermission(perm);
```

**注意，校验权限的时候会校验调用链路径上所有类的权限；假如调用链是从i开始，一直调用到m，校验逻辑如下**

```
 for (int i = m; i > 0; i--) {
     if (caller i's domain does not have the permission)
         throw AccessControlException
     else if (caller i is marked as privileged) {
         if (a context was specified in the call to doPrivileged)
             context.checkPermission(permission)
         if (limited permissions were specified in the call to doPrivileged) {
             for (each limited permission) {
                 if (the limited permission implies the requested permission)
                     return;
             }
         } else
             return;
     }
 }
```

 代码执行的时候，每一次方法的调用都代表着一次入栈，而权限校验的时候则正好是从栈顶开始，依次判断每个栈帧是否具有权限，一直到栈底。
![](http://picgo.qisiii.asia/post/2024072941a244f2fb0cc49d443daef8016f2140.png)

### doPrivileged(临时授权)

```
public static native <T> T doPrivileged(PrivilegedAction<T> action);
```

这个方法的功能是将当前类所拥有的权限，**能且仅能**临时赋予其上游调用方。

在这个场景下，必然存在多个域，且只有某些域拥有权限A，但是其他域并没有这个权限。在java语言中很容易出现这个情况，比如我们调用一些第三方jar包的方法，三方jar包还能调用别的三方jar包，这种场景很有可能只有最底层的方法所对应的域拥有权限。此时为了方法的成功，就可以使用该方法。

使用的时候就是将代码逻辑放入AccessController.doPrivileged中即可，如下述代码一般。

```
//项目B，会打成security-demo.jar
public class PermissionDemo {
    /**
     * 使用特权访问机制
     * @param file
     */
    public void runWithOutPermission(String file){
        AccessController.doPrivileged((PrivilegedAction<String>) () -> {
            //hutool的FileUtil
            String s = FileUtil.readString(file, "utf-8");
            System.out.println(s);
            return s;
        });
    }
}
//项目A，引入security-demo.jar
public class Aperson {
    public static void main(String[] args) {
        new PermissionDemo().runWithOutPermission("C:\\Users\\Administrator\\Desktop\\test.txt");
    }
}
```

这里需要注意的是，AccessController.doPrivileged所在的当前类也需要拥有权限。以这个例子为例，文件读写是在hutool的FileUtil中执行，hutool对应的是域C；PermissionDemo对应的是域B，且会将自身权限向上传递；而Aperson对应的是域A。这个例子中，想要Aperson执行成功，必须是域C和域B都拥有test.txt的read权限。

对应的policy如下

```
grant codeBase  "file:/C:/Users/Administrator/.m2/repository/cn/hutool/hutool-all/5.7.11/-"{
    permission java.io.FilePermission "C:\\Users\\Administrator\\Desktop\\*", "read";
};
grant codeBase  "file:/C:/Users/Administrator/.m2/repository/xxx/xxx/security-demo/-"{
    permission java.io.FilePermission "C:\\Users\\Administrator\\Desktop\\*", "read";
};
```

从栈帧的角度来看的话，判断到doPrivilege对应的那层之后，校验就直接返回了，不校验下面层是否存在权限。
![](http://picgo.qisiii.asia/post/20240729e6ac0c020bf517a6047fcaaf00888f5d.png)

## ProtectDomain

protectDomain类由codeSource和permission构成
![](http://picgo.qisiii.asia/post/20240729ce4b8b3a012225b7ad7e9bbf0612ca99.png)

![](http://picgo.qisiii.asia/post/20240729ed7a6012769be9a09f74a0adda74bea7.png)

## CodeSource

类的来源，一般为jar包路径或者classpath路径（target/classes）

因为所有类在通过ClassLoader引入的，所以ClassLoader知道类的基本信息，在defineClass时，将CodeSource和Permission进行了绑定。同理，由于类必须通过ClassLoader加载，对于使用自定义ClassLoader加载的类，就只有那个类加载器知道对应的CodeSource和permission。因此，不同的类加载器本身就属于不同的域。

## Permission

Java抽象出的顶层的类，核心方法是implies，该方法用来判断当前线程是否隐含指定权限，由各自的子类实现。子类实现过多，这里就不列举了。

**PermissionCollection**本质是个list，里面是某一类权限的多个实例，比如文件夹A-读权限，文件夹B-写权限，文件夹C-读写权限。

**Permissions**核心是一个map,key是Permissoin子类,value是PermissionCollection

## SecurityManager

SecurityManage里有一堆check方法，调用的是AccessController.checkPermission方法，入参就是Permission各个子类的实例化。

### 开启方式：

隐性：启动时添加-Djava.security.manager

显性：System.setSecurityManager

```
public class NoShowTest {
     static class CustomManager extends SecurityManager{
         @Override
         public void checkRead(String file) {
             throw new AccessControlException("无权限访问");
         }
     }
    public static void main(String[] args) {
       System.setSecurityManager(new CustomManager());
       System.getSecurityManager().checkRead("C:\\Users\\Administrator\\Desktop\\liveController.txt");
    }
}
```

## Policy

启动时通过 -Djava.security.policy=xxxx\custom.policy，如果没有指定，则默认使用jdk路径下\jre\lib\security\java.policy

# 参考：

[Java安全:SecurityManager与AccessController - 掘金](https://juejin.cn/post/6844903657775824910#heading-1)

[Java沙箱机制的实现——安全管理器、访问控制器 - 掘金](https://juejin.cn/post/6844904150321332232?from=search-suggest#heading-15)

[第21章-再谈类的加载器](https://zhuanlan.zhihu.com/p/268637283)

https://openjdk.org/jeps/411

https://docs.oracle.com/en/java/javase/20/security/java-security-overview1.html#GUID-BBEC2DC8-BA00-42B1-B52A-A49488FCF8FE

[AccessController.doPrivileged - 山河已无恙 - 博客园](https://www.cnblogs.com/liruilong/p/14810513.html)
