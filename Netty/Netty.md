# Netty

Netty 是一个异步的、基于事件驱动的网络应用程序框架，用以快速开发高性能、高可靠性的网络 IO 程序。

主要针对在 TCP 协议下， 面向 Clients 端的高并发应用，或者 Peer to Peer 场景下的大量数据持续传输的应用

TCPI/IP    ->    JDK 原生    ->    NIO    ->    Netty



* 互联网行业：在分布式系统中，各个节点之间需要远程服务调用，高性能的 RPC 框架必不可少， Netty 作为异步高性能的通信框架，往往作为基础通信组件被这些 RPC 框架调用
* 阿里分布式服务框架 Dubbo 的 RPC 框架使用 Dubbo 协议进行节点通信， Dubbo 协议默认使用 Netty 作为基础通信组件，用于实现各进程节点之间的内部通信





在 Reactor 模式中，Reactor 等待**某个事件或者可应用或者操作的状态发生（比如文件描述符可读写，或者是 Socket 可读写）**。

然后把这个事件传给事先注册的 Handler（事件处理函数或者回调函数），由后者来做实际的读写操作。

其中的读写操作都需要应用程序同步操作，所以 Reactor 是非阻塞同步网络模型。



## IO 模型

![image-20220813153115020](https://s2.loli.net/2022/08/13/3jXDFiqxzs8JoZT.png)

Nio 叫做新的 IO 模型，定义了通讯双方的**介质抽象**叫 channel，缓冲区作为数据的**载体**在 channel 中进行传递，引用选择器监听通道



* IO 模型的简单理解： 用什么样的通道，进行数据的发送和接受。很大程度上决定了程序通信的性能
* Java 共支持 3 种网络编程模型 IO 模型： BIO NIO AIO
  * BIO ：<u>同步并阻塞</u>，（就是传统的 Java io）服务器实现模式为一个连接一个线程，即，客户端由连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销
  * NIO： <u>同步非阻塞</u>，服务器实现模式为一个线程处理多个请求（连接），即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到有连接由 IO 请求就进行处理
  * AIO： <u>异步非阻塞</u>，AIO 引入了异步通道的概念，采用了 Proactor 模式，有效的请求才启动线程，它的特点是**先由操作系统完成后才通知服务端程序启动线程去处理**，一般适用于连接数较多且连接时间较长的应用
* 适用场景分析
  * BIO： 适用于连接数目比较小，且固定的架构。对服务器资源要求比较高
  * NIO： 适用于连接数目多且连接时间比较短（轻操作）的架构，比如聊天服务器，弹幕系统，服务器间通讯
  * AIO：适用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用 OS 参与并发操作

### Bio 编程简单流程

1. 服务器端启动一个 ServerSocket
2. 客户端启动 Socket 对服务器进行通讯，默认情况下服务器端需要对每个客户建立一个线程与之通讯
3. 客户端发出请求后，先咨询服务器是否有线程响应，如果没有则会等待或者被拒绝
4. 如果服务器端有响应，客户端线程会等待请求结束后，才继续执行



### Nio

* Nio 有三大核心部分： **Channel、Buffer、Selector**，也就是通道、缓冲区、选择器
* Nio 是面向 **缓冲区**，或者面向块 编程的，数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动，这就增加了处理过程中的灵活性，使用它可以提供非阻塞式的高伸缩性网络
* Nio 的非阻塞模式，使一个线程从某通道发送请求或者读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用，就什么都不会获取，而不是保持线程阻塞，所以直至数据变得可以读取之前，该线程可以继续做其他的事情。非阻塞写也是如此，一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程可以同时去做别的事情
* 通俗理解： Nio 可以做到用一个线程处理多个操作。
* Http2.0 使用了多路复用技术，做到同一个连接并发处理多个请求，而且并发请求的数量比 Http1.1 大了好几个数量级

### Bio vs Nio

* Bio 以流的方式处理数据，而 Nio 以块的方式处理数据。后者的 IO 效率高很多
* Bio 是阻塞的， Nio 是非阻塞的
* Bio 基于字节流和字符流进行操作，而 Nio 基于 Channel 和 Buffer 进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入通道中。 Selector 用于监听多个通道的时间，因此使用**单个线程就可以监听多个客户端通道**



### Nio 三大核心原理示意图

1. 每个 channel 都会对应一个 buffer
2. selector 对应一个线程，一个线程对应多个 channel（连接）
3. 该图反映了有三个 channel 注册到 selector 程序
4. 程序切换到哪个 channel 是由事件决定的， Event 就是一个重要的
5. selector 会根据不同事件，在 channel 上切换
6. buffer 就是一个内存块
7. 数据的读取写入是通过 buffer。 Bio 中要么是输入流，要么是输出流，不能双向流动，但是 buffer 是既可以读也可以写的，需要使用 flip() 方法切换
8. channel 是双向的，可以反应底层操作系统的情况

<img src="C:\Users\Fannsy\AppData\Roaming\Typora\typora-user-images\image-20220806155908794.png" alt="image-20220806155908794" style="zoom: 33%;" />



### Buffer

​	缓冲区，本质上是一个可以读写数据的内存块，可以理解成一个 容器对象（含数组），该对象提供了一组方法，可以更轻松地使用内存块。缓冲区对象内置了一些机制，能够跟踪和记录缓冲区的状态变化情况。channel 提供从文件、网络读取数据的渠道，但是读取或写入的数据都必须经过 buffer

​	Buffer 类定义了所有缓冲区都具有的四个属性

	* Capacity： 可以容纳的最大数据量，创建时设定，不可以改变
	* Limit： 缓冲区的当前终点，不能对缓冲区超过极限的位置进行读写
	* Position：位置，下一个要被读写元素的索引
	* Mark：标记



### Channel

Nio 通道类似于 流，但有如下区别：

* 通道可以同时进行读写，而流只能读或写
* 通道可以实现异步读写数据
* 通道可以从缓冲读数据，也可以写数据到缓冲





### Aio

* JDK 7 引入了 AIO。在进行 IO 编程中，常用到两种模式， Reactor 和 Proactor， Java 的 NIO 就是 Reactor，当有事件触发时，服务端得到通知，进行相应的处理
* AIO 叫做 异步不阻塞 的IO，引入异步通道的概念，采用了 Proactor 模式，简化了程序编写。有效的请求才启动线程，它的特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于连接数较多且连接时间较长的应用
* 目前 AIO 还没有广泛应用，Netty 也是基于 NIO，而不是 AIO 



## 零拷贝

* Netty传输的高性能特点

  在Java的内存中，存在有堆内存、栈内存和字符串常量池等等，其中堆内存是占用内存空间最大的一块，也是Java对象存放的地方，一般我们的数据如果需要从IO读取到堆内存，中间需要经过Socket缓冲区，也就是说一个数据会被拷贝两次才能到达它的终点，如果数据量大，就会造成不必要的资源浪费。

  Netty针对这种情况，使用了NIO中的一大特性——零拷贝，当它需要接收数据的时候，它会在堆内存之外开辟一块内存，数据就直接从IO读到了那块内存中去，在netty里面通过ByteBuf可以直接对这些数据进行直接操作，从而加快了传输速度。



---



### mmap 优化

通过内存映射，将文件映射到内核缓冲区，同时，用户空间可以共享内核空间的数据。

这样，在进行网络传输时，就可以减少内核空间到用户空间的拷贝次数



### sendFile 优化

Linux 2.1 版本提供了 sendFile 函数，其基本原理如下：数据根本不经过用户态，直接从内核缓冲区进入到 Socket Buffer，同时，由于和用户态完全无关，就减少了一次上下文切换

在 Linux 2.4 版本中，做了一些修改，避免了从内核缓冲区拷贝到 Socket buffer 的操作，直接拷贝到协议栈，从而再一次减少了数据拷贝

* 零拷贝是从操作系统的角度来说的，因为内核缓冲区之间，没有数据是重复的
* 零拷贝不仅仅带来更少的数据复制，还能带来其他的性能优势，例如更少的上下文切换，更少的 CPU 缓存伪共享以及无 CPU 校验和计算



区别：

* mmap 适合小数据量读写， sendFile 适合大文件传输
* mmap 需要 4 次上下文切换， 3 次数据拷贝； sendFile 需要 3次上下文切换，最少 2 次数据拷贝
* sendFile 可以利用 DMA 方式，减少 CPU 拷贝， mmap 则不能（必须从内核拷贝到 Socket 缓冲区）



## ByteBuf 结构

![image.png](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-img1650817a1455afbbtplv-t2oaga2asx-zoom-in-crop-mark3024000.awebp)

ByteBuf 是一个字节容器，需要掌握以下几部分内容

* 读指针，写指针，变量 capacity。
* ByteBuf 里面其实还有一个参数 maxCapacity，当向 ByteBuf 写数据的时候，如果容量不足，那么这个时候可以进行扩容，直到 capacity 扩容到 maxCapacity，超过 maxCapacity 就会报错



## 心跳机制

> 在 TCP 长连接中，客户端和服务器之间定期发送的一种特殊数据包，通知对方自己还在线，确保 TCP 连接的有效性‘



Netty 中提供了 IdleStateHandler 用于检测连接闲置，该 Handler 可以检测连接未发生读写事件而触发相应事件

```Java
public IdleStateHandler(int readerIdleTimeSeconds, int writerIdleTimeSeconds, int allIdleTimeSeconds) {
    this((long)readerIdleTimeSeconds, (long)writerIdleTimeSeconds, (long)allIdleTimeSeconds, TimeUnit.SECONDS);
}
```



**eaderIdleTimeSeconds 读超时**：当在指定的时间间隔内没有从 `Channel` 读取到数据时，会触发一个 `READER_IDLE` 的 `IdleStateEvent` 事件。

**riterIdleTimeSeconds 写超时**：即当在指定的时间间隔内没有数据写入到 `Channel` 时，会触发一个 `WRITER_IDLE` 的 `IdleStateEvent` 事件。

**llIdleTimeSeconds 读/写超时**：即当在指定的时间间隔内没有读或写操作时，会触发一个 `ALL_IDLE` 的 `IdleStateEvent` 事件。



要实现 Netty 服务端心跳检测机制需要在服务器端的 `ChannelInitializer` 中加入如下的代码：

```Java
pipeline.addLast(new IdleStateHandler(3, 0, 0, TimeUnit.SECONDS));
```



## Future

![img](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-img081054572112678.png)



在并发编程中，我们通常会用到一组非阻塞的模型： Promise， Future 和 Callback。其中的 Future 表示一个**可能还没有实际完成的异步任务的结果**。针对这个结果可以添加 Callback 以便任务执行成功或失败后做出对应的操作。而 Promise 交由任务执行者，**任务执行者通过 Promise 可以标记任务完成或者失败**。

https://www.jianshu.com/p/a06da3256f0c

JDK 中的 Future 对象,其中只有一个 `isDone()` 方法判断一个异步操作是否完成，但是对于完成的定义过于模糊。（正常终止、抛出异常、用户取消）都会使 `isDone()` 方法返回 true。在我们的使用中，我们更有可能是对这三种情况分别处理

```Java
    // 取消异步操作
    boolean cancel(boolean mayInterruptIfRunning);
    // 异步操作是否取消
    boolean isCancelled();
    // 异步操作是否完成，正常终止、异常、取消都是完成
    boolean isDone();
    // 阻塞直到取得异步操作结果
    V get() throws InterruptedException, ExecutionException;
    // 同上，但最长阻塞时间为timeout
    V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException;
```

出于以上原因，Netty 扩展了 JDK 的 Future 接口。

```Java
    // 异步操作完成且正常终止
    boolean isSuccess();
    // 异步操作是否可以取消
    boolean isCancellable();
    // 异步操作失败的原因
    Throwable cause();
    // 添加一个监听者，异步操作完成时回调，类比javascript的回调函数
    Future<V> addListener(GenericFutureListener<? extends Future<? super V>> listener);
    Future<V> removeListener(GenericFutureListener<? extends Future<? super V>> listener);
    // 阻塞直到异步操作完成
    Future<V> await() throws InterruptedException;
    // 同上，但异步操作失败时抛出异常
    Future<V> sync() throws InterruptedException;
    // 非阻塞地返回异步结果，如果尚未完成返回null
    V getNow();
```

![image-20220903102758100](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20220903102758100.png)

可知， Future 对象有两种状态： 尚未完成 和 已经完成。其中 已完成又有三种状态 ： 成功、失败、用户取消。

`isSuccess` 查询是否执行成功

`isCancellable` 查询操作是否取消

`cause` 如果操作失败，给出抛出的异常（失败原因）

## CompletableFuture

### 简介

https://juejin.cn/post/6970558076642394142

Java 5 ， Future

我们在异步执行一个任务的时候，一般是用线程池 Executor 去创建。如果不需要有返回值，任务实现 Runnable 接口；如果需要有返回值，任务实现 Callable 接口，调用 Executor  的 submit 方法，再使用 Future 获取即可。

如果多个线程之间存在**依赖组合**，可以使用同步组件 CountDownLatch、CyclicBarrier 等，但是比较麻烦。

更简单的方法是使用 CompletableFuture

---

Future + 线程池异步配合，一定程度上也能提高程序的运行效率。

但是 Future 对于结果的获取不太友好，只能通过**阻塞**或者**轮询**的方式得到任务的结果

* Future.get() ： 阻塞调用，在线程获取结果之前会一直**阻塞**
* Future.isDone()：可以在程序中**轮询查询**这个方法的执行结果

阻塞的方式和异步编程的设计理念相违背，而轮询的方式会浪费 CPU 资源。

因此， JDK 8 设计出 CompletableFuture，提供了一种**观察者模式**类似的机制，可以让任务执行完成后通知监听的一方。

```Java
public class FutureTest {

    public static void main(String[] args) throws InterruptedException, ExecutionException, TimeoutException {

        UserInfoService userInfoService = new UserInfoService();
        MedalService medalService = new MedalService();
        long userId =666L;
        long startTime = System.currentTimeMillis();

        //调用用户服务获取用户基本信息
        CompletableFuture<UserInfo> completableUserInfoFuture = CompletableFuture.supplyAsync(() -> userInfoService.getUserInfo(userId));

        Thread.sleep(300); //模拟主线程其它操作耗时

        CompletableFuture<MedalInfo> completableMedalInfoFuture = CompletableFuture.supplyAsync(() -> medalService.getMedalInfo(userId)); 

        UserInfo userInfo = completableUserInfoFuture.get(2,TimeUnit.SECONDS);//获取个人信息结果
        MedalInfo medalInfo = completableMedalInfoFuture.get();//获取勋章信息结果
        System.out.println("总共用时" + (System.currentTimeMillis() - startTime) + "ms");

    }
}

```



CompletableFuture的 `supplyAsync` 方法，提供了异步执行的功能，线程池也不用单独创建了。实际上，它 CompletableFuture 使用了默认线程池是 **ForkJoinPool.commonPool**。



### 使用场景

#### 创建异步任务

Completable 创建异步任务，一般有 `supplyAsync`  和 `runAsync` 两个方法

*  `supplyAsync` : 执行 Completable  任务，支持返回值
*  `runAsync` ： 执行 Completable  任务，没有返回值



```Java
//使用默认内置线程池ForkJoinPool.commonPool()，根据supplier构建执行任务
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
//自定义线程，根据supplier构建执行任务
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)

```



```Java
//使用默认内置线程池ForkJoinPool.commonPool()，根据runnable构建执行任务
public static CompletableFuture<Void> runAsync(Runnable runnable) 
//自定义线程，根据runnable构建执行任务
public static CompletableFuture<Void> runAsync(Runnable runnable,  Executor executor)

```



---

```Java
public class FutureTest {

    public static void main(String[] args) {
        //可以自定义线程池
        ExecutorService executor = Executors.newCachedThreadPool();
        //runAsync的使用
        CompletableFuture<Void> runFuture = CompletableFuture.runAsync(() -> System.out.println("run,关注公众号:捡田螺的小男孩"), executor);
        //supplyAsync的使用
        CompletableFuture<String> supplyFuture = CompletableFuture.supplyAsync(() -> {
                    System.out.print("supply,关注公众号:捡田螺的小男孩");
                    return "捡田螺的小男孩"; }, executor);
        //runAsync的future没有返回值，输出null
        System.out.println(runFuture.join());
        //supplyAsync的future，有返回值
        System.out.println(supplyFuture.join());
        executor.shutdown(); // 线程池需要关闭
    }
}
//输出
run,关注公众号:捡田螺的小男孩
null
supply,关注公众号:捡田螺的小男孩捡田螺的小男孩

```

## 工作窃取算法

工作中，我们经常会用到线程池。通常是任务产生后放到一个任务队列，线程池中的线程不断从任务队列中取任务执行。

但是这种设计在一些情况下不是最优的。

若是一个非常耗时的大任务（比如大数组排序），就可能出现线程池中只有一个线程在处理这个大任务。而其他线程却空闲着。**最主要的是，若是多核服务器，只有一核在疯狂计算，其他处理器（CPU）无法援助繁忙的处理器（CPU）。**

更常见的实现是基于工作窃取的线程池。

工作窃取（work-stealing）算法是指某个线程从其他队列里窃取任务来执行。工作窃取的运行流程图如下

![在这里插入图片描述](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-img16babea6cba89734tplv-t2oaga2asx-zoom-in-crop-mark3024000.awebp)

被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。

工作窃取算法的优点是充分利用线程进行并行计算，并减少了线程间的竞争，其缺点是在某些情况下还是存在竞争，比如双端队列里只有一个任务时。并且消耗了更多的系统资源，比如创建多个线程和多个双端队列。

即，基于work-strealing的线程池并不是在所有情况下都是最优的，应用它的最佳情景是**线程池工作负荷比较重**，外部客户大量提交任务到线程池中。



## ChannelHandler 生命周期

https://www.cnblogs.com/yuanged/p/12595977.html











## Netty 高性能的原因

Netty 作为异步事件驱动，响应式编程的框架，高性能之处主要来自于其 **IO 模型**、 **线程处理模型**、异步处理以及零拷贝。前者决定如何收发数据，后者决定如何处理数据。

### IO 模型

---

传统Nio ：

在 I/O 复用模型中，会用到 Select，这个函数也会使**进程阻塞**，但是和阻塞 I/O 所不同的是这两个函数可以同时阻塞多个 I/O 操作。

----

**Netty = Nio + 响应式**

一个 I/O 线程可以并发处理 N 个客户端连接和读写操作，并且不需要每次都进行用户态内核态轮询是否有新的连接需要处理，而是基于事件的响应式编程，每当有新的 IO 事件，会通知 selector 进行处理，**减少了用户态内核态切换开销**



---

### 线程模型

主从 Reactors 多线程模型。

bossGroup负责accept，worker负责读写。使用的时候一般再定义一个线程池专门负责处理具体业务使得IO线程和业务线程隔离

----

### 异步处理

Netty 内部的异步操作

异步的概念和同步相对。当一个异步过程调用发出后，调用者不能立刻得到结果。实际处理这个调用的部件在完成后，通过状态、通知和回调来通知调用者。

在 Netty 中所有的 IO 操作都是异步的，包括 Bind、Write、Connect 等操作会简单的返回一个 ChannelFuture。调用者并不能立刻获得结果，而是通过 Future-Listener 机制，用户可以方便的主动获取或者通过通知机制获得 IO 操作结果。

相比传统阻塞 I/O，执行 I/O 操作后线程会被阻塞住, 直到操作完成；**异步处理的好处是不会造成线程阻塞，线程在 I/O 操作期间可以执行别的程序，在高并发情形下会更稳定和更高的吞吐量。**

### 零拷贝

https://cloud.tencent.com/developer/article/1488088

Netty中的零拷贝与操作系统层面上的零拷贝不完全一样, Netty的零拷贝完全是在用户态(Java层面)的，更多是数据操作的优化。

ByteBuf 堆外内存，减少了一次到 jvm 虚拟机的数据拷贝过程

Netty的零拷贝主要体现在五个方面

1. Netty的接收和发送ByteBuffer使用直接内存进行Socket读写，不需要进行字节缓冲区的二次拷贝。**如果使用JVM的堆内存进行Socket读写，JVM会将堆内存Buffer拷贝一份到直接内存中，然后才写入Socket中**。相比于使用直接内存，消息在发送过程中多了一次缓冲区的内存拷贝。
2. Netty的文件传输调用FileRegion包装的transferTo方法，可以直接将文件缓冲区的数据发送到目标Channel，避免通过循环write方式导致的内存拷贝问题。
3. Netty提供CompositeByteBuf类, 可以将多个ByteBuf合并为一个逻辑上的ByteBuf, 避免了各个ByteBuf之间的拷贝。
4. 通过wrap操作, 我们可以将byte[]数组、ByteBuf、ByteBuffer等包装成一个Netty ByteBuf对象, 进而避免拷贝操作。
5. ByteBuf支持slice操作，可以将ByteBuf分解为多个共享同一个存储区域的ByteBuf, 避免内存的拷贝。









