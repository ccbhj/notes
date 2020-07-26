# Netty

1. 定义:

   1. Netty 是一个 **基于 NIO** 的 client-server(客户端服务器)框架，使用它可以快速简单地开发网络应用程序。
   2. 它极大地简化并优化了 TCP 和 UDP 套接字服务器等网络编程,并且性能以及安全性等很多方面甚至都要更好。
   3. 支持多种应用层协议 如 FTP，SMTP，HTTP 以及各种二进制和基于文本的传统协议。

2. 优点:

   + 统一的 API，支持多种传输类型，阻塞和非阻塞的。
   + 简单而强大的线程模型。
   + 自带编解码器解决 TCP 粘包/拆包问题。
   + 自带各种协议栈。
   + 真正的无连接数据包套接字支持。
   + 比直接使用 Java 核心 API 有更高的吞吐量、更低的延迟、更低的资源消耗和更少的内存复制。
   + 安全性不错，有完整的 SSL/TLS 以及 StartTLS 支持。
   + 社区活跃
   + 成熟稳定，经历了大型项目的使用和考验，而且很多开源项目都使用到了 Netty， 比如我们经常接触的 Dubbo、RocketMQ 等等。

3. 应用场景:

   1. **作为 RPC 框架的网络通信工具** ：不同服务节点之间经常需要相互调用
   2. **实现一个自己的 HTTP 服务器** ：通过 Netty 我们可以自己实现一个简单的 HTTP 服务器.
   3. **实现一个即时通讯系统** ：使用 Netty 我们可以实现一个可以聊天类似微信的即时通讯系统，
   4. **实现消息推送系统** 

4. 基本组件:

   1. channel

   **Channel** 接口是 **Netty** 对网络操作抽象类，它除了包括基本的 **I/O** 操作，如 **bind()**、**connect()**、**read()**、**write()** 等。

   比较常用的**Channel**接口实现类是**NioServerSocketChannel**（服务端）和**NioSocketChannel**（客户端），这两个 **Channel** 可以和 **BIO** 编程模型中的**ServerSocket**以及**Socket**两个概念对应上。

   2. eventloop

      **EventLoop 的主要作用实际就是负责监听网络事件并调用事件处理器进行相关 I/O 操作的处理。**

   3. channel future:

      **Netty** 是异步非阻塞的，所有的 I/O 操作都为异步的。可以用addListener()注册回调

      我们还可以通过 **ChannelFuture** 接口的 **sync()**方法让异步的操作变成同步的。

   4. ChannelHandler 和 ChannelPipeline

      **ChannelHandler** 是消息的具体处理器。他负责处理读写操作、客户端连接等事情。

      **ChannelPipeline** 为 **ChannelHandler** 的链，提供了一个容器并定义了用于沿着链传播入站和出站事件流的 API 。当 **Channel** 被创建时，它会被自动地分配到它专属的**ChannelPipeline**。

   5. EventLoopGroup

      上图是一个服务端对 EventLoopGroup 使用的大致模块图，其中 Boss EventloopGroup 用于接收连接，Worker EventloopGroup 用于具体的处理（消息的读写以及其他逻辑处理）。

      从上图可以看出：当客户端通过 connect 方法连接服务端时，bossGroup 处理客户端连接请求。当客户端处理完成后，会将这个连接提交给 workerGroup 来处理，然后 workerGroup 负责处理其 IO 相关操作<img src="/home/halo/notes/netty.jpg" style="zoom:150%;" />

5. ### 线程模型

   1. Reactor模型:

      Reactor 模式基于事件驱动，采用多路复用将事件分发给相应的 Handler 处理，非常适合处理海量 IO 的场景。

      在 Netty 主要靠 NioEventLoopGroup 线程池来实现具体的线程模型的 。

      我们实现服务端的时候，一般会初始化两个线程组：

      1. **bossGroup** :接收连接。
      2. **workerGroup** ：负责具体的处理，交由对应的 Handler 处理。

   2. 使用 NioEventLoopGroup 类的无参构造函数设置线程数量的默认值就是 **CPU 核心数 \*2** 

   3. 三种处理模型

      1. **单线程模型** ：

         一个线程需要执行处理所有的 accept、read、decode、process、encode、send 事件。对于高负载、高并发，并且对性能要求比较高的场景不适用。

      2. **多线程模型**

         一个 Acceptor 线程只负责监听客户端的连接，一个 NIO 线程池负责具体处理：accept、read、decode、process、encode、send 事件。满足绝大部分应用场景，并发连接量不大的时候没啥问题，但是遇到并发连接大的时候就可能会出现问题，成为性能瓶颈。

      3. **主从多线程模型**

         从一个 主线程 NIO 线程池中选择一个线程作为 Acceptor 线程，绑定监听端口，接收客户端连接的连接，其他线程负责后续的接入认证等工作。

6. 如何启动:

   1. 服务端:

      ```java
          // 1.bossGroup 用于接收连接，workerGroup 用于具体的处理
              EventLoopGroup bossGroup = new NioEventLoopGroup(1);
              EventLoopGroup workerGroup = new NioEventLoopGroup();
              try {
                  //2.创建服务端启动引导/辅助类：ServerBootstrap
                  ServerBootstrap b = new ServerBootstrap();
                  //3.给引导类配置两大线程组,确定了线程模型
                  b.group(bossGroup, workerGroup)
                          // (非必备)打印日志
                          .handler(new LoggingHandler(LogLevel.INFO))
                          // 4.指定 IO 模型
                          .channel(NioServerSocketChannel.class)
                          .childHandler(new ChannelInitializer<SocketChannel>() {
                              @Override
                              public void initChannel(SocketChannel ch) {
                                  ChannelPipeline p = ch.pipeline();
                                  //5.可以自定义客户端消息的业务处理逻辑
                                  p.addLast(new HelloServerHandler());
                              }
                          });
                  // 6.绑定端口,调用 sync 方法阻塞知道绑定完成
                  ChannelFuture f = b.bind(port).sync();
                  // 7.阻塞等待直到服务器Channel关闭(closeFuture()方法获取Channel 的CloseFuture对象,然后调用sync()方法)
                  f.channel().closeFuture().sync();
              } finally {
                  //8.优雅关闭相关线程组资源
                  bossGroup.shutdownGracefully();
                  workerGroup.shutdownGracefully();
              }
      ```

      

      1. 首先创建了两个 NioEventLoopGroup 对象实例：bossGroup 和 workerGroup。
         1. bossGroup : 用于处理客户端的 TCP 连接请求。
         2. workerGroup ：负责每一条连接的具体读写数据的处理逻辑，真正负责 I/O 读写操作，交由对应的 Handler 处理
      2. 接下来 我们创建了一个服务端启动引导/辅助类：ServerBootstrap，这个类将引导我们进行服务端的启动工作。
      3. 通过 .group() 方法给引导类 ServerBootstrap 配置两大线程组，确定了线程模型。
      4. 通过channel()方法给引导类 ServerBootstrap指定了 IO 模型为NIO
         + NioServerSocketChannel ：指定服务端的 IO 模型为 NIO，与 BIO 编程模型中的ServerSocket对应
         + NioSocketChannel : 指定客户端的 IO 模型为 NIO， 与 BIO 编程模型中的Socket对应
      5. 通过 .childHandler()给引导类创建一个ChannelInitializer ，然后制定了服务端消息的业务处理逻辑 HelloServerHandler 对象
      6. 调用 ServerBootstrap 类的 bind()方法绑定端口

   2. 客户端启动

      ```java
             //1.创建一个 NioEventLoopGroup 对象实例
              EventLoopGroup group = new NioEventLoopGroup();
              try {
                  //2.创建客户端启动引导/辅助类：Bootstrap
                  Bootstrap b = new Bootstrap();
                  //3.指定线程组
                  b.group(group)
                          //4.指定 IO 模型
                          .channel(NioSocketChannel.class)
                          .handler(new ChannelInitializer<SocketChannel>() {
                              @Override
                              public void initChannel(SocketChannel ch) throws Exception {
                                  ChannelPipeline p = ch.pipeline();
                                  // 5.这里可以自定义消息的业务处理逻辑
                                  p.addLast(new HelloClientHandler(message));
                              }
                          });
                  // 6.尝试建立连接
                  ChannelFuture f = b.connect(host, port).sync();
                  // 7.等待连接关闭（阻塞，直到Channel关闭）
                  f.channel().closeFuture().sync();
              } finally {
                  group.shutdownGracefully();
              }
      ```

      1.创建一个 NioEventLoopGroup 对象实例

      2.创建客户端启动的引导类是 Bootstrap

      3.通过 .group() 方法给引导类 Bootstrap 配置一个线程组

      4.通过channel()方法给引导类 Bootstrap指定了 IO 模型为NIO

      5.通过 .childHandler()给引导类创建一个ChannelInitializer ，然后制定了客户端消息的业务处理逻辑 HelloClientHandler 对象

      6.调用 Bootstrap 类的 connect()方法进行连接，这个方法需要指定两个参数：

      + inetHost : ip 地址
      + inetPort : 端口号

7. ### tcp粘包/拆包问题:

   1. 粘包: 在socket通讯过程中，如果通讯的一端一次性连续发送多条数据包，tcp协议会将多个数据包打包成一个tcp报文发送出去，这就是所谓的**粘包**。

   2. 拆包: 如果通讯的一端发送的数据包超过一次tcp报文所能传输的最大值时，就会将一个数据包拆成多个最大tcp长度的tcp报文分开传输，这就叫做**拆包**

   

   3. 出现原因:

      1. 粘包:

         - 要发送的数据小于TCP发送缓冲区的大小，TCP将多次写入缓冲区的数据一次发送出去；
         - 接收数据端的应用层没有及时读取接收缓冲区中的数据；
         - 数据发送过快，数据包堆积导致缓冲区积压多个数据后才一次性发送出去(如果客户端每发送一条数据就睡眠一段时间就不会发生粘包)；
      2. 拆包: 如果数据包太大，超过MSS的大小，就会被拆包成多个TCP报文分开传输。

      3. 解决方法:
         - 在数据包头加上数据包大小
         - 在数据包上加分隔符, 如netty的几个解码器:
           - **LineBasedFrameDecoder** : 发送端发送数据包的时候，每个数据包之间以换行符作为分隔
           - **DelimiterBasedFrameDecoder** : 可以自定义分隔符解码器
           - **FixedLengthFrameDecoder**: 固定长度解码器，它能够按照指定的长度对消息进行相应的拆包。

   4. ### netty长连接, 心跳机制:

      1. 长连接:

         对于频繁请求资源的客户来说， 即使 client 与 server 完成一次读写，它们之间的连接并不会主动关闭，后续的读写操作会继续使用这个连接。长连接的可以省去较多的 TCP 建立和关闭的操作，降低对网络资源的依赖，节约时间。

      2. 心跳机制:

         在 TCP 保持长连接的过程中，可能会出现断网等网络异常出现，异常发生的时候， client 与 server 之间如果没有交互的话，他们是无法发现对方已经掉线的。为了解决这个问题, 我们就需要引入 **心跳机制** 。

         心跳机制的工作原理是: 在 client 与 server 之间在一定时间内没有数据交互时, 即处于 idle 状态时, 客户端或服务器就会发送一个特殊的数据包给对方, 当接收方收到这个数据报文后, 也立即发送一个特殊的数据报文, 回应发送方, 此即一个 PING-PONG 交互。所以, 当某一端收到心跳消息后, 就知道了对方仍然在线, 这就确保 TCP 连接的有效性.

   5. ### netty零拷贝:

      1. 避免数据流经用户空间(mmap+write/sendfile(linux2.1引入)):

         Netty在这一层对零拷贝实现就是`FileRegion`类的`transferTo()`方法，不再是数据从设备A->内核缓冲区->用户缓冲区->内核缓冲区->设备B, 而是直接从设备A的内核缓冲区发到B的内核缓冲区(如socket)

      2. java层次:Netty的接收和发送ByteBuffer使用直接内存进行Socket读写，不需要进行字节缓冲区的二次拷贝。如果使用JVM的堆内存进行Socket读写，JVM会将堆内存Buffer拷贝一份到直接内存中，然后才写入Socket中。相比于使用直接内存，消息在发送过程中多了一次缓冲区的内存拷贝。

      3. Netty提供CompositeByteBuf类, 可以将多个ByteBuf合并为一个逻辑上的ByteBuf, 避免了各个ByteBuf之间的拷贝。

      4. 通过wrap操作, 我们可以将byte[]数组、ByteBuf、ByteBuffer等包装成一个Netty ByteBuf对象, 进而避免拷贝操作
      
      5. ByteBuf支持slice操作，可以将ByteBuf分解为多个共享同一个存储区域的ByteBuf, 避免内存的拷贝。