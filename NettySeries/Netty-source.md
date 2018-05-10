#### 

## Bootstrap

### 简述

从**Bootstrap/ServerBootstrap类**入手简述Netty程序客户端和服务端的**初始化和启动的流程**

### Bootstrap 介绍

Bootstrap是Netty提供的一个便利的**工厂类**，可以通过它来完成Netty的客户端或服务端的Netty**初始化**。

### 客户端部分

#### 连接源码

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

从上面的客户端代码展示了 **Netty 客户端初始化时所需的所有动作**:

1. **EventLoopGroup**: 不论是服务器端还是客户端, 都必须指定 EventLoopGroup. 这个例子的NioEventLoopGroup, 表示一个 NIO 的EventLoopGroup.
2. **ChannelType**: 指定 Channel 的类型. 因为是客户端, 因此使用了 NioSocketChannel.
3. **Handler**: 设置数据的处理器.

#### NioSocketChannel 的初始化过程

在 Netty 中, **Channel 是一个 Socket 的抽象,** 

> 操作： 
>
> * **Socket 状态(是否是连接还是断开)的查询**
> * **Socket 的读写**

> 对应Netty应用而言：
>
> ​	一个连接，对应的一个Channel 实例.

NioSocketChannel的类层次结构如下：

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/NioSocketChannel.png?raw=true)



##### ChannelFactory 和 Channel 类型的确定

###### Channel类型

**<协议，IO版本，C/S>集合唯一确定一个Channel类型。**

> **协议**：除了TCP之外，还包括UDP、Sctp等。
>
> **IO版本**：主要是指异步IO（Nio）和阻塞IO（Oio）。
>
> **C/S** ：即客户端和服务端。
>

一些常用的 Channel 类型:

- NioSocketChannel,              异步的客户端    TCP Socket 连接.
- NioServerSocketChannel,  异步的服务器端 TCP Socket 连接.
- NioDatagramChannel,       异步的                 UDP 连接
- NioSctpChannel,                 异步的客户端     Sctp 连接.
- NioSctpServerChannel,     异步的服务器端  Sctp 连接.
- OioSocketChannel,             同步的客户端     TCP Socket 连接.
- OioServerSocketChannel, 同步的服务器端 TCP Socket 连接.
- OioDatagramChannel,       同步的                UDP 连接
- OioSctpChannel,                 同步的服务器端 Sctp连接.
- OioSctpServerChannel,     同步的客户端     TCP Socket 连接.

###### ChannelFactory和Channel类型的指定

Bootstrap中通过**channel() 方法**的调用来设置Channel的类型

调用 channel() 方法, 传入 NioSocketChannel.class, <u>这个方法其实同时初始化了一个 BootstrapChannelFactory</u>

> 可以理解为Bookstrap.channel()静态方法的调用同时确定了**ChannelFactory和Channel**两个类型，
>
> 前者就是BootstrapChannelFactory，后者是传入channel()方法的参数。

```java
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    }
    return channelFactory(new BootstrapChannelFactory<C>(channelClass));
}

	public B channelFactory(ChannelFactory<? extends C> channelFactory) {
        if (channelFactory == null) {
            throw new NullPointerException("channelFactory");
        }
        if (this.channelFactory != null) {
            throw new IllegalStateException("channelFactory set already");
        }

        this.channelFactory = channelFactory;
        return (B) this;
    }
```

而 BootstrapChannelFactory 实现了 ChannelFactory 接口, 它提供了唯一的方法, 即 **newChannel**. 

> ChannelFactory, 顾名思义, 就是产生 Channel 的工厂类.

进入到 BootstrapChannelFactory.newChannel 中, 我们看到其实现代码如下:

```java
@Override
public T newChannel() {
    // 删除 try 块
    return clazz.newInstance();
}
```

根据上面代码的提示, 我们就可以确定:

- Bootstrap 中的 ChannelFactory 的实现是 BootstrapChannelFactory
- 生成的 Channel 的具体类型是 NioSocketChannel. 
  Channel 的实例化过程, 其实就是调用的 ChannelFactory#newChannel 方法, 而实例化的 Channel 的具体的类型又是和在初始化 Bootstrap 时传入的 channel() 方法的参数相关. 

##### Channel 实例化

前面已经知道了如何确定一个 Channel 的类型, 并且了解到 Channel 是通过工厂方法 ChannelFactory.newChannel() 来实例化的, 那么 ChannelFactory.newChannel() 方法在哪里调用呢?
继续跟踪, 我们发现其调用链是:

```java
Bootstrap.connect -> Bootstrap.doConnect -> AbstractBootstrap.initAndRegister
```

在 AbstractBootstrap.initAndRegister 中就调用了 **channelFactory().newChannel()** 来获取一个新的 NioSocketChannel 实例, 其源码如下:

```java
final ChannelFuture initAndRegister() {
    // 去掉非关键代码
    final Channel channel = channelFactory().newChannel();
    init(channel);
    ChannelFuture regFuture = group().register(channel);
}
```

在 **newChannel** 中, 通过**类对象的 newInstance** 来获取一个新 Channel 实例, 因而会调用NioSocketChannel 的**默认构造器**.
NioSocketChannel 默认构造器代码如下:

```java
public NioSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
public NioSocketChannel(SocketChannel socket) {
    this(null, socket);
}
public NioSocketChannel(Channel parent, SocketChannel socket) {
    super(parent, socket);
    config = new NioSocketChannelConfig(this, socket.socket());
}
```

> `这里的代码比较关键`, 在这个构造器中, 会调用 **newSocket** 来打开一个新的 Java NIO SocketChannel:
>
> ```java
> private static SocketChannel newSocket(SelectorProvider provider) {
>     ...
>     return provider.openSocketChannel();
> }
> ```
>


接着会调用父类, 即 AbstractNioByteChannel 的构造器:
```java
AbstractNioByteChannel(Channel parent, SelectableChannel ch){}
```

并传入参数 parent 为 null, ch 为刚才使用 newSocket 创建的 Java NIO SocketChannel, 因此生成的 NioSocketChannel 的 parent channel 是空的.

```java
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
    super(parent, ch, SelectionKey.OP_READ);
}
```

接着会继续调用父类 AbstractNioChannel 的构造器, 并传入了参数 **readInterestOp = SelectionKey.OP_READ**:

```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    // 省略 try 块
    // 配置 Java NIO SocketChannel 为非阻塞的.
    ch.configureBlocking(false);
}
```

然后继续调用父类 AbstractChannel 的构造器:

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    unsafe = newUnsafe();
    pipeline = new DefaultChannelPipeline(this);
}
```

到这里, 一个完整的 NioSocketChannel 就初始化完成了, 

##### NioSocketChannel初始化小节

稍微总结一下构造一个 NioSocketChannel 所需要做的工作:

- NioSocketChannel

  * 调用 NioSocketChannel.newSocket(DEFAULT_SELECTOR_PROVIDER) 打开一个新的 Java NIO SocketChannel
  * SocketChannelConfig config = new NioSocketChannelConfig(this, socket.socket())

- AbstractNioChannel 中的属性:
  - SelectableChannel ch 被设置为 Java SocketChannel, 即 NioSocketChannel#newSocket 返回的 Java NIO SocketChannel.
  - readInterestOp 被设置为 SelectionKey.OP_READ
  - SelectableChannel ch 被配置为非阻塞的 ch.configureBlocking(false)

- AbstractChannel(Channel parent) 中初始化 AbstractChannel 的属性:

  - parent 属性置为 null

  - unsafe 通过newUnsafe() 实例化一个 unsafe 对象, 它的类型是 AbstractNioByteChannel.NioByteUnsafe 内部类

  - pipeline 是 new DefaultChannelPipeline(this) 新创建的实例. 

    `这里体现了:Each channel has its own pipeline and it is created automatically when a new channel is created.`

#### 关于 unsafe 字段的初始化

在实例化 NioSocketChannel 的过程中, 会在父类 AbstractChannel 的构造器中, 调用 newUnsafe() 来获取一个 unsafe 实例. 那么 unsafe 是怎么初始化的呢? 它的作用是什么?
**其实 unsafe 特别关键, 它封装了对 Java 底层 Socket 的操作, 因此实际上是沟通 Netty 上层和 Java 底层的重要的桥梁.**

Unsafe 接口所提供的方法, 这些方法其实都会对应到相关的 Java 底层的 Socket 的操作.

```java
interface Unsafe {
    SocketAddress localAddress();
    SocketAddress remoteAddress();
    void register(EventLoop eventLoop, ChannelPromise promise);
    void bind(SocketAddress localAddress, ChannelPromise promise);
    void connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);
    void disconnect(ChannelPromise promise);
    void close(ChannelPromise promise);
    void closeForcibly();
    void deregister(ChannelPromise promise);
    void beginRead();
    void write(Object msg, ChannelPromise promise);
    void flush();
    ChannelPromise voidPromise();
    ChannelOutboundBuffer outboundBuffer();
}
```

回到 AbstractChannel 的构造方法中, 在这里调用了 newUnsafe() 获取一个新的 unsafe 对象, 而 newUnsafe 方法在 NioSocketChannel 中被重写了:

```java
@Override
protected AbstractNioUnsafe newUnsafe() {
    return new NioSocketChannelUnsafe();
}
```

NioSocketChannel.newUnsafe 方法会返回一个 NioSocketChannelUnsafe 实例. 从这里我们就可以确定了, 在实例化的 NioSocketChannel 中的 unsafe 字段, 其实是一个 NioSocketChannelUnsafe 的实例.

#### 关于 pipeline 的初始化

上面分析了一个 Channel (在这个例子中是 NioSocketChannel) 的大体初始化过程, 但是漏掉了一个关键的部分, 即 ChannelPipeline 的初始化. 

> Each channel has its own pipeline and it is created automatically when a new channel is created.

在实例化一个 Channel 时, 必然伴随着实例化一个 ChannelPipeline. 在上述示例中在 AbstractChannel 的构造器看到了 pipeline 字段被初始化为 DefaultChannelPipeline 的实例. 

DefaultChannelPipeline 构造器

```java
public DefaultChannelPipeline(AbstractChannel channel) {
    if (channel == null) {
        throw new NullPointerException("channel");
    }
    this.channel = channel;

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```

调用 DefaultChannelPipeline 的构造器, 传入了一个 channel, 而这个 channel 其实就是实例化的 NioSocketChannel, DefaultChannelPipeline 会将这个 NioSocketChannel 对象保存在channel 字段中. DefaultChannelPipeline 中还有两个特殊的字段, 即 head 和 tail, 这两个字段是一个双向链表的头和尾. 其实在 DefaultChannelPipeline 中, **维护了一个以 AbstractChannelHandlerContext 为节点的双向链表**, 这个链表是 **Netty 实现 Pipeline 机制的关键**. 

HeadContext 的继承层次结构如下所示:

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/HeadContext.png?raw=true)

TailContext的继承层次结构如下所示：

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/TailContext.png?raw=true)我们可以看到, 链表中 head 是一个 **ChannelOutboundHandler**, 而 tail 则是一个 **ChannelInboundHandler**.
接着看一下 HeadContext 的构造器:

```java
HeadContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, HEAD_NAME, false, true);
    unsafe = pipeline.channel().unsafe();
}
```

它调用了父类 AbstractChannelHandlerContext 的构造器, 并传入参数 inbound = false, outbound = true.
TailContext 的构造器与 HeadContext 的相反, 它调用了父类 AbstractChannelHandlerContext 的构造器, 并传入参数 inbound = true, outbound = false.

综上， head是一个 outboundHandler, 而 tail 是一个inboundHandler, 关于这一点, 大家要特别注意, 因为在分析到 Netty Pipeline 时, 我们会反复用到 **inbound 和 outbound** 这两个属性.

#### 关于 EventLoop 初始化

EchoClient.java 代码中, 在一开始就实例化了一个 **NioEventLoopGroup** 对象, 因此就从它的构造器中追踪一下 EventLoop 的初始化过程.
首先来看一下 NioEventLoopGroup 的类继承层次:

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/EventLoopGroup.png?raw=true)

NioEventLoop 有几个重载的构造器, 不过内容都没有什么大的区别, 最终都是调用的父类MultithreadEventLoopGroup构造器:

```java
protected MultithreadEventLoopGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
    super(nThreads == 0? DEFAULT_EVENT_LOOP_THREADS : nThreads, threadFactory, args);
}
```

> 其中有一点有意思的地方是, 如果我们传入的线程数 nThreads 是0, 那么 Netty 会为我们设置默认的线程数 DEFAULT_EVENT_LOOP_THREADS, 而这个默认的线程数是怎么确定的呢?
> 其实很简单, 在静态代码块中, 会首先确定 DEFAULT_EVENT_LOOP_THREADS 的值:
>
> ```java
> static {
>     DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
>             "io.netty.eventLoopThreads", Runtime.getRuntime().availableProcessors() * 2));
> }
> ```
>
> Netty 会首先从系统属性中获取 "io.netty.eventLoopThreads" 的值, 如果我们没有设置它的话, 那么就返回默认值: 处理器核心数 * 2.
>

回到MultithreadEventLoopGroup构造器中, 这个构造器会继续调用父类 MultithreadEventExecutorGroup 的构造器:

```java
protected MultithreadEventExecutorGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
    // 去掉了参数检查, 异常处理 等代码.
    children = new SingleThreadEventExecutor[nThreads];
    if (isPowerOfTwo(children.length)) {
        chooser = new PowerOfTwoEventExecutorChooser();
    } else {
        chooser = new GenericEventExecutorChooser();
    }

    for (int i = 0; i < nThreads; i ++) {
        children[i] = newChild(threadFactory, args);
    }
}
```

根据代码, 我们就很清楚 MultithreadEventExecutorGroup 中的处理逻辑了

- 创建一个大小为 nThreads 的 SingleThreadEventExecutor 数组
- 根据 nThreads 的大小, 创建不同的 Chooser, 即如果 nThreads 是 2 的幂, 则使用 PowerOfTwoEventExecutorChooser, 反之使用 GenericEventExecutorChooser. 不论使用哪个 Chooser, 它们的功能都是一样的, 即从 children 数组中选出一个合适的 EventExecutor 实例.
- 调用 newChhild 方法初始化 children 数组.

根据上面的代码可以知道, MultithreadEventExecutorGroup 内部维护了一个 EventExecutor 数组, **Netty 的 EventLoopGroup 的实现机制其实就建立在 MultithreadEventExecutorGroup 之上**. 每当 Netty 需要一个 EventLoop 时, 会调用 next() 方法获取一个可用的 EventLoop.
上面代码的最后一部分是 newChild 方法, 这个是一个抽象方法, 它的任务是实例化 EventLoop 对象. 我们跟踪一下它的代码, 可以发现, 这个方法在 NioEventLoopGroup 类中实现了, 其内容很简单，就是实例化一个 NioEventLoop 对象, 然后返回它.

```java
@Override
protected EventExecutor newChild(
        ThreadFactory threadFactory, Object... args) throws Exception {
    return new NioEventLoop(this, threadFactory, (SelectorProvider) args[0]);
}
```

最后总结一下整个 EventLoopGroup 的初始化过程:

- EventLoopGroup(其实是MultithreadEventExecutorGroup) 内部维护一个类型为 EventExecutor children 数组, 其大小是 nThreads, 这样就构成了一个线程池
- 如果我们在实例化 NioEventLoopGroup 时, 如果指定线程池大小, 则 nThreads 就是指定的值, 反之是处理器核心数 * 2
- MultithreadEventExecutorGroup 中会调用 newChild 抽象方法来初始化 children 数组
- 抽象方法 newChild 是在 NioEventLoopGroup 中实现的, 它返回一个 NioEventLoop 实例.
- NioEventLoop 属性:
  - SelectorProvider provider 属性: NioEventLoopGroup 构造器中通过 SelectorProvider.provider() 获取一个 SelectorProvider
  - Selector selector 属性: NioEventLoop 构造器中通过调用通过 selector = provider.openSelector() 获取一个 selector 对象.

#### channel 的注册过程

在前面的分析中,提到, channel 会在 Bootstrap.initAndRegister 中进行初始化, 但是这个方法还会将初始化好的 Channel 注册到 EventGroup 中. 接下来我们就来分析一下 Channel 注册的过程.
回顾一下 AbstractBootstrap.initAndRegister 方法:

```java
final ChannelFuture initAndRegister() {
    // 去掉非关键代码
    final Channel channel = channelFactory().newChannel();
    init(channel);
    ChannelFuture regFuture = group().register(channel);
}
```

当Channel 初始化后, 会紧接着调用 group().register() 方法来注册 Channel, 我们继续跟踪的话, 会发现其调用链如下:

> AbstractBootstrap.initAndRegister -> MultithreadEventLoopGroup.register -> SingleThreadEventLoop.register -> AbstractUnsafe.register

通过跟踪调用链, 最终发现是调用到了 unsafe 的 register 方法, 那么**AbstractUnsafe.register 方法**中到底做了什么:

```java
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    // 省略条件判断和错误处理
    AbstractChannel.this.eventLoop = eventLoop;
    register0(promise);
}
```

首先, 将 eventLoop 赋值给 Channel 的 eventLoop 属性, 而我们知道这个 eventLoop 对象其实是 MultithreadEventLoopGroup.next() 方法获取的, 根据我们前面 **关于 EventLoop 初始化** 小节中, 我们可以确定 next() 方法返回的 eventLoop 对象是 NioEventLoop 实例.
register 方法接着调用了 register0 方法:

```java
private void register0(ChannelPromise promise) {
    boolean firstRegistration = neverRegistered;
    doRegister();
    neverRegistered = false;
    registered = true;
    safeSetSuccess(promise);
    pipeline.fireChannelRegistered();
    // Only fire a channelActive if the channel has never been registered. This prevents firing
    // multiple channel actives if the channel is deregistered and re-registered.
    if (firstRegistration && isActive()) {
        pipeline.fireChannelActive();
    }
}
```

register0 又调用了 AbstractNioChannel.doRegister:

```java
@Override
protected void doRegister() throws Exception {
    // 省略错误处理
    selectionKey = javaChannel().register(eventLoop().selector, 0, this);
}
```

javaChannel() 这个方法在前面我们已经知道了, 它返回的是一个 Java NIO SocketChannel, 这里我们将这个 SocketChannel 注册到与 eventLoop 关联的 selector 上了.

总结一下 Channel 的注册过程:

- 首先在 AbstractBootstrap.initAndRegister中, 通过 group().register(channel), 调用 MultithreadEventLoopGroup.register 方法
- 在MultithreadEventLoopGroup.register 中, 通过 next() 获取一个可用的 SingleThreadEventLoop, 然后调用它的 register
- 在 SingleThreadEventLoop.register 中, 通过 channel.unsafe().register(this, promise) 来获取 channel 的 unsafe() 底层操作对象, 然后调用它的 register.
- 在 AbstractUnsafe.register 方法中, 调用 register0 方法注册 Channel
- 在 AbstractUnsafe.register0 中, 调用 AbstractNioChannel.doRegister 方法
- AbstractNioChannel.doRegister 方法通过 javaChannel().register(eventLoop().selector, 0, this) 将 Channel 对应的 Java NIO SockerChannel 注册到一个 eventLoop 的 Selector 中, 并且将当前 Channel 作为 attachment.

总的来说, Channel 注册过程所做的工作就是将 Channel 与对应的 EventLoop 关联, 因此这也体现了, 在 Netty 中, 每个 Channel 都会关联一个特定的 EventLoop, 并且这个 Channel 中的所有 IO 操作都是在这个 EventLoop 中执行的; 当关联好 Channel 和 EventLoop 后, 会继续调用底层的 Java NIO SocketChannel 的 register 方法, 将底层的 Java NIO SocketChannel 注册到指定的 selector 中. 通过这两步, 就完成了 Netty Channel 的注册过程.

#### handler 的添加过程

Netty 的一个强大和灵活之处就是**基于 Pipeline 的自定义 handler 机制**. 基于此, 我们可以像添加插件一样自由组合各种各样的 handler 来完成业务逻辑. 例如我们需要处理 HTTP 数据, 那么就可以在 pipeline 前添加一个 Http 的编解码的 Handler, 然后接着添加我们自己的业务逻辑的 handler, 这样网络上的数据流就向通过一个管道一样, 从不同的 handler 中流过并进行编解码, 最终在到达我们自定义的 handler 中.
在这一小节中, 从简单的入手, 展示一下自定义的 handler 是如何以及何时添加到 ChannelPipeline 中的.

```java
...
.handler(new ChannelInitializer<SocketChannel>() {
     @Override
     public void initChannel(SocketChannel ch) throws Exception {
         ChannelPipeline p = ch.pipeline();
         if (sslCtx != null) {
             p.addLast(sslCtx.newHandler(ch.alloc(), HOST, PORT));
         }
         //p.addLast(new LoggingHandler(LogLevel.INFO));
         p.addLast(new EchoClientHandler());
     }
 });
```

这个代码片段就是实现了 handler 的添加功能. 我们看到, Bootstrap.handler 方法接收一个 ChannelHandler, 而我们传递的是一个 派生于 ChannelInitializer 的匿名类, 它正好也实现了 ChannelHandler 接口. 我们来看一下, ChannelInitializer 类内到底有什么玄机:

```java
@Sharable
public abstract class ChannelInitializer<C extends Channel> extends ChannelInboundHandlerAdapter {

    private static final InternalLogger logger = InternalLoggerFactory.getInstance(ChannelInitializer.class);
    protected abstract void initChannel(C ch) throws Exception;

    @Override
    @SuppressWarnings("unchecked")
    public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        initChannel((C) ctx.channel());
        ctx.pipeline().remove(this);
        ctx.fireChannelRegistered();
    }
    ...
}
```

ChannelInitializer 是一个抽象类, 它有一个抽象的方法 initChannel, 我们正是实现了这个方法, 并在这个方法中添加的自定义的 handler 的. 那么 initChannel 是哪里被调用的呢? 答案是 ChannelInitializer.channelRegistered 方法中. 
我们来关注一下 channelRegistered 方法. 从上面的源码中, 我们可以看到, 在 channelRegistered 方法中, 会调用 initChannel 方法, 将自定义的 handler 添加到 ChannelPipeline 中, 然后调用 ctx.pipeline().remove(this) 将自己从 ChannelPipeline 中删除. 上面的分析过程, 可以用如下图片展示:
一开始, ChannelPipeline 中只有三个 handler, head, tail 和我们添加的 ChannelInitializer.

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/ChannelPipeLIne01.png?raw=true)

接着 initChannel 方法调用后, 添加了自定义的 handler:

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/ChannelPipeLine02.png?raw=true)

最后将 ChannelInitializer 删除:

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/ChannelPipeLine03.png?raw=true)

以上解释了自定义的 handler 是如何添加到 ChannelPipeline 中的。

#### 客户端连接分析

经过上面分析 Netty 初始化,再来分析一下客户端是如何发起 TCP 连接的.

首先, 客户端通过调用 **Bootstrap** 的 **connect** 方法进行连接.
在 connect 中, 会进行一些参数检查后, 最终调用的是 **doConnect0** 方法, 其实现如下:

```java
private static void doConnect0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {

    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                if (localAddress == null) {
                    channel.connect(remoteAddress, promise);
                } else {
                    channel.connect(remoteAddress, localAddress, promise);
                }
                promise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```

在 doConnect0 中, 会在 event loop 线程中调用 Channel 的 connect 方法, 而这个 Channel 的具体类型是什么呢? 我们在 Channel 初始化这一小节中已经分析过了, 这里 channel 的类型就是 **NioSocketChannel**.
进行跟踪到 channel.connect 中, 我们发现它调用的是 DefaultChannelPipeline#connect, 而, pipeline 的 connect 代码如下:

```java
@Override
public ChannelFuture connect(SocketAddress remoteAddress) {
    return tail.connect(remoteAddress);
}
```

而 tail 字段, 我们已经分析过了, 是一个 TailContext 的实例, 而 TailContext 又是 AbstractChannelHandlerContext 的子类, 并且没有实现 connect 方法, 因此这里调用的其实是 AbstractChannelHandlerContext.connect, 我们看一下这个方法的实现:

```java
@Override
public ChannelFuture connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {

    // 删除的参数检查的代码
    final AbstractChannelHandlerContext next = findContextOutbound();
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeConnect(remoteAddress, localAddress, promise);
    } else {
        safeExecute(executor, new OneTimeTask() {
            @Override
            public void run() {
                next.invokeConnect(remoteAddress, localAddress, promise);
            }
        }, promise, null);
    }

    return promise;
}
```

上面的代码中有一个关键的地方, 即 **final AbstractChannelHandlerContext next = findContextOutbound()**, 这里调用 **findContextOutbound** 方法, 从 DefaultChannelPipeline 内的双向链表的 tail 开始, 不断向前寻找第一个 outbound 为 true 的 AbstractChannelHandlerContext, 然后调用它的 invokeConnect 方法, 其代码如下:

```java
private void invokeConnect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) {
    // 忽略 try 块
    ((ChannelOutboundHandler) handler()).connect(this, remoteAddress, localAddress, promise);
}
```

还记得我们在 "关于 pipeline 的初始化" 这一小节分析的的内容吗? 我们提到, 在 DefaultChannelPipeline 的构造器中, 会实例化两个对象: head 和 tail, 并形成了双向链表的头和尾. head 是 HeadContext 的实例, 它实现了 ChannelOutboundHandler 接口, 并且它的 outbound 字段为 true. 因此在 findContextOutbound 中, 找到的 AbstractChannelHandlerContext 对象其实就是 head. 进而在 invokeConnect 方法中, 我们向上转换为 ChannelOutboundHandler 就是没问题的了.
而又因为 HeadContext 重写了 connect 方法, 因此实际上调用的是 HeadContext.connect. 我们接着跟踪到 HeadContext.connect, 其代码如下:

```java
@Override
public void connect(
        ChannelHandlerContext ctx,
        SocketAddress remoteAddress, SocketAddress localAddress,
        ChannelPromise promise) throws Exception {
    unsafe.connect(remoteAddress, localAddress, promise);
}
```

这个 connect 方法很简单, 仅仅调用了 unsafe 的 connect 方法. 而 unsafe 又是什么呢?
回顾一下 HeadContext 的构造器, 我们发现 unsafe 是 pipeline.channel().unsafe() 返回的, 而 Channel 的 unsafe 字段, 在这个例子中, 我们已经知道了, 其实是 AbstractNioByteChannel.NioByteUnsafe 内部类. 兜兜转转了一大圈, 我们找到了创建 Socket 连接的关键代码.
进行跟踪 NioByteUnsafe -> AbstractNioUnsafe.connect:

```java
@Override
public final void connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    boolean wasActive = isActive();
    if (doConnect(remoteAddress, localAddress)) {
        fulfillConnectPromise(promise, wasActive);
    } else {
        ...
    }
}
```

AbstractNioUnsafe.connect 的实现如上代码所示, 在这个 connect 方法中, 调用了 doConnect 方法, `注意, 这个方法并不是 AbstractNioUnsafe 的方法, 而是 AbstractNioChannel 的抽象方法.` doConnect 方法是在 NioSocketChannel 中实现的, 因此进入到 **NioSocketChannel.doConnect** 中:

```java
@Override
protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
    if (localAddress != null) {
        javaChannel().socket().bind(localAddress);
    }

    boolean success = false;
    try {
        boolean connected = javaChannel().connect(remoteAddress);
        if (!connected) {
            selectionKey().interestOps(SelectionKey.OP_CONNECT);
        }
        success = true;
        return connected;
    } finally {
        if (!success) {
            doClose();
        }
    }
}
```

我们终于看到的最关键的部分了, 庆祝一下!
上面的代码不用多说, 首先是获取 Java NIO SocketChannel, 即我们已经分析过的, 从 NioSocketChannel.newSocket 返回的 SocketChannel 对象; 然后是调用 SocketChannel.connect 方法完成 Java NIO 层面上的 Socket 的连接.
最后, 上面的代码流程可以用如下时序图直观地展示:

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/NettyClientTimeDiagram.png?raw=true)





### 服务器端部分

在分析客户端的代码时, 我们已经对 Bootstrap 启动 Netty 有了一个大致的认识, 那么接下来分析服务器端时, 就会相对简单一些了.
首先还是来看一下服务器端的启动代码:

```
public final class EchoServer {

    static final boolean SSL = System.getProperty("ssl") != null;
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));

    public static void main(String[] args) throws Exception {
        // Configure SSL.
        final SslContext sslCtx;
        if (SSL) {
            SelfSignedCertificate ssc = new SelfSignedCertificate();
            sslCtx = SslContextBuilder.forServer(ssc.certificate(), ssc.privateKey()).build();
        } else {
            sslCtx = null;
        }

        // Configure the server.
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 100)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     //p.addLast(new LoggingHandler(LogLevel.INFO));
                     p.addLast(new EchoServerHandler());
                 }
             });

            // Start the server.
            ChannelFuture f = b.bind(PORT).sync();

            // Wait until the server socket is closed.
            f.channel().closeFuture().sync();
        } finally {
            // Shut down all event loops to terminate all threads.
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

和客户端的代码相比, 没有很大的差别, 基本上也是进行了如下几个部分的初始化:

1. EventLoopGroup: 不论是服务器端还是客户端, 都必须指定 EventLoopGroup. 在这个例子中, 指定了 NioEventLoopGroup, 表示一个 NIO 的EventLoopGroup, 不过服务器端需要指定两个 EventLoopGroup, 一个是 bossGroup, 用于处理客户端的连接请求; 另一个是 workerGroup, 用于处理与各个客户端连接的 IO 操作.
2. ChannelType: 指定 Channel 的类型. 因为是服务器端, 因此使用了 NioServerSocketChannel.
3. Handler: 设置数据的处理器.

### Channel 的初始化过程

我们在分析客户端的 Channel 初始化过程时, 已经提到, Channel 是对 Java 底层 Socket 连接的抽象, 并且知道了客户端的 Channel 的具体类型是 NioSocketChannel, 那么自然的, 服务器端的 Channel 类型就是 NioServerSocketChannel 了.
那么接下来我们按照分析客户端的流程对服务器端的代码也同样地分析一遍, 这样也方便我们对比一下服务器端和客户端有哪些不一样的地方.

#### Channel 类型的确定

同样的分析套路, 我们已经知道了, 在客户端中, Channel 的类型其实是在初始化时, 通过 Bootstrap.channel() 方法设置的, 服务器端自然也不例外.

在服务器端, 我们调用了 ServerBootstarap.channel(NioServerSocketChannel.class), 传递了一个 NioServerSocketChannel Class 对象. 这样的话, 按照和分析客户端代码一样的流程, 我们就可以确定, NioServerSocketChannel 的实例化是通过 BootstrapChannelFactory 工厂类来完成的, 而 BootstrapChannelFactory 中的 clazz 字段被设置为了 NioServerSocketChannel.class, 因此当调用 BootstrapChannelFactory.newChannel() 时:

```
@Override
public T newChannel() {
    // 删除 try 块
    return clazz.newInstance();
}
```

就获取到了一个 NioServerSocketChannel 的实例.

最后我们也来总结一下:

- ServerBootstrap 中的 ChannelFactory 的实现是 BootstrapChannelFactory
- 生成的 Channel 的具体类型是 NioServerSocketChannel. 
  Channel 的实例化过程, 其实就是调用的 ChannelFactory.newChannel 方法, 而实例化的 Channel 的具体的类型又是和在初始化 ServerBootstrap 时传入的 channel() 方法的参数相关. 因此对于我们这个例子中的服务器端的 ServerBootstrap 而言, 生成的的 Channel 实例就是 NioServerSocketChannel.

#### NioServerSocketChannel 的实例化过程

首先还是来看一下 NioServerSocketChannel 的实例化过程.
下面是 NioServerSocketChannel 的类层次结构图:







### EventExecutorGroup

```java
extends ScheduledExecutorService, Iterable<EventExecutor> 
```

![](https://github.com/zy475459736/markdown-pics/blob/master/Diagrams/EventExecutorGroup.png?raw=true)



### EventExecutor

![](https://github.com/zy475459736/markdown-pics/blob/master/Diagrams/EventExecutor.png?raw=true)





### Future

```java
package io.netty.util.concurrent
```

![](https://github.com/zy475459736/markdown-pics/blob/master/Diagrams/NettyFuture.png?raw=true)



### Promise

![](https://github.com/zy475459736/markdown-pics/blob/master/Diagrams/Promise.png?raw=true)



### AbstractBootstrap

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/AbstractBootstrap.png?raw=true)



### Bootstrap

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/Bootstrap.png?raw=true)

