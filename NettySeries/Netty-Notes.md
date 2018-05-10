[TOC]



#### Netty包结构

```
org
└── jboss
    └── netty
		├── bootstrap 配置并启动服务的类
		├── buffer 缓冲相关类，对NIO Buffer做了一些封装
		├── channel 核心部分，处理连接
		├── container 连接其他容器的代码
		├── example 使用示例
		├── handler 基于handler的扩展部分，实现协议编解码等附加功能
		├── logging 日志
		└── util 工具类
```



#### 基础入门

> Netty定义：高性能、异步事件驱动的NIO框架，提供对TCP UDP 和文件传输的支持。
>

作为一个异步NIO框架，Netty的所有IO操作都是 **异步非阻塞 **的， **future-listener机制**，用户可以方便地主动获取或者通过通知机制来获得io操作结果





#### RPC调用的性能模型分析

##### 传统RPC性能差的三宗罪
![image](https://res.infoq.com/articles/netty-high-performance/zh/resources/0529010.png)
- **网络传输方式问题**

  BIO通信模型 典型的一请求一应答模型
  最大的问题就是<u>不具备弹性伸缩能力</u>，服务端的线程个数和并发访问数成线性正比

- **序列化方式问题** 
    Java序列化存在如下几个典型问题
    - Java序列化机制无法跨版本
    - Java序列化后的码流太大
    - 序列化性能差（cpu资源占用高）

- **线程模型问题** 

    每个TCP连接都占用1个线程

    ​

##### 高性能的三个主题
![image](https://res.infoq.com/articles/netty-high-performance/zh/resources/0529011.png)
- **传输** IO模型的选择
- **协议** http？内部私有协议？后者的性能相对更优
- **线程** Reactor线程模型



#### Netty高性能之道

##### 异步非阻塞通讯

> **IO多路复用技术**
>
> 通过把多个IO的阻塞复用到同一个select的阻塞上，从而使得系统在单线程的情况下可以同时处理多个客户端请求。
>
> 与传统的多线程/多进程模型比，I/O多路复用的最大优势是系统开销小，系统不需要创建新的额外进程或者线程，也不需要维护这些进程和线程的运行，降低了系统的维护工作量，节省了系统资源。

JDK NIO通信模型如下所示：
![image](https://res.infoq.com/articles/netty-high-performance/zh/resources/0529012.png)
Netty架构按照Reactor模式设计和实现
**服务端通信**序列图如下：
![image](https://res.infoq.com/articles/netty-high-performance/zh/resources/0529013.png)
**客户端通信**序列图如下：
![image](https://res.infoq.com/articles/netty-high-performance/zh/resources/0529014.png)



##### 零拷贝
Netty的“零拷贝”主要体现在如下三个方面：

1. Netty的接收和发送**ByteBuffer采用DIRECT BUFFERS**，使用堆外直接内存进行Socket读写，不需要进行字节缓冲区的二次拷贝。如果使用传统的堆内存（HEAP BUFFERS）进行Socket读写，JVM会将堆内存Buffer拷贝一份到直接内存中，然后才写入Socket中。相比于堆外直接内存，消息在发送过程中多了一次缓冲区的内存拷贝。
2. Netty提供了**组合Buffer对象**，可以聚合多个ByteBuffer对象，用户可以像操作一个Buffer那样方便的对组合Buffer进行操作，避免了传统通过内存拷贝的方式将几个小Buffer合并成一个大的Buffer。
3. Netty的文件传输采用了**transferTo**方法，它可以直接将文件缓冲区的数据发送到目标Channel，避免了传统通过循环write方式导致的内存拷贝问题。

（todo 代码实践）

##### 内存池

（todo 代码实践）

##### 高效的Reactor线程模型

常用的有以下三种：

* Reactor单线程模型
* Reactor多线程模型
* 主从Reactor多线程模型

###### Reactor单线程模型(适用于小容量应用场景)

> Reactor单线程模型，指的是所有的IO操作都在同一个NIO线程上面完成

![](https://res.infoq.com/articles/netty-high-performance/zh/resources/0808020.png)





###### Reactor多线程模型

Reactor多线程模型与单线程模型最大的区别就是有一组NIO线程处理IO操作

![image](https://res.infoq.com/articles/netty-high-performance/zh/resources/08080211.png)

- 一个NIO线程-Acceptor线程用于监听，接受客户端tcp连接请求
- 一个NIO线程池 用于网络IO操作。负责消息的读取发送 编码解码。
- 1NIO线程：N条链路



###### 主从Reactor线程模型示意图

> **主从Reactor线程模型的特点是**：
>
> 服务端用于接收客户端连接的不再是个1个单独的NIO线程，而是一个独立的NIO线程池。Acceptor接收到客户端TCP连接请求处理完成后（可能包含接入认证等），将新创建的SocketChannel注册到IO线程池（sub reactor线程池）的某个IO线程上，由它负责SocketChannel的读写和编解码工作。Acceptor线程池仅仅只用于客户端的登陆、握手和安全认证，一旦链路建立成功，就将链路注册到后端subReactor线程池的IO线程上，由IO线程负责后续的IO操作。

![image](https://res.infoq.com/articles/netty-high-performance/zh/resources/08080221.png)

##### 无锁化的串行设计理念
**串行化设计**：即消息的处理尽可能在同一个线程内完成，期间不进行线程切换，这样就避免了多线程竞争和同步锁。
![image](https://res.infoq.com/articles/netty-high-performance/zh/resources/0529035.png)

##### 高效的并发编程

1. volatile的大量且正确的使用
2. CAS 和 原子类 的广泛使用
3. 线程安全容器的使用
4. 读写锁提升并发性能

##### 高性能的序列化框架

序列化性能的关键因素总结如下：

1) 序列化后的码流大小（网络带宽的占用）；
2) 序列化&反序列化的性能（CPU资源占用）；
3) 是否支持跨语言（异构系统的对接和开发语言切换）。

##### 灵活的TCP参数配置能力

合理设置TCP参数在某些场景下对于性能的提升可以起到显著的效果





[原文](http://www.infoq.com/cn/articles/netty-high-performance)