---
title: 手写简易Netty_01——搭建项目
date: 2023-07-03 20:35:16
tags: 
 - Netty
 - Java
 - Tiny_Netty
categories:
 - [Tiny_Netty]
 - [Netty]
---



# 基本步骤 🚀

![image-20230622192508774](img/手写简易Netty-01/image-20230622192508774-16883882639981.png)

![image-20230622192533845](img/手写简易Netty-01/image-20230622192533845-16883882639992.png)

# 案例介绍和设计数据结构 💡

我们将编写一个服务器（server）和客户端（client），以支持网络通信，包括请求和响应。同时设计了两个接口，用于授权和保持连接的请求和响应。



![image-20230622193251995](img/手写简易Netty-01/image-20230622193251995.png)





消息体（body）使用 JSON 格式，表示操作（operation）的请求和响应。

消息头（header）中有多种操作类型（OP），需要定义不同的操作代码（OP_Code）。版本（version）用于表示版本号和处理兼容性。流ID（streamID）用作唯一标识符。

长度字段表示消息的长度，用于表示一个帧（frame）。



![image-20230622193446330](img/手写简易Netty-01/image-20230622193446330.png)



我们不深入业务逻辑，只关注思想和流程。

# 常见易错点 🔍

- `LengthFieldBasedFrameDecoder` 中未正确设置 `initialBytesToStrip`。如果未设置，解决粘包和半包问题时，仍然会携带长度字段。

- `ChannelHandler` 的顺序不正确：

  - 编写顺序应按照1~5进行编写，但执行顺序是 1 -> 4 -> 2 -> 3 -> 5。

  ```java
  1. pipeline.addLast(new OrderFrameDecoder());
  2. pipeline.addLast(new OrderFrameEncoder());
  3. pipeline.addLast(new OrderProtocolEncoder());
  4. pipeline.addLast(new OrderProtocolDecoder());
  5. pipeline.addLast(new OrderServerProcessHandler());
  ```

- `ChannelHandler` 共享与非共享问题（线程安全）：

  - 例如，`pipeline.addLast(new LoggingHandler(LogLevel.INFO));` 中的 `LoggingHandler` 是可以共享的，因此不需要在每个 `ChannelHandler` 中创建。但在多线程环境中，不应该共享的 `Handler` 却被共享了，就会出现线程问题。

- 分配 `ByteBuf`：应使用 `ByteBufAllocator.DEFAULT` 等分配器，而不是使用 `ChannelHandlerContext.alloc()`。在分配 `ByteBuf` 时，无论是堆外内存还是堆内内存，都可以使用。如果切换了实现方式，那么使用了 `ByteBufAllocator.DEFAULT` 的地方，实现方式不一致，就会出现问题。

- 未考虑释放 `ByteBuf`：

  - 即使不接收消息，仍然应调用 `release` 方法进行释放。

- 错误地认为 `ChannelHandlerContext.write(msg)` 就会将数据写出：

  - `write` 方法只是将消息添加到队列中。
  - 该方法仅在当前 `pipeline` 的位置寻找下一个符合条件的 `handler`。

- 误用 `ChannelHandlerContext.channel().writeAndFlush(msg)`：

  - 此消息发送会使整个 `pipeline` 重新执行一遍。如果在中间的处理器执行了该方法，将会从头开始执行，导致死循环。

