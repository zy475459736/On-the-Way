## 背景

### 传统的BIO编程

BIO&伪BIO 

> 前者是典型的一请求一应答通信模型，缺乏弹性伸缩能力，在并发数膨胀后基本就崩了。
>
> 后者是在前者模型的基础上，增加了一线程池，通过限制程序所用的线程总数来保证服务器机器本身的性能，但是并发数膨胀后，吞吐依旧较低，客户端的体验依旧无法保证。
>
> 最大性能瓶颈在于：Java网络编程中所使用的Stream，它自身是阻塞的（细节另行探讨）。



### NIO带来的变化

JDK中NIO类库的引入，带来的对标Stream的Channel，提供了阻塞和**非阻塞**两种模式。

对于高负载高并发的应用而言，需要使用NIO的非阻塞模式进行开发。



### AIO编程

NIO2.0引入了新的异步通道的概念，并提供了异步文件通道和异步套接字通道的实现。

异步通道提供两种方式获取操作结果：

* java.util.concurrent.Future类表示异步操作的结果；
* 在执行异步操作的时候传入一个java.nio.channels;
* CompletionHandler接口的实现类作为操作完成的回调。

NIO2.0的异步套接字通道是真正的异步非阻塞IO，对应UNIX网络编程中的AIO。



## 服务端

### JDK NIO类库

直接使用JDK NIO的类库开发基于NIO的异步服务端时，需要使用到：

* Selector
* ServerSocketChannel/SocketChannel
* ByteBuffer
* SelectionKey

而Netty为了向使用者屏蔽NIO通信的底层细节，在和用户交互的边界做了封装，目的就是为了减少用户开发工作量，降低开发难度。



### Netty服务端创建流程

#### ServerBootStrap

它是Netty服务端的启动辅助类，提供了一系列的方法用于设置服务端启动相关的参数。

底层通过<u>门面模式</u>对各种能力进行抽象和封装，避免开发人员过多地和底层API打交道。

**ServerBootStrap只有一个无参的构造函数，采用Builder模式来进行相关的初始化。**

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/Server-Client%20Bootstrap.png?raw=true)



#### EventLoopGroup

> 类比Reactor模式中的线程池，它实际是EventLoop的数组。

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/EventLoopGroup.png?raw=true)





##### EventLoop

> 类比NIO类库中的Selector

EventLoop的职责是处理所有注册到本线程多路复用器Selector上的Channel，Selector的轮询操作由绑定的EventLoop线程run方法驱动，在一个循环体内循环执行。

除了网络IO事件外，同时还处理用户自定义的Task和定时任务Task。

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/NioEventLoop.png?raw=true)



#### NioServerSocketChannel

> 类比NIO类库中的Channel，或者说ServerSocketChannel

用户并不关系底层的实现细节和工作原理，因此只需通过ServerBootStrap的channel方法直接制定服务端Channel的类型即可。

Netty通过工厂类，利用反射完成Channel对象的创建。

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/NioServerSocketChannel.png?raw=true)



#### ChannelPipeline

> 并不是NIO服务端必须的，它是Netty中特有的，其本质是一个负责处理网络事件的职责链，负责管理和执行ChannelHandler。

在**链路建立的时候创建并初始化**ChannelPipeline。

网络事件以事件流的形式在ChannelPipeline中流转，由ChannelPipeLine根据ChannelHandler的执行策略调度ChannelHandler的执行。

<u>典型的网络事</u>件如下：

* 链路注册
* 链路激活
* 链路断开
* 接受请求信息
* 请求消息接受并处理完毕
* 发送应答消息
* 链路异常
* 用户自定义事件

#### ChannelHandler

> ChannelHandler是Netty提供给用户 **定制** 和 **扩展** 的关键接口。

利用ChannelHandler可以实现大多数的功能定制；同时Netty也提供了实现好的Handler，总结如下：

* 系统编解码框架 ByteToMessageCodec
* 通用基于长度的半包解码器 LengthFieldBasedFrameDecoder
* 码流日志打印 LoggingHandler
* SSL安全认真 SslHandler
* 链路空闲检测 IdleStateHandler
* 流量整形ChannelTrafficShapingHandler
* Base64编解码 Base64Decoder & Base64Encoder

#### 绑定并启动监听端口

流程如下：

1、在绑定监听端口之前系统会做一系列的初始化和检测工作。

2、完成之后，启动监听端口，

3、将ServerSocketChannel注册到Selector上监听客户端连接。

#### Selector轮询

由Reactor线程NioEventLoop负责调度和执行Selector轮询操作，选择准备就绪的Channel集合。

#### Channel准备就绪

当Selector轮询到准备就绪的Channel之后，就由Reactor线程NioEventLoop执行ChannelPipeline的相应方法，最终调度并执行ChannelHandler。

#### ChannelHandler的调度并执行

Netty系统ChannelHandler和用户定制ChannelHandler的执行。





## Netty服务端源码分析

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/Server-Client%20Bootstrap.png?raw=true)



ServerBootStrap.group()传入两个NioEventLoopGroup，其中bossGroup传入AbstractBootStrap构造函数中。workerGroup传入ServerBootStrap中。

BootstrapChannelFactory是AbstractBootstrap的静态内部类，职责是根据Channel的类型通过反射创建Channel的实例。

制定NioServerSocketChannel后，需要设置TCP的一些参数，作为服务端，主要是设置TCP的backlog参数，该参数制定了内核为此套接口排队的最大连接个数，Netty默认是100，可以修改。

TCP参数设置完成后，可以为启动辅助类及其父类分别制定Handler：

* 子类中Handler是NioServerSocketChannel对应ChannelPipeline的Handler
* 父类中Handler是客户端新接入的连接SocketChannel对应的ChannelPipeline的Handler

本质区别就是：ServerBootstrap中的Handler是NioServerSocketChannel使用的，所有连接该监听端口的客户端都会执行它，父类AbstractBootstrap中的Handler是个工厂类，为每个新接入的客户端都创建一个新的Handler。

服务端启动的最后一步，就是绑定本地端口，启动服务：

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/NettyServer00.png?raw=true)

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/NettyServer01.png?raw=true)

![](https://github.com/zy475459736/markdown-pics/blob/master/Netty/NettyServer02.png?raw=true)



