# 目录

* 可扩展的网络服务

* 事件驱动处理

* Reactor的特点：
    1. 基本版本
    2. 多线程版本
    3. 其他变化

* java NIO 非阻塞IO的API介绍

    ​



## Network Services
* 种类很多，各种各样的网络服务、分布式组件等
* 相同的步骤：
    读取请求->请求解码->逻辑处理->响应编码->响应发出
* 本质不同，成本也不同
    xml的解析、文件的传输、网页的生成、计算性的服务....

### 传统的服务化设计

![](https://raw.githubusercontent.com/zy475459736/markdown-pics/master/Scalable_IO_in_Java/classic%20service%20designs.png)

一个handler对应一个线程

###  传统的ServerSocket 循环

![](https://raw.githubusercontent.com/zy475459736/markdown-pics/master/Scalable_IO_in_Java/classic%20serversocket%20loop.png)

注意：上述例子中还需要包含较多的异常处理



### 可扩展性所带来的好处

* 负载上升阶段优雅的降级
* 增长资源利用的持续改善
* 可用性和性能要求的满足
  1. 低延迟
  2. 平滑度过业务高峰
  3. tunable的服务质量
* “分治” 策略通常是伸缩性的最佳实践




### “分治”

- 将“计算处理”拆成多个小型任务
  - 每一个任务都能非阻塞地执行
- 任务一“就绪”，就执行
  - 通常是由IO事件触发
- java.nio所提供的基本机制
  - 非阻塞的读写
  - 分配和IO事件相关的任务
- 众多可能的变化
  * 一系列的事件驱动设计



### 事件驱动设计

* 通常效率更高

  * 资源占用率低
    * 一般一个客户端不需要占用一个线程
  * less overhead
    * 上下文切换少、lock也少
  * 分配速率却会比较慢
    * 必须manually bind actions to events

* 通常实现的复杂度也更高

  * 必须分割成一个个非阻塞action
    * 类似GUI事件驱动action
    * 不能消除所有的blocking
  * 必须跟踪服务的逻辑状态

  ​

### Reactor Pattern

* Reactor通过分配合适的handler来响应IO事件

* Handler执行非阻塞的Actions

* 将Handler绑定给事件

* See Schmidt et al, Pattern-Oriented Software Architecture, Volume 2 (POSA2)

  Also Richard Stevens's networking books, MattWelsh's SEDA framework, etc

### Basic Reactor 设计

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/basic%20reactor%20design.png)

### java.nio 提供的工具

* Channels
  * 支持非阻塞读，连接文件、网络等
* Buffers
  * 类似于数组对象，可以通过Channel直接地读写
* Selectors
  * IO事件的告知，隶属于哪一Channels
* SelectionKeys
  * 维护IO事件的状态和绑定

### Reactor 1:设置

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/reactor%201%20setup.png)



### Reactor 2:循环分配(Dispatch Loop)

![](https://raw.githubusercontent.com/zy475459736/markdown-pics/master/Scalable_IO_in_Java/reactor%202%20dispatch%20loop.png)

### Reactor 3:Acceptor

![](https://raw.githubusercontent.com/zy475459736/markdown-pics/master/Scalable_IO_in_Java/reactor%203%20acceptor.png)

### Reactor 4:Handler setup

![](https://raw.githubusercontent.com/zy475459736/markdown-pics/master/Scalable_IO_in_Java/reactor%204%20handler%20setup.png)



### Reactor 5 : Request Handling

![屏幕快照 2018-04-05 下午12.00.22](https://raw.githubusercontent.com/zy475459736/markdown-pics/master/Scalable_IO_in_Java/reactor%205%20request%20handling.png)

### Per-State Handlers

![](https://raw.githubusercontent.com/zy475459736/markdown-pics/master/Scalable_IO_in_Java/per-state%20handlers.png)



### 多线程版的设计

* 为保证伸缩性，策略化地加线程—>个人理解为控制好线程的数量
  * 主要是配合多核处理器
* 工作线程（剔除read和send后，剩下的decode、compute、encode）
  * Reactor应该快速驱动handlers
    * handler处理会减慢Reactor
  * 非io处理能力应该弱于IO处理能力—>个人理解为更多地去关注IO处理能力的提升，因为这个慢的话，会导致阻塞
* 多个Reactor线程
  * Reactor线程主要负责IO处理
  * Reactor上做好负载均衡
    * 在cpu和IO使用率中保证负载均衡



### 工作线程

* 降低非IO处理能力，把资源让给用于加速Reactor线程
  * 和POSA2 Proactor设计类似
* simpler than reworking compute-bound processing into event-driven form
  * 需要保持“真正的”非阻塞计算
    * 足够的计算能力能够覆盖高峰
* 与IO一起进行并行处理比较难
  * 最好的情况：when can first read all input into a buffer
* 利用线程池以进行一定的控制
  * 线程池这种方式下，通常所需要的线程数会远小于客户端的数量



### 工作线程池

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/worker%20thread%20pools.png)



### Handler with Thread Pool

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/handler%20with%20thread%20pool.png)



### 协调任务

* handoffs（接力）
  * 每一个任务都会使能、调用下一个任务
  * 速度快但是比较脆弱（brittle）
* 回调给per-handler 分配器
  * 设置状态、附加字段等
  * 和GoF传递者模式不一样的地方
* 队列
  * 比如 passing buffer across stages
* Futures
  * 当每一个任务产生一个结果，
  * Coordination layered on top of join or wait/notify

### 使用池化的执行器

* 一个tunable的工作线程池
* 主方法 execute（Runnable r）
* 为以下提供控制
  * 任务队列
  * 最大线程数
  * 最小线程数
  * 预热 vs 按需产生的线程
  * 保持存活间隔，直到空闲线程 死亡
    * to be later replaced by new ones if necessary
  * 饱和策略
    * 阻塞、丢弃、producer-runs 等

### 多Reactor线程

* 使用Reactor池

  * 用来匹配cpu和io使用率

  * 静态或者动态创建

    * 每一个都有各自的selector、thread和dispatch loop

  * main acceptor distributes to other reactors

    ![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/multiple%20reactor%20threads.png)

### 使用 多Reactors

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/using%20multiple%20reactors.png)

### 使用java.nio 的其它特征

* 每一个reactor对应多个selectors
  * 为了给不同的IO事件绑定不同的handler
  * 可能需要比较复杂的同步机制来协调
* File transfer
  * 自动化的file-to-net或者net-to-file的复制
* Memory-mapped files
  * 通过缓存来访问文件
* Direct buffers
  * 有时可以达到zero-copy transfer
  * 但是包含了大量的设置和终结
  * 适用于含有长连接的应用

### 基于连接的扩展

* 取代a single service request
  * 客户端链接
  * 客户端发送一系列的消息/请求
  * 客户端断开
* 举例
  * 数据库以及事务的monitor
  * 群聊或者多人参加的游戏
* 可以扩展基本的网络服务模式
  * 应付较多相对长生命周期的客户端
  * 跟踪客户端和session的状态
  * 在多台主机中部署服务

### API Walkthrough

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/api%20walkthrough.png)



### Buffer

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/buffer.png)



### ByteBuffer1

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/ByteBuffer%201.png)

### ByteBuffer2

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/ByteBuffer%202.png)



### Channel

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/Channel.png)



### SelectableChannel

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/Selectable%20Channel.png)

###SocketChannel

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/SocketChannel.png)

### ServerSocketChannel

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/ServerSocketChannel.png)

### FileChannel

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/FileChannel.png)

### Selector

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/Selector.png)

### SelectionKey

![](https://github.com/zy475459736/markdown-pics/raw/master/Scalable_IO_in_Java/SelectionKey.png)















