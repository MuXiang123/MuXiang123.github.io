---
title: 手写简易Netty_03——跟踪诊断
date: 2023-07-04 10:05:30
tags: 
 - Netty
 - Java
 - Tiny_Netty
categories:
 - [Tiny_Netty]
 - [Netty]
---

# 跟踪诊断 🕵️‍♂️

## 完善线程名 💡

在dubug模式中能够看见线程后面的数字，数字分别代表的意思是：

- 2-1 boss group

- 3-1 worker group

为了让我们在调试的时候更清晰的了解到具体是哪个线程，在serve中，添加如下代码：

```java
codeNioEventLoopGroup boss = new NioEventLoopGroup(0, new DefaultThreadFactory("boss"));
NioEventLoopGroup worker = new NioEventLoopGroup(0, new DefaultThreadFactory("worker"));
serverBootstrap.group(boss, worker);
```

打印的日志中便会出现线程名称。

## 完善Handler名称 🛠️

在debug模式下，能够看见handler的名称是默认使用类名，并且后面附带的数字意义是：

- $1 匿名内部类

- #0 防止一个pipline中加入了多个handle

通过给handler取名

```java
pipeline.addLast("frameDecoder", new OrderFrameDecoder());
```

如果有名称就不需要$1 #0 之类的了。

## netty日志的原理及使用 📚

### netty日志框架原理 🤔

通过对不同条件的判断可以使用SLF4J、Log4J、Log4J2、Jdk的log.

```java
codeprivate static InternalLoggerFactory newDefaultFactory(String name) {
    Object f;
    try {
        f = new Slf4JLoggerFactory(true);
        ((InternalLoggerFactory)f).newInstance(name).debug("Using SLF4J as the default logging framework");
    } catch (Throwable var7) {
        try {
            f = Log4JLoggerFactory.INSTANCE;
            ((InternalLoggerFactory)f).newInstance(name).debug("Using Log4J as the default logging framework");
        } catch (Throwable var6) {
            try {
                f = Log4J2LoggerFactory.INSTANCE;
                ((InternalLoggerFactory)f).newInstance(name).debug("Using Log4J2 as the default logging framework");
            } catch (Throwable var5) {
                f = JdkLoggerFactory.INSTANCE;
                ((InternalLoggerFactory)f).newInstance(name).debug("Using java.util.logging as the default logging framework");
            }
        }
    }

    return (InternalLoggerFactory)f;
}
```

netty将所有log的依赖都增加进去了，但是在程序中还是看不见Slf4J是为什么？

因为在pom里面，使用了一个Optional参数标识了。导致了虽然项目依赖netty，但是项目中没有将slf4J下载下来。

### 修改JDK looger日志级别 ⚙️

JDK自带的日志系统中的每一个日志记录器都可以有7个日志级别（LogLevel），从高级到低级，它们分别是：SEVERE、WARNING、INFO、CONFIG、FINE、FINER、FINEST。

进入JRE -> lib 找到 logging.properties 文件

修改下面两行即可

```properties
#日志级别
.level= INFO
#控制台
java.util.logging.ConsoleHandler.level = INFO
```

### 使用slf4j + log4j示例 🔌

在项目的pom文件中添加依赖即可，如果是添加log4j，那么需要添加相应的配置文件。

### 衡量好logging handler的位置和级别 ⚖️

```java
pipeline.addLast(new LoggingHandler(LogLevel.INFO));
```

核心是设置好级别 LogLevel的级别

通过设置LogginHandler的级别，可以打印出不同阶段的日志。

可以在childHandler方法内，按照需求设置相应的log
