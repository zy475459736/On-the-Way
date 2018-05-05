## Bootstrap

### 简述

展示一下Netty的客户端和服务端的初始化和启动的流程，不会深入每个功能模块

会从Bootstrap/ServerBootstrap类 入手。

### Bootstrap

Bootstrap是Netty提供的一个便利的工厂类，可以通过它来完成Netty的客户端或服务端的Netty初始化。

### 客户端部分

下面是源码**example/src/main/java/io/netty/example/echo/EchoClient.java** 的客户端部分的启动代码:

```java
EventLoopGroup group = new NioEventLoopGroup();
try {
    Bootstrap b = new Bootstrap();
    b.group(group)
     .channel(NioSocketChannel.class)
     .option(ChannelOption.TCP_NODELAY, true)
     .handler(new ChannelInitializer<SocketChannel>() {
         @Override
         public void initChannel(SocketChannel ch) throws Exception {
             ChannelPipeline p = ch.pipeline();
             p.addLast(new EchoClientHandler());
         }
     });

    // Start the client.
    ChannelFuture f = b.connect(HOST, PORT).sync();

    // Wait until the connection is closed.
    f.channel().closeFuture().sync();
} finally {
    // Shut down the event loop to terminate all threads.
    group.shutdownGracefully();
}
```

从上面的客户端代码虽然简单, 但是却展示了 Netty 客户端初始化时所需的所有内容:

1. EventLoopGroup: 不论是服务器端还是客户端, 都必须指定 EventLoopGroup. 在这个例子中, 指定了 NioEventLoopGroup, 表示一个 NIO 的EventLoopGroup.
2. ChannelType: 指定 Channel 的类型. 因为是客户端, 因此使用了 NioSocketChannel.
3. Handler: 设置数据的处理器.

### NioSocketChannel 的初始化过程

在 Netty 中, Channel 是一个 Socket 的抽象, 它为用户提供了关于 Socket 状态(是否是连接还是断开) 以及对 Socket 的读写等操作. 每当 Netty 建立了一个连接后, 都会有一个对应的 Channel 实例.

NioSocketChannel的类层次结构如下：

N