#### Executor

![](https://github.com/zy475459736/markdown-pics/raw/master/Diagrams/Executor.png)

执行所提交的Runnable 任务。

将 **任务的提交** 和 **任务的执行** 解耦。

任务的执行机制包括：线程的使用、调度等。

使用Executor后也不用再显示地创建线程。



#### ExecutorService 

```java
implements Executor
```

![](https://github.com/zy475459736/markdown-pics/blob/master/Diagrams/ExecutorService.png?raw=true)

主要提供如下和 **任务管理** 功能 相关的方法：

* 停止
* 跟踪一个或者多个异步任务的进度（通过Future）





#### AbstractExecutorService

```java
implements ExecutorService
```

![](https://github.com/zy475459736/markdown-pics/blob/master/Diagrams/AbstractExecutorService.png?raw=true)

ExecutorService的默认实现：使用newTaskFor()方法返回一个FutureTask（RunnabeFuture的默认实现类）来实现submit()、invokeAny()、invokeAll()方法。

子类可重写newTaskFor()方法。



#### ScheduledExecutorService

```java
implements ExecutorService
```

![](https://github.com/zy475459736/markdown-pics/blob/master/Diagrams/ScheduledExecutorService.png?raw=true)

主要是提供任务执行相关的调度管理方法：

* with certain delay
* periodically





#### Callable

![](https://github.com/zy475459736/markdown-pics/blob/master/Diagrams/Callable.png?raw=true)

和Runnable类似，不同之处仅在于：会**返回任务运行后的结果**

> 该结构：
>
> 1、由范性来定义
>
> 2、可能是个异常



#### Future<V>

![](https://github.com/zy475459736/markdown-pics/blob/master/Diagrams/Future.png?raw=true)

Future代表了异步计算的结果。

如果调用get()方法时，计算还没有完成，就会block在该方法上。



#### RunnableFuture<V>

```java
extends  Runnable, Future<V> 
```

![](https://github.com/zy475459736/markdown-pics/blob/master/Diagrams/RunnableFuture.png?raw=true)

一个可以运行的Future。run()方法执行结束之后，Future的isDone()就返回true，且可以get()到结果。



#### FutureTask<v>

```java
implements RunnableFuture<V> 
```

![](https://github.com/zy475459736/markdown-pics/blob/master/Diagrams/FutureTask.png?raw=true)

一个可以取消的异步计算。

**提供了Future的基本实现：**

* 计算的开始、取消 和 查询
* 结果的获取

runAndReset()方法

可以用于包装Callable和Runnable对象。

