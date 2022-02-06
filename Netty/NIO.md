## NIO

### **IO模型主要分类：**

-  同步(synchronous) IO 和异步(asynchronous) IO
-  阻塞(blocking) IO 和非阻塞(non-blocking)IO
-  同步阻塞(blocking-IO)简称 BIO
-  同步非阻塞(non-blocking-IO)简称 NIO
-  异步非阻塞(synchronous-non-blocking-IO)简称 AIO

**1.BIO (同步阻塞 I/O 模式)**

数据的读取写入必须阻塞在一个线程内等待其完成。

这里使用那个经典的烧开水例子，这里假设一个烧开水的场景，有一排水壶在烧开水，BIO 的工作模式就是， 叫一个线程停留在一个水壶那，直到这个水壶烧开，才去处理下一个水壶。但是实际上线程在等待水壶烧开的时间段什么都没有做。

**2.NIO（同步非阻塞）**

同时支持阻塞与非阻塞模式，但这里我们以其同步非阻塞 I/O 模式来说明，那么什么叫做同步非阻塞？如果还拿烧开水来说，NIO 的做法是叫一个线程不断的轮询每个水壶的状态，看看是否有水壶的状态发生了改变，从而进行下一步的操作。

**3.AIO （异步非阻塞I/O模型）**

异步非阻塞与同步非阻塞的区别在哪里？异步非阻塞无需一个线程去轮询所有 IO 操作的状态改变，在相应的状态改变后，系统会通知对应的线程来处理。对应到烧开水中就是，为每个水壶上面装了一个开关，水烧开之后，水壶会自动通知我水烧开了。

**4.IO 与 NIO 区别**

|   IO   |    NIO     |
| :----: | :--------: |
| 面向流 | 面向缓冲区 |
| 阻塞IO |  非阻塞IO  |
|   无   |   选择器   |

**5.同步与异步的区别**：

同步：发送一个请求，等待返回，再发送下一个请求，同步可以避免出现死锁，脏读的发生。

异步：发送一个请求，不等待返回，随时可以再发送下一个请求，可以提高效率，保证并发。

**6.阻塞和非阻塞**

阻塞：传统的 IO 流都是阻塞式的。也就是说，当一个线程调用 read() 或者 write() 方法时，该线程将被阻塞，直到有一些数据读读取或者被写入，在此期间，该线程不能执行其他任何任务。在完成网络通信进行 IO 操作时，由于线程会阻塞，所以服务器端必须为每个客户端都提供一个独立的线程进行处理，当服务器端需要处理大量的客户端时，性能急剧下降。

非阻塞：Java NIO 是非阻塞式的。当线程从某通道进行读写数据时，若没有数据可用时，该线程会去执行其他任务。线程通常将非阻塞 IO 的空闲时间用于在其他通道上执行 IO 操作，所以单独的线程可以管理多个输入和输出通道。因此 NIO 可以让服务器端使用一个或有限几个线程来同时处理连接到服务器端的所有客户端。

**7.BIO、NIO、AIO 适用场景**

-  BIO 方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4 以前的唯一选择。
-  NIO 方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，并发局限于应用中，编程比较复杂。
-  AIO 方式使用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用OS参与并发操作，编程比较复杂，JDK7 开始支持。

[阻塞与非阻塞  同步与异步](https://mp.weixin.qq.com/s/8qq6yjZgYex0YF_Am2RlgA)

## **NIO 的 3 个核心概念**

NIO 重点是把 **Channel（通道），Buffer（缓冲区），Selector（选择器）**三个之间的关系弄清楚。

### **1.缓冲区 Buffer**

Buffer 是一个对象。它包含一些要写入或者读出的数据。在面向流的 I/O 中，可以将数据写入或者将数据直接读到 Stream 对象中。

**在 NIO 中，所有的数据都是用缓冲区处理。**这也就本文上面谈到的 **IO 是面向流的，NIO 是面向缓冲区的。**

缓冲区实质是一个数组，通常它是一个字节数组（ByteBuffer），也可以使用其他类的数组。但是一个缓冲区不仅仅是一个数组，缓冲区提供了对数据的结构化访问以及维护读写位置（limit）等信息。

最常用的缓冲区是 ByteBuffer，一个 ByteBuffer 提供了一组功能于操作 byte 数组。除了 ByteBuffer，还有其他的一些缓冲区，事实上，每一种 Java 基本类型（除了 Boolean）都对应一种缓冲区。

#### ByteBuffer 核心变量

Buffer 类维护了 4 个核心变量属性来提供**关于其所包含的数组的信息**。它们是：

- 容量 Capacity

  **缓冲区能够容纳的数据元素的最大数量**。容量在缓冲区创建时被设定，并且永远不能被改变。(不能被改变的原因也很简单，底层是数组嘛)

- 上界 Limit

  **缓冲区里的数据的总数**，代表了当前缓冲区中一共有多少数据。

- 位置 Position

  **下一个要被读或写的元素的位置**。Position 会自动由相应的 `get( )`和 `put( )`函数更新。

- 标记 Mark

  一个备忘位置。**用于记录上一次读写的位置**。

若要从缓冲区获取数据，需要调用`flip()`方法，**切换成读模式**，这个方法可以改动 position 和 limit 的位置：

```java
    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
```

若要想缓冲区写数据，需要调用`clear()`方法，**切换成写模式**,这个函数会“清空”缓冲区：

```java
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
```

> 注意：这里的“清空”缓冲区是核心变量回归”写模式“，缓冲区中的数据是没有清空的，但被”遗忘了“

​	

#### **2.通道 Channel**

Channel 是一个通道，可以通过它读取和写入数据，他就像自来水管一样，网络数据通过 Channel 读取和写入。

通道和流不同之处在于通道是双向的，流只是在一个方向移动，而且通道可以用于读，写或者同时用于读写。

因为 Channel 是全双工的，所以它比流更好地映射底层操作系统的 API，特别是在 UNIX 网络编程中，底层操作系统的通道都是全双工的，同时支持读和写。



**Channel 有四种实现：**

-  FileChannel:是从文件中读取数据。
-  DatagramChannel:从 UDP 网络中读取或者写入数据。
-  SocketChannel:从 TCP 网络中读取或者写入数据。
-  ServerSocketChannel:允许你监听来自 TCP 的连接，就像服务器一样。每一个连接都会有一个 SocketChannel 产生。



#### **3.多路复用器 Selector**

Selector 选择器可以监听多个 Channel 通道感兴趣的事情(read、write、accept(服务端接收)、connect），实现一个线程管理多个 Channel，节省线程切换上下文的资源消耗。Selector 只能管理**非阻塞**的通道，FileChannel 是阻塞的，无法管理。



**关键对象**

-  Selector：选择器对象，通道注册、通道监听对象和 Selector 相关。
-  SelectorKey：通道监听关键字，通过它来监听通道状态。

**监听注册**

监听注册在 Selector

> socketChannel.register(selector, SelectionKey.OP_READ);

**监听的事件有**

-  OP_ACCEPT: 接收就绪，serviceSocketChannel 使用的
-  OP_READ: 读取就绪，socketChannel 使用
-  OP_WRITE: 写入就绪，socketChannel 使用
-  OP_CONNECT: 连接就绪，socketChannel 使用



## 参考资料

[1] [高并发编程系列：NIO、BIO、AIO的区别，及NIO的应用和框架选型](https://youzhixueyuan.com/java-nio-introduce.html)

[2] [尚硅谷Netty视频教程](https://www.bilibili.com/video/BV1DJ411m7NR?p=11)

[3] [JDK10都发布了，nio你了解多少？](https://mp.weixin.qq.com/s?__biz=MzI4Njg5MDA5NA==&mid=2247484235&idx=1&sn=4c3b6d13335245d4de1864672ea96256&chksm=ebd7424adca0cb5cb26eb51bca6542ab816388cf245d071b74891dd3f598ccd825f8611ca20c&token=1834317504&lang=zh_CN&scene=21#wechat_redirect)

[4] [阻塞与非阻塞 同步与非同步](https://mp.weixin.qq.com/s/8qq6yjZgYex0YF_Am2RlgA)