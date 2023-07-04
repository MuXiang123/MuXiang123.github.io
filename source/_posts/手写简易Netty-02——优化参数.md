---
title: 手写简易Netty_02——优化参数
date: 2023-07-04 10:04:02
tags: 
 - Netty
 - Java
 - Tiny_Netty
categories:
 - [Tiny_Netty]
 - [Netty]
---

# 优化参数

## Linux 系统参数

例如：/proc/sys/net/ipv4/tcp_keepalive_time

在进行 TCP 连接时，系统会为每个连接创建一个 socket 句柄，即文件句柄。然而，Linux 对每个进程所能打开的文件句柄数量有限制。当数量超出限制时，会出现 "Too many open file" 错误。

```bash
ulimit -n [xxx]
```

注意：ulimit 命令修改的数值仅对当前登录用户的当前使用环境有效。系统重启或用户退出后，该设置会失效。因此，可以将其作为程序启动脚本的一部分，在程序启动前执行。

## Netty 支持的系统参数

- 例如：serverBootstrap.option(ChannelOption.SO_BACKLOG, 1024); 使用 SO_ 开头
- SocketChannel -> .childOption
- ServerSocketChannel -> .option

讨论 Netty 支持的系统参数 (ChannelOption.[XXX])：

- 不考虑 UDP: IP_MULTICAST_TTL
- 不考虑 OIO 编程： 
  - ChannelOption SO_TIMEOUT = ("SO_TIMEOUT"); 
  - OIO 是阻塞编程，默认永远阻塞，因此需要控制阻塞时间

### SocketChannel 的 7 个参数

| Netty 系统相关参数 | 功能                                                         | 默认值                                                       |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| SO_SNDBUF          | TCP 数据发送缓冲区大小                                       | /proc/sys/net/ipv4/tcp_wmem: 4K[min, default, max] 动态调整  |
| SO_RCVBUF          | TCP 数据接受缓冲区大小                                       | /proc/sys/net/ipv4/tcp_rmem: 4K                              |
| SO_KEEPALIVE       | TCP 层 keepalive                                             | 默认关闭                                                     |
| SO_REUSEADDR       | 地址重用，解决 "Address already in use" 的常见场景：多网卡（IP）绑定相同端口；使关闭连接后的端口早些时候可供再次使用 | 默认不开启 澄清：不是让 TCP 绑定完全相同 IP + Port 来重复启动 |
| SO_LINGER          | 关闭 Socket 的延迟时间，默认禁用该功能，socket.close() 方法立即返回 | 默认不开启                                                   |
| IP_TOS             | 设置 IP 头部的 Type-of-Service 字段，用于描述 IP 包的优先级和 QoS 选项。例如倾向于延时还是吞吐量？ | 1000 - minimize delay<br>0100 - maximize throughput <br/>0010 - maximize reliability <br/>0001 - minimize monetary cost <br/>0000 - normal service （默认值） <br/>socket 选项的值是一个提示。实现可以忽略该值，或者忽略特定值 |
| TCP_NODELAY        | 设置是否启用 Nagle 算法：使用该算法将小的碎片数据连接成更大的报文以提高发送效率 | False 如果需要发送一些较小的报文，则需要禁用该算法           |

### ServerSocketChannel 的 3 个参数

| Netty 系统相关参数       | 功能                                                         | 默认值                                                       |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| SO_RCVBUF 接收数据缓冲区 | 为 Accept 创建的 socket channel 设置 SO_RCVBUF，"设置从此 ServerSocket 接受的套接字的 SO_RCVBUF 选项的默认建议值" | 为什么有 SO_RCVBUF 而没有 SO_SNDBUF？我们可以在设置 sndbuf 之后再发送数据，这段时间可以自行控制 |
| SO_REUSEADDR             | 是否可以重用端口                                             | false                                                        |
| SO_BACKLOG               | 最大的等待连接数量                                           | Netty 在 Linux 下获取的值（io.netty.util.NetUtil）：<br>1. 首先尝试：/proc/sys/net/core/somaxcon <br>2. 然后尝试：sysctl <br>3. 如果都没有取到，使用默认值：128 <br>使用方式： `javaChannel().bind(localAddress, config.getBacklog());` |

### 参数调整要点

- option/childOption 区分不清：不会报错，但不会生效。
- 不了解的参数不要随意修改，避免过早优化。
- 最好可配置（动态配置更佳）。

### 需要调整的参数

最大打开文件数 • TCP_NODELAY • SO_BACKLOG • SO_REUSEADDR（视情况处理）

将 SO_BACKLOG 的默认值 128 调整为 1024

### 具体调整

```java
Server.java
// 系统参数
serverBootstrap.childOption(NioChannelOption.SO_KEEPALIVE, true);
serverBootstrap.option(NioChannelOption.SO_BACKLOG, 1024);
```

## Netty核心参数

ChannelOption

- childOption(ChannelOption.[XXX], [YYY])
- option(ChannelOption.[XXX], [YYY])

System property

- Dio.netty.[XXX] = [YYY]

### ChannelOption（非系统相关：共11个）

| Netty 系统相关参数              | 功能                                                         | 默认值                                                       |
| ------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| WRITE_BUFFER_WATER_MARK         | 高低水位线、间接防止写数据 OOM                               | 32k → 64k 😮，为什么不大？每个连接都64k的话，那就太大了。     |
| CONNECT_TIMEOUT_MILLIS          | 客户端连接服务器最大允许时间                                 | 30s ⏱️                                                        |
| MAX_MESSAGES_PER_READ           | 最大允许“连续”读次数                                         | 16次                                                         |
| WRITE_SPIN_COUNT                | 最大允许“连读”写次数                                         | 16次                                                         |
| ALLOCATOR ByteBuf               | 分配器                                                       | ByteBufAllocator.DEFAULT：大多池化、堆外                     |
| RCVBUF_ALLOCATOR                | 数据接收 ByteBuf 分配大小计算器（计算是否达到水位线） + 读次数控制器 | AdaptiveRecvByteBufAllocator                                 |
| AUTO_READ                       | 是否监听“读事件”                                             | 默认：监听“读”事件： 设置此标记的方法也触发注册或移除读事件的监听 |
| AUTO_CLOSE                      | “写数据”失败，是否关闭连接                                   | 默认打开，因为不关闭，下次还写，还是失败怎么办！             |
| MESSAGE_SIZE_ESTIMATOR          | 数据（ByteBuf、FileRegion等）大小计算器                      | DefaultMessageSizeEstimator.DEFAULT 例如计算 ByteBuf: byteBuf.readableBytes() |
| SINGLE_EVENTEXECUTOR_PER _GROUP | 当增加一个 handler 且指定 EventExecutorGroup 时：决定这个 handler 是否只用 EventExecutorGroup 中的一个固定的 EventExecutor （取决于next()实现） | 默认：true: 这个 handler 不管是否共享，绑定上唯一一个 event executor.所以小名“pinEventExecutor” 没有指定 EventExecutorGroup，复用 channel 的 NioEventLoop |
| ALLOW_HALF_CLOSURE              | 关闭连接时，允许半关。 https://issues.jboss.org/browse/NETTY-236 | 默认：不允许半关，如果允许，处理变成： shutdownInput(); pipeline.fireUserEventTriggered(ChannelInputShutd ownEvent.INSTANCE); |

### ALLOCATOR 与 RCVBUF_ALLOCATOR

**功能关联：**

- ALLOCATOR 负责 ByteBuf 怎么分配（例如：从哪里分配），
- RCVBUF_ALLOCATOR 负责计算为接收数据接分配多少 ByteBuf： 例如，AdaptiveRecvByteBufAllocator 有两大功能：
  - 动态计算下一次分配 bytebuf 的大小：guess()；
  - 判断是否可以继续读：continueReading()

**代码关联：**

```java
io.netty.channel.AdaptiveRecvByteBufAllocator.HandleImpl handle = AdaptiveRecvByteBufAllocator.newHandle(); 

ByteBuf byteBuf = handle.allocate(ByteBufAllocator)
```

其中： allocate的实现：

```java
ByteBuf allocate(ByteBufAllocator alloc) { return alloc.ioBuffer(guess()); }
```

### 在TinyNetty中调整

```java
Client.java
//调整连接超时
bootstrap.option(NioChannelOption.CONNECT_TIMEOUT_MILLIS, 10 * 1000);
```

## 三个让费脑的参数

大多数情况下都可以不予考虑

### 地址重用：SO_REUSEADDR 🔄

`SO_REUSEADDR`是一个套接字选项，用于控制套接字地址的重用。在TCP连接关闭后，操作系统会在一段时间内保持该套接字地址（IP地址+端口号）的占用状态，以确保任何延迟到达的数据能够正确传递到应用程序。

在TCP连接关闭流程中，客户端向服务器发送ack。此时，客户端会等待2MSL，以确保服务器正确接收到该ack。如果服务器未收到ack，则会发送超时重传的FIN消息，客户端将再次发送ack。如果服务器接收到了ack，则不会发送任何消息。

![image-20230624160306498](img/手写简易Netty-02——优化参数/image-20230624160306498.png)



因此，客户端需要等待的时间为：

**去向ack消息最大存活时间（MSL）+ 来向FIN消息最大存活时间（MSL） = 2MSL**

通常为60秒，但在网络质量较好的情况下，可以缩短到1秒。相信1秒的TIME_WAIT足够了。

当你将 `SO_REUSEADDR` 选项设置为 `true` 时，表示允许在使用过程中关闭的套接字被重新绑定到同一个地址。这样，你就可以立即重新绑定相同的地址，而无需等待保持期结束。这在某些特定的应用场景中非常有用，比如服务器需要快速重启或频繁绑定相同的地址。

```java
serverBootstrap.option(ChannelOption.SO_REUSEADDR, true);
```

### SO_LINGER：关闭时间 🌅

`SO_LINGER`定义了关闭套接字时是否立即关闭或等待一段时间。具体而言，`SO_LINGER`选项控制操作系统在关闭套接字时的行为。

默认情况下，当关闭套接字时，操作系统会尝试发送任何待发送的数据，并立即关闭套接字。这意味着，如果还有未发送的数据，它们可能会丢失。

通过设置`SO_LINGER`选项，你可以改变这种默认行为。当将`SO_LINGER`设置为非零的超时值时，关闭套接字的行为将发生变化。具体而言，当你关闭套接字时，操作系统会尝试发送所有待发送的数据，并等待指定的超时时间。

需要注意的是，`SO_LINGER`选项可能会影响行为和延迟。如果存在未发送完的数据，等待超时时间可能会导致套接字关闭的延迟。因此，在使用`SO_LINGER`选项时，你需要根据自己的需求仔细评估和权衡。

另外，将`timeout`设置为0表示关闭套接字时立即丢弃未发送的数据，并立即关闭套接字。

![image-20230624161738285](img/手写简易Netty-02——优化参数/image-20230624161738285.png)

#### 如何判断是否需要设置？

1. 重要数据的完整性
2. 如果你的应用程序需要在关闭套接字之前执行特定的清理操作或发送最后的消息，那么设置`SO_LINGER`可以提供可控的关闭行为。
3. 延迟和性能：需要注意的是，设置`SO_LINGER`可能会导致套接字关闭的延迟。如果你的应用程序对延迟非常敏感，或需要快速释放资源而快速关闭套接字，那么设置`SO_LINGER`可能不是一个好的选择。

### ALLOW_HALF_CLOSURE：半关闭 🤝

`ALLOW_HALF_CLOSURE`是Netty中的一个选项，用于控制是否允许半关闭（Half Closure）操作。半关闭是指在一个TCP连接中，一方关闭了输出流（发送方），但保持输入流（接收方）仍然打开。

默认情况下，Netty不允许半关闭操作。这意味着，当一方关闭输出流时，另一方将接收到连接关闭的事件，从而关闭整个连接。这种行为符合TCP协议的规范，因为一方关闭输出流意味着不再发送数据，连接可以被认为是完全关闭了。

然而，在某些情况下，你可能希望允许半关闭操作。例如，如果你的应用程序中的一方关闭了输出流，但仍希望接收另一方发送的数据，那么你可以使用`ALLOW_HALF_CLOSURE`选项。

需要注意的是，在启用半关闭后，在处理连接关闭事件时要特别小心。你可能需要检查连接的状态，以确定是完全关闭连接还是仅关闭输出流。

![image-20230624161951034](img/手写简易Netty-02——优化参数/image-20230624161951034.png)

```java
channel.pipeline().option(ChannelOption.ALLOW_HALF_CLOSURE, true);
```

