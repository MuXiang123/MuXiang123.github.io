---
title: netty入门
date: 2023-06-21 20:32:47
tags:
 - Netty
 - java
categories:
 - Netty
comments: true
---

## 1、简述

netty是一个异步事件驱动的网络应用程序框架，用于快速开发可维护的高性能协议服务器和客户端

[ netty.io](https://netty.io/)

## 2、为什么使用netty

### NIO缺点

- api繁杂，
- 需要熟悉java多线程，网络编程
- epoll bug会导致selector空轮询，最终导致cpu 100%

### netty优点

- api简单
- 支持多种协议
- 性能高
- 社区活跃

## 3、架构

![image-20230306195432035](img/netty入门/img-20230306195432035.png)

1. **绿色 core**：零拷贝、api库、可扩展事件模型
2. **黄色 protocol Support协议** 包括http、websocket ssl等等
3. **红色 传输服务** socket http tunnel等等

## 4、hello word

<img src="https://pic4.zhimg.com/80/v2-7eefba893a65706eb6bbe4115cbd0b83_720w.webp" alt="img" style="zoom: 80%;" />

根据上面图上的模型进行编写hello word，

### 4.1 创建服务端启动类

```java
public class MyServer {
    public static void main(String[] args) throws Exception {
        //创建两个线程组 boosGroup、workerGroup
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            //创建服务端的启动对象，设置参数
            ServerBootstrap bootstrap = new ServerBootstrap();
            //设置两个线程组boosGroup和workerGroup
            bootstrap.group(bossGroup, workerGroup)
                //设置服务端通道实现类型    
                .channel(NioServerSocketChannel.class)
                //设置线程队列得到连接个数    
                .option(ChannelOption.SO_BACKLOG, 128)
                //设置保持活动连接状态    
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                //使用匿名内部类的形式初始化通道对象    
                .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            //给pipeline管道设置处理器
                            socketChannel.pipeline().addLast(new MyServerHandler());
                        }
                    });//给workerGroup的EventLoop对应的管道设置处理器
            System.out.println("java技术爱好者的服务端已经准备就绪...");
            //绑定端口号，启动服务端
            ChannelFuture channelFuture = bootstrap.bind(6666).sync();
            //对关闭通道进行监听
            channelFuture.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

### 4.2 创建服务端处理器

```java
/**
 * 自定义的Handler需要继承Netty规定好的HandlerAdapter
 * 才能被Netty框架所关联，有点类似SpringMVC的适配器模式
 **/
public class MyServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //获取客户端发送过来的消息
        ByteBuf byteBuf = (ByteBuf) msg;
        System.out.println("收到客户端" + ctx.channel().remoteAddress() + "发送的消息：" + byteBuf.toString(CharsetUtil.UTF_8));
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        //发送消息给客户端
        ctx.writeAndFlush(Unpooled.copiedBuffer("服务端已收到消息，并给你发送一个问号?", CharsetUtil.UTF_8));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        //发生异常，关闭通道
        ctx.close();
    }
}
```

### 4.3创建客户端启动类

```java
public class MyClient {

    public static void main(String[] args) throws Exception {
        NioEventLoopGroup eventExecutors = new NioEventLoopGroup();
        try {
            //创建bootstrap对象，配置参数
            Bootstrap bootstrap = new Bootstrap();
            //设置线程组
            bootstrap.group(eventExecutors)
                //设置客户端的通道实现类型    
                .channel(NioSocketChannel.class)
                //使用匿名内部类初始化通道
                .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            //添加客户端通道的处理器
                            ch.pipeline().addLast(new MyClientHandler());
                        }
                    });
            System.out.println("客户端准备就绪，随时可以起飞~");
            //连接服务端
            ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 6666).sync();
            //对通道关闭进行监听
            channelFuture.channel().closeFuture().sync();
        } finally {
            //关闭线程组
            eventExecutors.shutdownGracefully();
        }
    }
}
```

### 4.4创建客户端处理器

```java
public class MyClientHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        //发送消息到服务端
        ctx.writeAndFlush(Unpooled.copiedBuffer("歪比巴卜~茉莉~Are you good~马来西亚~", CharsetUtil.UTF_8));
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //接收服务端发送过来的消息
        ByteBuf byteBuf = (ByteBuf) msg;
        System.out.println("收到服务端" + ctx.channel().remoteAddress() + "的消息：" + byteBuf.toString(CharsetUtil.UTF_8));
    }
}
```

### 4.5测试

先运行server再运行client

MyServer打印结果:

![img](https://pic1.zhimg.com/80/v2-aa144d6ad2688f69b0f5ef7dc916a3f4_720w.webp)



MyClient打印结果：

![img](https://pic4.zhimg.com/80/v2-e6bc4dec6eecb3ae30f55c7a6487e1f7_720w.webp)

## 5、netty的特性和重要组件

### 5.1 taskQueue任务队列

如果handler出来又一些长时间的业务处理，那么可以交给taskQueue异步处理。

```java
public class MyServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //获取到线程池eventLoop，添加线程，执行
        ctx.channel().eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                try {
                    //长时间操作，不至于长时间的业务操作导致Handler阻塞
                    Thread.sleep(1000);
                    System.out.println("长时间的业务处理");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```

### 5.2 schedduleTaskQueue延时任务队列

和任务队列相似，不同的是可以延迟一定时间再执行设置

```java
ctx.channel().eventLoop().schedule(new Runnable() {
    @Override
    public void run() {
        try {
            //长时间操作，不至于长时间的业务操作导致Handler阻塞
            Thread.sleep(1000);
            System.out.println("长时间的业务处理");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
},5, TimeUnit.SECONDS);//5秒后执行
```

### 5.3 Future异步机制

在搭建HelloWord工程的时候，我们看到有一行这样的代码：

```text
ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 6666);
```

很多操作都返回这个ChannelFuture对象，究竟这个ChannelFuture对象是用来做什么的呢？

ChannelFuture提供操作完成时一种异步通知的方式。一般在Socket编程中，等待响应结果都是同步阻塞的，而Netty则不会造成阻塞，因为ChannelFuture是采取类似观察者模式的形式进行获取结果。请看一段代码演示：

### 5.4 BootStrap 和ServerBootStrap

Bootstrap和ServerBootStrap是Netty提供的一个创建客户端和服务端启动器的工厂类，使用这个工厂类非常便利地创建启动类，根据上面的一些例子，其实也看得出来能大大地减少了开发的难度。首先看一个类图：

![img](https://pic1.zhimg.com/80/v2-dab6b780993979fcb86ef14553c16720_720w.webp)

可以看出都是继承于AbstractBootStrap抽象类，所以大致上的配置方法都相同。

一般来说，使用Bootstrap创建启动器的步骤可分为以下几步：

![img](https://pic4.zhimg.com/80/v2-dd3a866c356ee7bd24d23319d08116ef_720w.webp)

#### 1） group()

服务端要使用两个线程组：

- bossGroup 用于监听客户端连接，专门负责与客户端创建连接，并且把连接注册到workerGroup的Selector中。
- workerGroup 处理每一个连接发生的读写时间。

**线程组中线程数是多少？**

```java
//使用一个常量保存
    private static final int DEFAULT_EVENT_LOOP_THREADS;

    static {
        //NettyRuntime.availableProcessors() * 2，cpu核数的两倍赋值给常量
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
        }
    }
    
    protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
        //如果不传入，则使用常量的值，也就是cpu核数的两倍
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
    }
```

#### 2）channel()

通道类型有：

**NioSocketChannel**： 异步非阻塞的客户端 TCP Socket 连接。

**NioServerSocketChannel**： 异步非阻塞的服务器端 TCP Socket 连接。

> 常用的就是这两个通道类型，因为是异步非阻塞的。所以是首选。

**OioSocketChannel：** 同步阻塞的客户端 TCP Socket 连接。

**OioServerSocketChannel：** 同步阻塞的服务器端 TCP Socket 连接。

**NioSctpChannel：** 异步的客户端 Sctp（Stream Control Transmission Protocol，流控制传输协议）连接。

**NioSctpServerChannel**： 异步的 Sctp 服务器端连接。

#### 3）option() childOption()

- option设置的是服务端用于接收进来的连接，也就是bossGroup
- childOption是提供给父管道接收的连接，workerGroup

SocketChannel参数，也就是childOption()常用的参数：

> **SO_RCVBUF** Socket参数，TCP数据接收缓冲区大小。
> **TCP_NODELAY** TCP参数，立即发送数据，默认值为Ture。
> **SO_KEEPALIVE** Socket参数，连接保活，默认值为False。启用该功能时，TCP会主动探测空闲连接的有效性。

ServerSocketChannel参数，也就是option()常用参数：

> **SO_BACKLOG** Socket参数，服务端接受连接的队列长度，如果队列已满，客户端连接将被拒绝。默认值，Windows为200，其他为128。

#### 4）设置流水线（重点）

ChannelPipeline是Netty处理请求的责任链，ChannelHandler则是具体处理请求的处理器。实际上每一个channel都有一个处理器的流水线。

在Bootstrap中childHandler()方法需要初始化通道，实例化一个ChannelInitializer，这时候需要重写initChannel()初始化通道的方法，装配流水线就是在这个地方进行。代码演示如下：

```java
//使用匿名内部类的形式初始化通道对象
bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel socketChannel) throws Exception {
        //给pipeline管道设置自定义的处理器
        socketChannel.pipeline().addLast(new MyServerHandler());
    }
});
```

处理器Handler主要分为两种：

> ChannelInboundHandlerAdapter(入站处理器)、ChannelOutboundHandler(出站处理器)

入站指的是数据从底层java NIO Channel到Netty的Channel。

出站指的是通过Netty的Channel来操作底层的java NIO Channel。

**ChannelInboundHandlerAdapter处理器常用的事件有**：

1. 注册事件 fireChannelRegistered。
2. 连接建立事件 fireChannelActive。
3. 读事件和读完成事件 fireChannelRead、fireChannelReadComplete。
4. 异常通知事件 fireExceptionCaught。
5. 用户自定义事件 fireUserEventTriggered。
6. Channel 可写状态变化事件 fireChannelWritabilityChanged。
7. 连接关闭事件 fireChannelInactive。

**ChannelOutboundHandler处理器常用的事件有**：

1. 端口绑定 bind。
2. 连接服务端 connect。
3. 写事件 write。
4. 刷新时间 flush。
5. 读事件 read。
6. 主动断开连接 disconnect。
7. 关闭 channel 事件 close。

#### 5） bind()

提供用于服务端或者客户端绑定服务器地址和端口号，默认是异步启动。如果加上sync()方法则是同步

### 5、 channel

channel为用户提供：

1. 通道当前的状态（例如它是打开？还是已连接？）
2. channel的配置参数（例如接收缓冲区的大小）
3. channel支持的IO操作（例如读、写、连接和绑定），以及处理与channel相关联的所有IO事件和请求的ChannelPipeline。

#### 1）获取状态

```java
boolean isOpen(); //如果通道打开，则返回true
boolean isRegistered();//如果通道注册到EventLoop，则返回true
boolean isActive();//如果通道处于活动状态并且已连接，则返回true
boolean isWritable();//当且仅当I/O线程将立即执行请求的写入操作时，返回true。
```

#### 2）获取配置

获取单条配置信息，使用getOption()，代码演示：

```java
ChannelConfig config = channel.config();//获取配置参数
//获取ChannelOption.SO_BACKLOG参数,
Integer soBackLogConfig = config.getOption(ChannelOption.SO_BACKLOG);
//因为我启动器配置的是128，所以我这里获取的soBackLogConfig=128
```

获取多条配置信息，使用getOptions()，代码演示：

```java
ChannelConfig config = channel.config();
Map<ChannelOption<?>, Object> options = config.getOptions();
for (Map.Entry<ChannelOption<?>, Object> entry : options.entrySet()) {
    System.out.println(entry.getKey() + " : " + entry.getValue());
}
```

#### 3）io操作

写操作 服务端到客户端

```text
ctx.channel().writeAndFlush(Unpooled.copiedBuffer("这波啊，这波是肉蛋葱鸡~", CharsetUtil.UTF_8));
```

连接

```text
ChannelFuture connect = channelFuture.channel().connect(new InetSocketAddress("127.0.0.1", 6666));//一般使用启动器，这种方式不常用
```

**通过channel获取ChannelPipeline**，并做相关的处理：

```text
//获取ChannelPipeline对象
ChannelPipeline pipeline = ctx.channel().pipeline();
//往pipeline中添加ChannelHandler处理器，装配流水线
pipeline.addLast(new MyServerHandler());
```

### 6、Selector

在NioEventLoop中，有一个成员变量selector，这是nio包的Selector，在之前[《NIO入门》](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s/GfV9w2B0mbT7PmeBS45xLw)中，我已经讲过Selector了。

Netty中的Selector也和NIO的Selector是一样的，就是用于监听事件，管理注册到Selector中的channel，实现多路复用器。

### 7、PiPeline与ChannelPipeline

我们知道可以在channel中装配ChannelHandler流水线处理器，那一个channel不可能只有一个channelHandler处理器，肯定是有很多的，既然是很多channelHandler在一个流水线工作，肯定是有顺序的。

![image-20230307193519223](img/netty入门/img-20230307193519223.png)

于是pipeline就出现了，pipeline相当于处理器的容器。初始化channel时，把channelHandler按顺序装在pipeline中，就可以实现按序执行channelHandler了。

![image-20230307193528870](img/netty入门/img-20230307193528870.png)

在一个Channel中，只有一个ChannelPipeline。该pipeline在Channel被创建的时候创建。ChannelPipeline包含了一个ChannelHander形成的列表，且所有ChannelHandler都会注册到ChannelPipeline中。

### 8、ChannelHandlerContext

在Netty中，Handler处理器是有我们定义的，上面讲过通过集成入站处理器或者出站处理器实现。这时如果我们想在Handler中获取pipeline对象，或者channel对象，怎么获取呢。

于是Netty设计了这个ChannelHandlerContext上下文对象，就可以拿到channel、pipeline等对象，就可以进行读写等操作。

![image-20230307193637688](img/netty入门/img-20230307193637688.png)

通过类图，ChannelHandlerContext是一个接口，下面有三个实现类。

![image-20230307193702314](img/netty入门/img-20230307193702314.png)

### 9、EventLoopGroup

![image-20230307193734269](img/netty入门/img-20230307193734269.png)

其中包括了常用的实现类NioEventLoopGroup。OioEventLoopGroup在前面的例子中也有使用过。

从Netty的架构图中，可以知道服务器是需要两个线程组进行配合工作的，而这个线程组的接口就是EventLoopGroup。

每个EventLoopGroup里包括一个或多个EventLoop，每个EventLoop中维护一个Selector实例。

#### 1）轮询机制实现原理

如果线程数是2的n次方，则采用这种算法。通过算发实现轮询。

index++ 对数组长度取模

```text
private final AtomicInteger idx = new AtomicInteger();
private final EventExecutor[] executors;

@Override
public EventExecutor next() {
    //idx.getAndIncrement()相当于idx++，然后对任务长度取模
    return executors[idx.getAndIncrement() & executors.length - 1];
}
```
