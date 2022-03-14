[toc]
# I/O 相关基础知识
## 网络编程基本模型

> 传输的是 IP 地址（&端口） + 需要传输的数据

客户端应用程序
⬇️
客户端操作系统
⬇️
客户端网卡
⬇️
服务器网卡
⬇️
网卡驱动：驱动程序（Device Driver）就是CPU控制和使用网卡的程序
⬇️
服务器操作系统
⬇️
TCP 模块➡️每个连接，对应一个名为「TCP 缓存区」的内存空间
⬇️
JVM
⬇️根据「端口号」找到
Java 应用服务（绑定了端口号）

## 同步 | 异步 | 阻塞 | 非阻塞

* 同步 Synchrony
    * When you execute something synchronously, you *wait for it to finish before moving on* to another task
    * 两个同步任务相互依赖，并且一个任务必须以依赖于另一任务的某种方式执行。 比如在`A->B`事件模型中，你需要先完成 A 才能执行 B
    * 再换句话说，同步调用者被调用者未处理完请求之前，调用不返回，调用者会一直等待结果的返回

* 异步 Asynchrony
    * When you execute something asynchronously, *you can move on to another task before it finishes*
    * 两个异步的任务完全独立的，一方的执行不需要等待另外一方的执行
    * 异步调用种一调用就返回结果不需要等待结果返回，当结果返回的时候通过回调函数或者其他方式拿着结果再做相关事情

* 阻塞： 阻塞就是发起一个请求，调用者一直等待请求结果返回，也就是当前线程会被挂起，无法从事其他任务，只有当条件就绪才能继续。

* 非阻塞： 非阻塞就是发起一个请求，调用者不用一直等着结果返回，可以先去干其他事情。

同步 / 异步是从行为角度描述事物的，而阻塞和非阻塞描述的当前事物的状态（等待调用结果时的状态）。

# BIO
## BIO 处理数据

Java 处理数据分为下面几个步骤

1. 绑定端口（比如 8080 端口）：`ServerSocket serverSocket = new ServerSocket(8080);`

2. 获取新连接，并映射为 Socket 对象：`Socket socketConnection = serverSocket.accept();`

3. 因为数据在操作系统的 TCP 缓冲区内，所以要读取连接中传递的数据。

    * 先准备一个存放数据的容器：`byte[] request = new byte[1024];`

    * 然后读取连接里面的数据：`socketConnection.getInputStream().read(request);`

4. 响应内容……

## 理解 BIO 的 Blocking 及其问题

上面的代码，只能进行一个连接，因为不止有一个连接，所以需要循环创建连接（因为线程资源宝贵，所以使用的是线程池）。

为了不出现死循环，所以有「阻塞 Blocking」。

* 也就是说，如果没有新的连接，就不会跑代码。

* 如果连接已经建立好，但是数据还没到，也会不跑下面的代码（阻塞 Blocking）

> 阻塞的实现来自于 native，由 JVM 分配

BIO 的问题：

* 假设出现信号不好的情况，一个连接已经建立了，可是数据迟迟没有传输过来，那么，这个连接就会一直处于 blocking 状态，占用线程资源。

# NIO

New I/O 或者 Non Blocking I/O

## NIO 的特性 / NIO 与 IO（BIO）的区别

1. IO 流是阻塞的，NIO 流是不阻塞的（Non-Blocking IO）
    * Java IO 的各种流是阻塞 Blocking 的。多线程下，为了保证线程的安全，只要没有新的连接，下面的操作数据的代码就不会执行，处于 Blocking 状态。
    * 除此之外，在建立了 I/O 连接后，当一个线程调用 `read()` 或 `write()` 时，为了保证其他线程不干扰，该线程也会一直处于 Blocking 状态，直到有数据被读取或完全写入。该线程在此期间处于停滞状态，无法完成其他操作。
    * Java NIO 是 Non-Blocking IO 的操作。在线程从通道 Channel 读取 `read` 数据到 buffer 时，即便没有数据传输，还可以继续做别的事情。
    * 当数据到达 buffer 中后，线程再继续处理刚刚读取到数据的这个 buffer。
    * 写数据也是一样的。一个线程请求写入一些数据到某通道，并不需要等待它完全写入，就可以去做别其他的事情。
    
2. IO 面向流 (Stream oriented)，而 NIO 面向缓冲区 (Buffer oriented)
     * Buffer 是一个对象，它包含一些要写入或者要读出的数据。
    * 在 Stream Oriented 面向流的 I/O 中，数据直接 `read` 或 `write` 到 Stream 对象中。
    * 虽然 Stream 中也有 Buffer 开头的扩展类，但只是流的包装类，还是从流读到缓冲区，而 NIO 却是直接在 Buffer 中进行操作。
    * 在 NIO 读取数据时，它是直接读到缓冲区中的; 在写入数据时，写入到缓冲区中。任何时候访问 NIO 中的数据，都是通过缓冲区进行操作。
    * 最常用的缓冲区是 ByteBuffer, 一个 ByteBuffer 提供了一组功能用于操作 byte 数组。除了 ByteBuffer, 还有其他的一些缓冲区，事实上，每一种 Java 基本类型（除了 Boolean 类型）都对应有一种缓冲区。

3. NIO 通过 Channel（通道） 进行读写
    * 通道 Channel 是双向的，可读也可写
    * 流 Stream 的读写是单向的
    * 无论读写，通道 Channel 只和 Buffer 交互
    * 因为 Buffer，通道 Channel 可以异步地 asynchronously 读写

4. Selector (选择器)
    * NIO 有选择器，而 IO 没有
    * 选择器用于使用单个线程 Thread 处理多个通道 Channel。因此，它需要较少的线程来处理这些通道。
    * 线程之间的切换对于操作系统来说是昂贵的。 因此，为了提高系统效率选择器是有用的。

## NIO 读数据和写数据方式及核心组件

NIO 中的所有 IO 都是从 Channel（通道） 开始的。

从通道进行数据读取 ：创建一个缓冲区，然后请求通道读取数据。

从通道进行数据写入 ：创建一个缓冲区，填充数据，并要求通道写入数据。

NIO 包含下面几个核心的组件：

* Channel(通道)
* Buffer(缓冲区)
* Selector(选择器)

整个 NIO 体系包含的类远远不止这三个，只能说这三个是 NIO 体系的 “核心 API”。

## NIO 处理数据的方法

```java
import java.io.*;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;
import java.util.Set;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadPoolExecutor;

public class NIO {

    //处理请求的线程池
    public static final ThreadPoolExecutor THREAD_POOL = (ThreadPoolExecutor) Executors.newCachedThreadPool();

    public static void main(String[] args) throws IOException {

        //打开 Server 的 Socket Channel（默认是阻塞模式）
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        /**
         * 也可以像下面这样设置为非阻塞
         * serverSocketChannel.configureBlocking(false);
         */

        //绑定端口（到时候访问 http://localhost:8080）
        serverSocketChannel.bind(new InetSocketAddress(8080));

        /*
        JDK 封装了所有操作系统的多路复用功能
        使用「选择器 Selector」，来选择程序需要处理的连接
        被称为"事件机制"
        */
        Selector selector = Selector.open();

        //不断获取连接
        while (true) {
            //获取新连接，并映射为 SocketChannel 对象（Java 面向对象）
            SocketChannel socketChannel = serverSocketChannel.accept();

            //默认还是 Blocking 模式，这里要手动设置为 Non Blocking 模式
            socketChannel.configureBlocking(false);
            /**
             * 假设前面把 serverSocketChannel 设置成了「非阻塞」
             * "serverSocketChannel.configureBlocking(false);"
             *
             * 那么，当前的「SocketChannel socketChannel」可能会返回 null：
             * 1、在 Non Blocking 模式下，有新连接会返回连接；
             * 2、没有连接的话，可能 socketChannel 会返回 null
             *
             * 这种情况下，要加上：
             * if (socketChannel == null) {
             *     continue;
             * }
             */


            /*
             但是，有连接不代表有请求 request
             可能存在没数据传输却浪费线程的情况
             所以让连接去注册 Selector 的「事件机制」
             */
            socketChannel.register(selector, SelectionKey.OP_READ);
            //SelectionKey.OP_READ 代表「"连接"发生了 read 读取事件」

            //注册好了后，要获取对应的"事件通知"（这里是获取 read 事件）
            selector.select();
            //这里先得到所有的"事件"
            Set<SelectionKey> eventKeys = selector.selectedKeys();

            //循环所有"事件"
            Iterator<SelectionKey> iterator = eventKeys.iterator();
            while (iterator.hasNext()) {
                //得到每个"事件"
                SelectionKey event = iterator.next();

                //如果这个事件是 Readable（可以 read 的事件），才开始读取
                if (event.isReadable()) {

                    //遇到 read 读取事件后，建立新的连接，这次的连接才是有效的连接
                    SocketChannel socketConnection = (SocketChannel) event.channel();

                    //向线程池提交，获取线程
                    THREAD_POOL.submit(() -> {

                        //每次分配（allocate）1024 个字符的数组用于存储数据
                        //ByteBuffer 可以理解为「数组」的高级封装，用于接收并方便后续操作数据
                        //超过 1024 的部分，JVM 会自动处理（分开几次传输）
                        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

                        //读取连接里面的数据
                        socketConnection.read(byteBuffer);//这里的读取 read 也是「非阻塞」


                        //本来是 ByteBuffer 用于 read
                        //现在转换为 write 模式，可以进行 output 数据的操作
                        byteBuffer.flip();

                        //这里打印一下里面的数据，也可以不进行
                        System.out.println(new String(byteBuffer.array()));
                        //上面的 ByteBuffer 是 Non Blocking，有可能里面没有数据

                        //TODO 响应内容的代码……
                        return null;
                    });
                }
            }
        }
    }
}
```

# Java 中的 IO 流
## Java 中 IO 流分为几种

* 按照流的流向分，可以分为输入流和输出流；
* 按照操作单元划分，可以划分为字节流和字符流；
* 按照流的角色划分为节点流和处理流。

Java IO 流的 40 多个类都是从如下 4 个抽象类基类中派生出来的。

* InputStream/Reader: 所有的输入流的基类，前者是字节输入流，后者是字符输入流。
* OutputStream/Writer: 所有输出流的基类，前者是字节输出流，后者是字符输出流。

# 既然有了字节流, 为什么还要有字符流?

* 问题本质想问：不管是文件读写还是网络发送接收，信息的最小存储单元都是字节，那为什么 I/O 流操作要分为字节流操作和字符流操作呢？

* 回答：字符流是由 Java 虚拟机将字节转换得到的，问题就出在这个过程还算是非常耗时，并且，如果我们不知道编码类型就很容易出现乱码问题。所以， I/O 流就干脆提供了一个直接操作字符的接口，方便我们平时对字符进行流操作。**如果音频文件、图片等媒体文件用字节流比较好，如果涉及到字符的话使用字符流比较好**。

BIO,NIO,AIO 有什么区别?

* BIO (Blocking I/O): 同步阻塞 I/O 模式，数据的读取写入必须阻塞在一个线程内等待其完成。在活动连接数不是特别高（小于单机 1000）的情况下，这种模型是比较不错的，可以让每一个连接专注于自己的 I/O 并且编程模型简单，也不用过多考虑系统的过载、限流等问题。线程池本身就是一个天然的漏斗，可以缓冲一些系统处理不了的连接或请求。但是，当面对十万甚至百万级连接的时候，传统的 BIO 模型是无能为力的。因此，我们需要一种更高效的 I/O 处理模型来应对更高的并发量。

* NIO (New I/O): NIO 是一种同步非阻塞的 I/O 模型，在 Java 1.4 中引入了 NIO 框架，对应 java.nio 包，提供了 Channel , Selector，Buffer 等抽象。NIO 中的 N 可以理解为 Non-blocking，不单纯是 New。它支持面向缓冲的，基于通道的 I/O 操作方法。 NIO 提供了与传统 BIO 模型中的 Socket 和 ServerSocket 相对应的 SocketChannel 和 ServerSocketChannel 两种不同的套接字通道实现, 两种通道都支持阻塞和非阻塞两种模式。阻塞模式使用就像传统中的支持一样，比较简单，但是性能和可靠性都不好；非阻塞模式正好与之相反。对于低负载、低并发的应用程序，可以使用同步阻塞 I/O 来提升开发速率和更好的维护性；对于高负载、高并发的（网络）应用，应使用 NIO 的非阻塞模式来开发

* AIO (Asynchronous I/O): AIO 也就是 NIO 2。在 Java 7 中引入了 NIO 的改进版 NIO 2, 它是异步非阻塞的 IO 模型。异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。AIO 是异步 IO 的缩写，虽然 NIO 在网络操作中，提供了非阻塞的方法，但是 NIO 的 IO 行为还是同步的。对于 NIO 来说，我们的业务线程是在 IO 操作准备好时，得到通知，接着就由这个线程自行进行 IO 操作，IO 操作本身是同步的。查阅网上相关资料，我发现就目前来说 AIO 的应用还不是很广泛，Netty 之前也尝试使用过 AIO，不过又放弃了。

