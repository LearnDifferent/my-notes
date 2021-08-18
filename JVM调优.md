# Garbage Collection

## GC Basics

Automatic Garbage Collection: 

- Automatic garbage collection is the process of looking at heap memory, identifying which objects are in use and which are not, and deleting the unused objects
- An in use object, or a referenced object, means that some part of your program still maintains a pointer to that object. 
- An unused object, or unreferenced object, is no longer referenced by any part of your program. So the memory used by an unreferenced object can be reclaimed

In Java, process of deallocating memory is handled automatically by the garbage collector. The basic process can be described as follows:

- 1: Marking
	- The first step in the process is called marking. This is where the garbage collector identifies which pieces of memory are in use and which are not
- 2a: Normal Deletion
	- Normal deletion removes unreferenced objects leaving referenced objects and pointers to free space.
- 2b: Deletion with Compacting 
	- To further improve performance, in addition to deleting unreferenced objects, you can also compact the remaining referenced objects.
	- By moving referenced object together, this makes new memory allocation much easier and faster

参考资料：[Java Garbage Collection Basics](https://www.oracle.com/technetwork/tutorials/tutorials-1873457.html)

## GC Roots

GC Roots：

* GC Roots 是指一组必须活跃的引用。
* 使用 GC Roots /“可达性分析算法”来判断对象是否存活，只要 GC Roots 能找到该对象，就说明存活，反之则不存活。

GC Roots 基本思路：

* 通过一系列名为「GCRoots」的对象作为起始点，从这个被称为 GC Roots 的对象开始向下搜索，如果一个对象到 GC Roots 没有任何 Reference Chain（引用链）相连时，则说明此对象不可用
* 即给定一个集合的引用作为根出发，通过引用关系遍历对象图，能被遍历到的（可到达的）对象就被判定为存活，没有被遍历到的就自然被判定为死亡
* 更多 GC Roots 内容，可以查阅 [JVM之GCRoots概述](https://blog.csdn.net/weixin_41910694/article/details/90706652)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d053c7d6f7724a11a32ecda6c7104b2e~tplv-k3u1fbpfcp-zoom-1.image)

**GC Roots 是 Reference Variables**（用于 point objects or values），会**指向 Heap 中的 Instance Objects**，且<u>不会被回收</u>。具体的 GC Roots 有：

* JVM Stack 的 Stack Frame 中的 Local Variables
* Metaspace 中 `static` 类静态属性
* Metaspace 中 `final` 常量
* Native Method Stack 中的 JNI（用于调用 Native 方法）

[GC Roots 示例](https://www.cnblogs.com/rumenz/articles/14099927.html)：

```java
public class Rumenz{
    public static void main(String[] args) {
        Rumenz a = new Rumenz();
        a = null;
    }
}
```

`a` 是 Stack 中的 Frame 的 Local Variable， `a` 储存的是 `new` 出来的 `Rumenz` 在 Heap 中的内存地址。

也就是说，Local Variable `a` 是 Object `Rumenz` 的 Reference Variable，所以 `a` 是 GC Root。

当 `a=null` 时，`a` 存储的就不再是 `Rumenz` 所在的内存地址了，所以 `Rumenz` 没有指向 `a` 这个 GC Root，`Rumenz` 将会被回收。

***

```java
class Rumenz {
    public static Rumenz rum;

    public static void main(String[] args) {
        Rumenz a = new Rumenz(); // 这个 Rumenz 会被回收
        a.rum = new Rumenz(); // 这个 Rumenz 不会被回收
        a = null;
    }
}
```

上面的代码中，`Rumenz a = new Rumenz();` 和 `a = null;` ，说明 `a` 一开始指向的 `Rumenz` 会被回收。

代码中的 `a.rum` ，表示 `Rumenz` 类型的 `a` 的类静态属性（Class 的全局静态变量） `rum`，所以 `a.rum` 是 GC Root。

所以 `a.rum = new Rumenz();` 中，`a.rum` 指向的 `Rumenz` 不会被回收。

***

```java
public class Rumenz{
    public final Rumenz r = new Rumenz(); // 这个 Rumenz 不会被回收

    public static void main(String[] args){
        Rumenz a = new Rumenz(); // 这个 Rumenz 会被回收
        a = null;
    }
}
```

`public final Rumenz r = new Rumenz();` 中，`final` 常量 `r` 指向的 `Rumenz` 不会被回收。

## GC Algorithms

**GC 常用算法**

Copying（复制算法）：

* 步骤：
	1. 将空间分位 2 个区域，其中一个区域存放 Objects，另一个区域为空
	2. 扫描存放了 objects 的当前区域，将其中的 live objects 紧凑地 copy 到另一个区域，然后销毁当前区域的所有的 objects
	3. 交替两个区域的功能角色，等待下次 GC
* 优点：开销比较小，没有内存碎片
* 缺点：需要 2 倍的内存空间，而且其中一个空间会被浪费
* 使用场景：Objects 的存活度较低的 Young Gen

Mark-Sweep（标记清除）：

* 步骤：
	1. 扫描区域内的所有 objects，mark（标记）所有 live objects
	2. 定点 sweep（清除）没有被标记的 objects
* 优点：不需要额外的空间
* 缺点：
	* 两次扫描严重浪费时间，会产生 fragmentation（内存碎片），导致后续需要存放 Humorous Object 的时候，无法分配连续的内存空间
	* Mark 和 Sweep 两个过程的效率都不高

Mark-Compact（标记压缩/标记整理）：

* 步骤：
	1. Mark reachable objects（标记 live objects）
	2. Then a compacting step relocates the marked objects towards the beginning of the heap area（将 live objects 压缩到一段连续的内存空间中）
	3. 除了存放 live objects 的那段连续的内存空间，其余空间的 Objects 全部销毁掉
* 使用场景：有较多 live objects 的 Old Gen

---

GC Algorithms 的简单比较：

- Copying 的效率最高，但内存利用率不及 Mark-Compact 和 Mark-Sweep
- 内存整齐度：Copying = Mark-Compact > Mark-Sweep

Generational Collection：

- Young Gen：Objects 存活率低，所以 live objects 比较少，使用高效的 Copying 所需要的“复制对象”的成本，就会相应降低
- Old Gen：Objects 存活率高，而且因为本身占据的空间就很大了，没有额外的空间去完成 Copying，所以混合使用 Mark-Sweep + Mark-Compact（所以，这里也是调优的重点）



# JVM Garbage Collectors

主要参考资料：

- [JVM Garbage Collectors](https://www.baeldung.com/jvm-garbage-collectors) 
- [Java Platform, Standard Edition HotSpot Virtual Machine Garbage Collection Tuning Guide](https://docs.oracle.com/javase/9/gctuning/toc.htm)
- [【java】垃圾收集器|g1收集器](https://www.bilibili.com/video/BV13J411g7A1) 和 [java_jvm垃圾收集器.md](https://github.com/sunwu51/notebook/blob/master/19.09/java_jvm%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8.md)

JVM has these types of GC implementations:

## Serial & ParNew & Serial Old

Serial：

- 关键词：<u>Young Gen</u>，<u>单线程</u>，<u>Copying（复制算法）</u>，<u>STW</u>，<u>配合 Serial Old 使用</u>
- The serial collector（串行回收器） uses a single thread to perform all garbage collection work, which makes it relatively efficient because there is no communication overhead between threads.
- It's <u>best-suited to single processor machines</u> and be useful on multiprocessors for applications with small data sets (up to approximately 100 MB). 
- 开启：`-XX:+UseSerialGC`

Serial Old：

- 关键词：<u>Old Gen</u>，<u>单线程</u>，<u>Mark-Compact（标记-压缩算法）</u>，<u>STW</u>，<u>配合 Serial 使用</u>
- Serial Old 和 Parallel Old 都是 Mark-Compact 算法，不会产生 fragment（内存碎片）

ParNew：

- 关键词：<u>Young Gen</u>，<u>多线程</u>，<u>Copying（复制算法）</u>，<u>STW</u>，<u>配合 CMS 使用</u>
- ParNew（并行）是 Serial 的多线程版，可以充分的利用CPU资源，减少回收的时间
- 开启： `-XX:+UseParNewGC`

## Parallel Scavenge & Parallel Old

Parallel Old：

- 关键词：<u>Old Gen</u>，<u>多线程</u>，<u>Mark-Compact（标记压缩算法）</u>，<u>STW</u>，<u>和 Parallel Scavenge 绑定使用</u>
- Parallel Old 是 Parallel Scavenge 的 Old Gen 版本
- Parallel Old 和 Serial Old 都是 Mark-Compact 算法，不会产生 fragment（内存碎片）

Parallel Scavenge：

-  关键词：<u>Young Gen</u>，<u>多线程</u>，<u>Copying（复制算法）</u>，<u>STW</u>，<u>配合 Parallel Old 使用</u>
-  The parallel collector is also known as *throughput collector* , [because its main goal is to maximize overall throughput of the application](https://dzone.com/articles/gc-explained-parallel-collector)
-  Parallel Scavenge 是 throughput（吞吐量）优先的回收器，高吞吐量则可以高效率地利用 CPU 时间，尽快完成程序的运算任务，主要适合在后台运算而不需要太多交互的任务
-  The parallel collector is intended for applications with medium-sized to large-sized data sets that are <u>run on multiprocessor or multithreaded hardware</u>.
-  The numbers of garbage collector threads can be controlled with `-XX:ParallelGCThreads`
-  You can enable Parallel Scavenge Collector by using the `-XX:+UseParallelGC` option.

关于 Throughput：

- [スループット](https://ja.wikipedia.org/wiki/%E3%82%B9%E3%83%AB%E3%83%BC%E3%83%97%E3%83%83%E3%83%88)（実効伝送速度）は、一般に単位時間当たりの処理能力のこと
- Throughput 指 <u>application threads 的用时</u> 占据 <u>程序总用时</u> 的比例
- 比如：<u>Throughput 为 99% 表示程序运行了 100  秒，而 application threads 运行了 99 秒，GC 线程运行运行了 1 秒</u>
- [high throughput](https://docs.oracle.com/cd/E13150_01/jrockit_jvm/jrockit/geninfo/diagnos/tune_app_thruput.html) means that more transactions are executed during a given amount of time. You can also measure the throughput by measuring how long it takes to perform a specific task or calculation.

铺垫一下知识： [High Throughput VS Shorter GC Pause](https://zhuanlan.zhihu.com/p/100265755) ：

- High Throughput 的好处让资源最大限度地用于“生产性工作”
- Shorter GC Pause 可以提高客户端的用户体验，缩短因为 application threads 而造成的停顿时间
- 每次启动并运行 GC 线程的时候，都非常消耗 CPU 资源，所以<u>想要实现 High Throughput，就要减少 GC 的次数</u>
- 如果<u>减少了 GC 的次数，就会让每次 GC 的工作量增大，由此造成更长时间的 GC Pause</u>
- 所以，想要<u>实现 Shorter GC Pause，需要增加 GC 的次数，让每个 GC 的工作量降低，从而让每次的 GC 都能更快完成任务</u>，减少 GC Pause 的时间

Parallel Scavenge 提供了和 throughput 相关的参数选项：

- The maximum GC Pause time goal (gap [in milliseconds] between two GC) is specified with `-XX:MaxGCPauseMillis` 
	- 该参数允许的值是一个大于 0 的毫秒数，Collector 将尽可能地保证 GC Pause 的时间不超过设定值
	- 代价就是 Shorter GC Pause decreases throughput
- The time spent doing garbage collection versus the time spent outside of garbage collection is called the maximum throughput target and can be specified by `-XX:GCTimeRatio`
	- 该参数可以决定 throughput（吞吐量）大小
	- [For example `-XX:GCTimeRatio=19` sets a goal of 5% of the total time for GC and throughput goal of 95%. That is, the application should get 19 times as much time as the collector.](https://docs.oracle.com/javase/7/docs/technotes/guides/vm/gc-ergonomics.html)
		- 当 `GCTimeRatio` 为 19 时，表示 <u>application threads 的运行时间</u>是 <u>GC 消耗的时间</u>的 19 倍
		- 假设 GC 耗时 5 秒，那么 application threads 的耗时就是 `5 x 19 = 95`  秒，总共耗时 `5 + 95 = 100` 秒，所以 Throughput 为 `95 / 100 = 95%` 
	- <u>By default the value is 99</u>. This was selected as *a good choice for server applications* . 
	- <u>A value that is too high will cause the size of the heap to grow to its maximum</u>.

Parallel Scavenge 其他默认的参数：

-  Parallel compaction is enabled by default if the option `-XX:+UseParallelGC` has been specified. You can disable it by using the `-XX:-UseParallelOldGC` option.
	-  Parallel compaction is a feature that enables the parallel collector to perform major collections in parallel. Without parallel compaction, major collections are performed using a single thread, which can significantly limit scalability. 
-  默认会使用 `-XX:UseAdaptiveSizePolic` 参数，[它动态调整一些 size 参数](https://zhuanlan.zhihu.com/p/149879026)

## The Mostly Concurrent Collectors: CMS & G1

在学习 CMS 和 G1 之前，要先了解一下 mostly concurrent 的概念。下文摘抄自官方文档 - [7 The Mostly Concurrent Collectors](https://docs.oracle.com/javase/9/gctuning/mostly-concurrent-collectors.htm#JSGCT-GUID-DFA8AF9C-F3BC-4F12-99CE-45AB6F22F15A)。

The mostly concurrent collectors <u>perform parts of their work concurrently to the application</u>, hence their name. The Java HotSpot VM includes two mostly concurrent collectors:

- Concurrent Mark Sweep (CMS) collector: This collector is for applications that prefer shorter garbage collection pauses and can afford to share processor resources with the garbage collection.
- Garbage-First (G1) garbage collector: This server-style collector is for multiprocessor machines with a large amount of memory. It meets garbage collection pause-time goals with high probability while achieving high throughput.

The mostly concurrent collector <u>trades processor resources</u> (which would otherwise be available to the application) <u>for shorter major collection pause time</u>.

The most visible overhead is the use of one or more processors during the concurrent parts of the collection. On an *N* processor system, the concurrent part of the collection uses *K*/*N* of the available processors, where 1 <= *K* <= ceiling*{N/4*} （取 *N/4* 的上限）.

>[举例来说](https://blog.csdn.net/MGCTEL/article/details/86161220)：
>
>N = 16，ceiling{N/4}  = ceiling{16/4} = 4,  占用 1/16 <= K/16 <= 4/16（大于或等于 6.25%，小于或等于 25%）的处理器资源
>
>N = 8，ceiling{N/4}  = ceiling{8/4} = 2,  占用 1/8 <= K/8 <= 2/8 的处理器资源
>
>N = 4，ceiling{N/4}  = ceiling{4/4} = 1,  占用 1/4 <= K/4 <= 1/4 的处理器资源

In addition to the use of processors during concurrent phases, additional overhead is incurred to enable concurrency. Thus, while garbage collection pauses are typically much shorter with the concurrent collector, application throughput also tends to be slightly lower than with the other collectors.

On a machine with more than one processing core, processors are available for application threads during the concurrent part of the collection, so the concurrent garbage collector thread doesn't pause the application. 

This usually results in shorter pauses, but again fewer processor resources are available to the application and some slowdown should be expected, especially if the application uses all of the processing cores maximally.

As *N* increases, the reduction in processor resources due to concurrent garbage collection becomes smaller, and the benefit from concurrent collection increases. See [Concurrent Mode Failure](https://docs.oracle.com/javase/9/gctuning/concurrent-mark-sweep-cms-collector.htm#GUID-700D5A4A-75EE-4CDC-9A43-5DF8FEBE24DD), which discusses potential limits to such scaling.

Because at least one processor is used for garbage collection during the concurrent phases, the concurrent collectors don't normally provide any benefit on a uniprocessor (single-core) machine.

## CMS

> The CMS collector is deprecated as of JDK 9. 
>
> JDK 14 completely dropped the CMS support.

CMS（Concurrent Mark Sweep）：

- 关键词：<u>Old Gen</u>，<u>Mark-Sweep（标记-清除算法）</u>，<u>配合 PawNew 使用</u>
- This collector is for applications that prefer shorter garbage collection pauses and can afford to share processor resources with the garbage collection.
- Simply put, applications using this type of GC respond slower on average but do not stop responding to perform garbage collection.
- 极大的降低 STW 的时间，主要用于服务端
- 特点是<u>并发标记清除（CMS，Concurrent Mark Sweep）</u>，可以获取最短回收停顿时间（Shorter GC Pause）

**Mark-Sweep 只会清理标记了的对象，所以可以实现多线程并发清理，而不会 STW。这种方法可以缩短 GC Pause，但是会有产生 fragment（内存碎片）**

Use the option `-XX:+UseConcMarkSweepGC` to enable the CMS collector

步骤：

1. CMS-initial-mark（初始标记 / 初次标记）：
	- 标出 Old Gen 的 GC Roots 对象和被 Young Gen 引用的对象
	- 耗时短，<u>STW</u>
2. CMS-concurrent-mark（并发标记）：
	- 通过 [Tracing GC](https://www.baeldung.com/java-gc-cyclic-references#tracing-gcs)（可达性分析）标记所有 Old Gen 的对象 
	- 耗时长，但<u>不用 STW</u>
3. CMS-remark（重新标记）：
	- 因为 CMS-concurrent-mark 的时候，可能有些 Objects 的引用发生了改变，所以需要修正这些错误
	- 耗时中等，<u>STW</u> 
4. CMS-concurrent-sweep（并发清除 / 并发清理）：
	- 使用<u>标记清理（标记清除）算法</u>，定点清理内存
	- 因为不影响其他位置的内存，所以可以实现<u>并发清理，不会 STW</u>
5. CMS-concurrent-reset（并发重置）
	- 等待下次 CMS 的触发
	- 可以和其他正在运行的线程同时运行的线程，<u>不会 STW</u>

# G1 Basic

参考资料：官方文档 - [Garbage-First Garbage Collector](https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector.htm#JSGCT-GUID-0394E76A-1A8F-425E-A0D0-B48A3DC82B42)

## Basic

> The G1 (Garbage-First) garbage collector is the default collector, so typically you don't have to perform any additional actions. You can explicitly enable it by providing `-XX:+UseG1GC` on the command line.

G1 is a **generational**, **incremental**, **parallel**, **mostly concurrent**, **stop-the-world**, and **evacuating** garbage collector which **monitors pause-time goals in each of the stop-the-world pauses**.

G1 的 **Space-reclamation** （存储空间回收利用） efforts <u>concentrate on the young generation</u> where it is most efficient to do so, with occasional space-reclamation in the old generation:

- To keep stop-the-world pauses short for space-reclamation, G1 <u>performs space-reclamation incrementally in steps and in parallel</u>.

G1 有 STW 和并发多线程的运作方式：

- <u>Some operations are always performed in stop-the-world pauses to improve throughput</u>. 
- <u>Other operations</u> that would <u>take more time with the application stopped</u> such as whole-heap operations like *global marking* <u>are performed in parallel and concurrently</u> with the application.

<u>G1 achieves predictability by tracking information about previous application behavior and garbage collection pauses</u> to build a model of the associated costs. It uses this information to size the work done in the pauses. For example, <u>G1 reclaims space in the most efficient areas first</u> (that is the areas that are mostly filled with garbage, therefore the name).

**G1 reclaims space mostly by using evacuation** : 

- Live objects found within selected memory areas to collect are copied into new memory areas, compacting them in the process. 
- After an evacuation has been completed, the space previously occupied by live objects is reused for allocation by the application.

G1 is <u>not a real-time collector</u>. It tries to meet set pause-time targets with high probability over a longer time, but not always with absolute certainty for a given pause.

## Usage Scenarios

G1 garbage collector is <u>targeted for multiprocessor machines with a large amount of memory</u>.  It attempts to meet garbage collection pause-time goals with high probability while achieving high throughput with little need for configuration. 

G1 aims to provide the best balance between latency and throughput using current target applications and environments whose features include:

- <u>Heap sizes</u> up to ten of GBs or <u>larger</u>, with <u>more than 50%</u> of the Java heap <u>occupied with live data</u>.
- Rates of <u>object allocation</u> and <u>promotion</u> that can <u>vary significantly over time</u>（在不同的时间，会有所不同）. 
- A significant amount of fragmentation in the heap.（Heap 分为大量区域）
- Predictable pause-time target goals that aren’t longer than a few hundred milliseconds, <u>avoiding long garbage collection pauses</u>.

~~The G1 collector achieves high performance and tries to meet pause-time goals in several ways described in the following sections.~~

## Heap Layout & Region

**G1 partitions the heap into a set of equally sized heap regions, each a contiguous range of virtual memory** as shown in Figure 9-1. 

**G1 将 Heap 划分为大约 2000 个 regions，每个 regions 的大小是 1MB 到 32MB（必须是 2 的幂次方大小）**。

![Figure 9-1 G1 Garbage Collector Heap Layout](https://docs.oracle.com/javase/9/gctuning/img/jsgct_dt_004_grbg_frst_hp.png)

> 上图为：[Figure 9-1 G1 Garbage Collector Heap Layout](https://docs.oracle.com/javase/9/gctuning/img_text/jsgct_dt_004_grbg_frst_hp.htm)
>
> Light gray: Empty Regions  
>
> Red: Eden Regions  
>
> Red with "S": Survivor Regions  
>
> Light blue: Old Regions
>
> light blue with "H": Humongous Regions

**A region is the unit of memory allocation and memory reclamation** :

- At any given time, each of these regions can be empty, or assigned to a particular generation, young or old. 
- <u>As requests for memory comes in, the memory manager hands out free regions</u>. 
- <u>The memory manager assigns free regions to a generation and then returns them to the application as free space into which it can allocate itself</u>.

The Young Generation contains Eden regions and Survivor regions. These regions provide the same function as the respective contiguous spaces in other collectors, with the difference that in G1 these regions are typically <u>laid out in a noncontiguous pattern in memory</u>. Old regions make up the old generation. <u>Old generation regions may be **humongous** for objects that span multiple regions</u>.

**每个 regions 都可以作为 Young Gen（Eden + Survivor）和 Old Gen。**

An application always allocates into a young generation, that is, eden regions, with the exception of humongous, objects that are directly allocated as belonging to the old generation.

**Humongous objects 只能被存入 Old Gen，存储 Humongous Objects 的 region 被称为 Humongous Region**:

- **当 Object 大于等于 Region 大小的一半时，被视为 Humongous Object**
- **如果某个 Humongous Object 比一个 region 的 size 还要大，就会申请多个连续的 regions 来存放该 Humongous Object**

G1 garbage collection pauses can reclaim space in the young generation as a whole, and any additional set of old generation regions at any collection pause. 

During the pause G1 copies objects from this *collection set* to one or more different regions in the heap. The destination region for an object depends on the source region of that object: the entire young generation is copied into either survivor or old regions, and objects from old regions to other, different old regions using aging.

## GC Cycle

On a high level, the G1 collector alternates between two phases: 

- The **young-only phase** contains garbage collections that <u>fill up the currently available memory with objects in the old generation gradually</u>.
- The **space-reclamation phase** is where G1 <u>reclaims space in the old generation incrementally</u>, in addition to <u>handling the young generation</u>. 
- <u>Then the cycle restarts with a young-only phase</u>.

Figure 9-2 gives an overview about this cycle with an example of the sequence of garbage collection pauses that could occur:

![Figure 9-2 Garbage Collection Cycle Overview](https://docs.oracle.com/javase/9/gctuning/img/jsgct_dt_001_grbgcltncyl.png)

> 上图为：[Figure 9-2 Garbage Collection Cycle Overview](https://docs.oracle.com/javase/9/gctuning/img_text/jsgct_dt_001_grbgcltncyl.htm)

The following list describes **the phases, their pauses and the transition between the phases of the G1 garbage collection cycle** in detail:

1. **Young-only phase**: 
	- This phase <u>starts with a few young-only collections that promote objects into the old generation</u>. 
	- The <u>transition between the young-only phase and the space-reclamation phase starts when the old generation occupancy reaches the Initiating Heap Occupancy threshold</u>. At this time, G1 <u>schedules an Initial Mark young-only collection instead of a regular young-only collection</u>.
	- *Initial Mark* : 
		- This type of collection <u>starts the marking process</u> in addition to <u>performing a regular young-only collection</u>. 
		- **Concurrent marking** determines all currently <u>reachable (live) objects</u> in the <u>old generation regions</u> to be <u>kept for the following space-reclamation</u> phase. 
		- <u>While marking hasn’t completely finished, regular young collections may occur</u>. 
		- <u>Marking finishes with two special **stop-the-world** pauses: Remark and Cleanup.</u>
	- *Remark* (STW): 
		- This pause <u>finalizes the marking itself</u>, and <u>performs global reference processing and class unloading</u>. 
		- <u>Between Remark and Cleanup</u> G1 <u>calculates a summary of the liveness information concurrently</u>, which will <u>be finalized and used in the Cleanup pause to update internal data structures</u>.
	- *Cleanup* (STW): 
		- This pause also <u>reclaims completely empty regions</u>, and <u>determines whether a space-reclamation phase will actually follow</u>. 
		- <u>If a space-reclamation phase follows, the young-only phase completes with a single young-only collection</u>.
2. **Space-reclamation phase**: 
	- This phase <u>consists of multiple mixed collections</u> that in addition to young generation regions, also evacuate live objects of sets of old generation regions. 
	- The space-reclamation phase <u>ends when G1 determines that evacuating more old generation regions wouldn't yield enough free space worth the effort</u>.
3. After space-reclamation, the collection cycle restarts with another young-only phase. 
4. As backup, <u>if the application runs out of memory while gathering liveness information</u>, G1 performs an in-place <u>stop-the-world full heap compaction (Full GC)</u> like other collectors.

## Ergonomic Defaults for G1

This topic provides an overview of the most important defaults specific to G1 and their default values. They give a rough overview of expected behavior and resource usage using G1 without any additional options.

| Option and Default Value                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `-XX:MaxGCPauseMillis=200`                                   | The goal for the maximum pause time.                         |
| `-XX:GCPauseTimeInterval`=*<ergo>*                           | The goal for the maximum pause time interval. By default G1 doesn’t set any goal, allowing G1 to perform garbage collections back-to-back in extreme cases. |
| `-XX:ParallelGCThreads`=*<ergo>*                             | The maximum number of threads used for parallel work during garbage collection pauses. This is derived from the number of available threads of the computer that the VM runs on in the following way: if the number of CPU threads available to the process is fewer than or equal to 8, use that. Otherwise add five eighths of the threads greater than to the final number of threads. |
| `-XX:ConcGCThreads`=*<ergo>*                                 | The maximum number of threads used for concurrent work. By default, this value is `-XX:ParallelGCThreads` divided by 4. |
| `-XX:+G1UseAdaptiveIHOP` `-XX:InitiatingHeapOccupancyPercent=45` | Defaults for controlling the initiating heap occupancy indicate that adaptive determination of that value is turned on, and that for the first few collection cycles G1 will use an occupancy of 45% of the old generation as mark start threshold. |
| `-XX:G1HeapRegionSize=<ergo> `                               | The set of the heap region size based on initial and maximum heap size. So that heap contains roughly 2048 heap regions. The size of a heap region can vary from 1 to 32 MB, and must be a power of 2. |
| `-XX:G1NewSizePercent=5` `-XX:G1MaxNewSizePercent=60`        | The size of the young generation in total, which varies between these two values as percentages of the current Java heap in use. |
| `-XX:G1HeapWastePercent=5`                                   | The allowed unreclaimed space in the collection set candidates as a percentage. G1 stops the space-reclamation phase if the free space in the collection set candidates is lower than that. |
| `-XX:G1MixedGCCountTarget=8`                                 | The expected length of the space-reclamation phase in a number of collections. |
| `-XX:G1MixedGCLiveThresholdPercent=85`                       | Old generation regions with higher live object occupancy than this percentage aren't collected in this space-reclamation phase. |

Note: `<ergo>` means that the actual value is determined ergonomically depending on the environment.

## Comparison to Other Collectors

This is a summary of the main differences between G1 and the other collectors:

- Parallel GC can compact and reclaim space in the old generation only as a whole. G1 incrementally distributes this work across multiple much shorter collections. This substantially shortens pause time at the potential expense of throughput.
- Similar to the CMS, G1 concurrently performs part of the old generation space-reclamation concurrently. However, CMS can't defragment the old generation heap, eventually running into long Full GC's.
- G1 may exhibit higher overhead than other collectors, affecting throughput due to its concurrent nature.

Due to how it works, G1 has some unique mechanisms to improve garbage collection efficiency:

- G1 can reclaim some completely empty, large areas of the old generation during any collection. This could avoid many otherwise unnecessary garbage collections, freeing a significant amount of space without much effort.
- G1 can optionally try to deduplicate duplicate strings on the Java heap concurrently.

Reclaiming empty, large objects from the old generation is always enabled. You can disable this feature with the option `-XX:-G1EagerReclaimHumongousObjects`. String deduplication is disabled by default. You can enable it using the option `-XX:+G1EnableStringDeduplication`.

# G1 Internals

This section describes some important details of the Garbage-First (G1) garbage collector.

## IHOP

About Determining Initiating Heap Occupancy.

The **Initiating Heap Occupancy Percent (IHOP)** is the threshold at which <u>an Initial Mark collection is triggered</u> and it is <u>defined as a percentage of the old generation size</u>.

Adaptive IHOP :

- G1 by default <u>automatically determines an optimal IHOP by observing how long marking takes and how much memory is typically allocated in the old generation during marking cycles</u>. This feature is called *Adaptive IHOP*. 
- If *Adaptive IHOP* is active, then the option <u>`-XX:InitiatingHeapOccupancyPercent` determines the initial value as a percentage of the size of the current old generation as long as there aren't enough observations to make a good prediction of the Initiating Heap Occupancy threshold</u>. 

Turn off Adaptive IHOP of G1 using the option`-XX:-G1UseAdaptiveIHOP`. In this case, the value of `-XX:InitiatingHeapOccupancyPercent` always determines this threshold.

Internally, <u>Adaptive IHOP tries to set the Initiating Heap Occupancy</u> so that the first mixed garbage collection of the space-reclamation phase starts when the old generation occupancy is at a <u>*current maximum old generation size*</u> <u>minus</u> the <u>*value of `-XX:G1HeapReservePercent`*</u> <u>as the extra buffer</u>.

## Marking

G1 marking uses an algorithm called **Snapshot-At-The-Beginning (SATB)** :

- It <u>takes a virtual snapshot of the heap at the time of the Initial Mark pause</u>, when **all objects that were live at the start of marking are considered live for the remainder of marking**. 
- This means that <u>objects that become dead (unreachable) during marking are still considered live</u> for the purpose of space-reclamation (with some exceptions). 

This may cause some additional memory wrongly retained compared to other collectors. However, SATB potentially <u>provides better latency during the Remark pause</u>. 

> See the [Garbage-First Garbage Collector Tuning](https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector-tuning.htm#GUID-90E30ACA-8040-432E-B3A0-1E0440AB556A) topic for more information about problems with marking.

 **The too conservatively considered live objects during that marking willbe reclaimed during the next marking**. 

## Behavior in Very Tight Heap Situations

When the application <u>keeps alive so much memory</u> so that an <u>evacuation can't find enough space to copy to</u>, **an evacuation failure occurs**. 

*Evacuation failure* means that G1 tries to **complete the current garbage collection by <u>keeping any objects that have already been moved in their new location</u>, and <u>not copying any not yet moved objects, only adjusting references between the object</u>** . 

> *Evacuation failure* may incur some additional overhead（额外开销）, but generally should be as fast as other young collections. 

<u>After</u> this garbage collection with the <u>evacuation failure</u>, <u>G1 will resume the application as normal</u> without any other measures :

- <u>G1 assumes</u> that the evacuation failure occurred close to the end of the garbage collection; that is, <u>most objects were already moved</u> and <u>there is enough space left to continue running the application until marking completes and space-reclamation starts</u>.
- I<u>f this assumption doesn’t hold, then G1 will eventually schedule a Full GC</u>. This type of collection performs in-place compaction of the entire heap. This might be very slow.

> See [Garbage-First Garbage Collector Tuning](https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector-tuning.htm#GUID-90E30ACA-8040-432E-B3A0-1E0440AB556A) for more information about problems with allocation failure or Full GC's before signalling out of memory.

## Humongous Objects

Humongous objects are objects larger or equal the size of half a region. The current region size is determined ergonomically as described in the [Ergonomic Defaults for G1 GC](https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector.htm#GUID-082C967F-2DAC-4B59-8A81-0CEC6EEB9016) section, unless set using the `-XX:G1HeapRegionSize` option.

These humongous objects are sometimes treated in special ways:

- Every humongous object <u>gets allocated as **a sequence of contiguous regions** in the old generation</u> :
	- The start of the object itself is always located <u>at the start of the first region in that sequence</u>. 
	- <u>Any leftover space in the last region of the sequence will be lost for allocation until the entire object is reclaimed</u>.
- Reclaim humongous objects:
	- <u>Generally</u>, humongous objects can <u>be reclaimed only at the end of marking</u> during the Cleanup pause, or during Full GC if they became unreachable. 
	- There is, however, a <u>special provision for humongous objects for arrays of primitive types</u> for example, `bool`, all kinds of integers, and floating point values. 
	- G1 opportunistically <u>tries to reclaim humongous objects if they are not referenced by many objects at any kind of garbage collection pause</u>. This behavior is enabled by default but you can disable it with the option `-XX:G1EagerReclaimHumongousObjects`.
- <u>Allocations of humongous objects may cause garbage collection pauses to occur prematurely</u>. G1 checks the Initiating Heap Occupancy threshold at every humongous object allocation and may force an initial mark young collection immediately, if current occupancy exceeds that threshold.
- The <u>humongous objects never move</u>, not even during a Full GC. This <u>can cause premature slow Full GCs or unexpected out-of-memory conditions</u> with lots of free space left due to fragmentation of the region space.

## Young-Only Phase Generation Sizing

During the young-only phase, the set of regions to collect (collection set), consists only of young generation regions. 

G1 always sizes the young generation at the end of a young-only collection: 

- This way, G1 can meet the pause time goals that were set using `-XX:MaxGCPauseTimeMillis` and `-XX:PauseTimeIntervalMillis` based on long-term observations of actual pause time
- It takes into account how long it took young generations of similar size to evacuate. This includes information like how many objects had to be copied during collection, and how interconnected these objects had been.

If not otherwise constrained, then G1 adaptively sizes the young generation size between the values that `-XX:G1NewSizePercent` and `-XX:G1MaxNewSizePercent` determine to meet pause-time. 

> See [Garbage-First Garbage Collector Tuning](https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector-tuning.htm#GUID-90E30ACA-8040-432E-B3A0-1E0440AB556A) for more information about how to fix long pauses.

## Space-Reclamation Phase Generation Sizing

During the space-reclamation phase, G1 <u>tries to maximize the amount of space that's reclaimed in the old generation in a single garbage collection pause</u> :

- The size of the **young generation is set to minimum** allowed, typically as determined by -XX:G1NewSizePercent, 
- and **any old generation regions to reclaim space are added** <u>until G1 determines that adding further regions will exceed the pause time goal</u>. 

In a particular garbage collection pause, G1 **adds old generation regions in order of their reclamation efficiency, highest first, and the remaining available time** <u>to get the final collection set</u>.

The number of old generation regions to take per garbage collection is bounded at the lower end by the number of potential candidate old generation regions (*collection set candidate regions*) to collect, divided by the length of the space-reclamation phase as determined by `-XX:G1MixedGCCountTarget`. 

The **collection set candidate regions** are all <u>old generation regions that have an occupancy that's lower than `-XX:G1MixedGCLiveThresholdPercent` at the start of the phase</u>.

**The phase ends when the remaining amount of space that can be reclaimed in the collection set candidate regions is less than the percentage set by `-XX:G1HeapWastePercent`.**

> See [Garbage-First Garbage Collector Tuning](https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector-tuning.htm#GUID-90E30ACA-8040-432E-B3A0-1E0440AB556A) for more information about how many old generation regions G1 will use and how to avoid long mixed collection pauses.



# G1 Tuning - General

This section describes how to adapt Garbage-First garbage collector (G1 GC) behavior in case it does not meet your requirements.

## General Recommendations

The general recommendation is to use G1 with its default settings, eventually giving it a different pause-time goal and setting a maximum Java heap size by using `-Xmx` if desired.

G1 defaults have been balanced differently than either of the other collectors. G1's goals in the default configuration are neither maximum throughput nor lowest latency, but to <u>provide relatively small, uniform（均匀的） pauses at high throughput</u>. However, G1's mechanisms to incrementally reclaim space in the heap and the pause-time control incur some overhead in both the application threads and in the space-reclamation efficiency.

If you prefer **high throughput**, then **relax the pause-time goal by using `-XX:MaxGCPauseMillis`** or **provide a larger heap**. 

If **latency** is the main requirement, then **modify the pause-time target** . **Avoid limiting the young generation size to particular values by using options like `-Xmn`, `-XX:NewRatio`** and others <u>because the young generation size is the main means for G1 to allow it to meet the pause-time</u>. Setting the young generation size to a single value overrides and practically disables pause-time control.

## Moving to G1 from Other Collectors

Generally, when moving to G1 from other collectors, particularly the Concurrent Mark Sweep collector, start by removing all options that affect garbage collection, and only set the pause-time goal and overall heap size by using `-Xmx` and optionally `-Xms`.

Many options that are useful for other collectors to respond in some particular way, have either no effect at all, or even decrease throughput and the likelihood to meet the pause-time target. An example could be setting young generation sizes that completely prevent G1 from adjusting the young generation size to meet pause-time goals.

# G1 Tuning - Improving G1 Performance

For diagnosis purposes, G1 provides comprehensive logging. A good start is to **use the `-Xlog:gc*=debug` option** and then refine the output from that if necessary. 

The log provides a detailed overview during and outside the pauses about garbage collection activity. This includes the type of collection and a breakdown of time spent in particular phases of the pause.

## Observing Full GC

A full heap garbage collection (Full GC) is often very time consuming，可以添加 `-Xlog:gc*=debug` 输出 log：

- Full GCs caused by too high heap occupancy in the old generation can be detected by finding the words `Pause Full` (Allocation Failure) in the log
- Full GCs are typically preceded by garbage collections that encounter an evacuation failure indicated by `To-space exhausted` tags.

The reason that a <u>Full GC occurs</u> is <u>because the application allocates too many objects that can't be reclaimed quickly enough</u> :

- Often concurrent marking has not been able to complete in time to start a space-reclamation phase. 
- The probability to run into a Full GC can be compounded by the allocation of many humongous objects. Due to the way these objects are allocated in G1, they may take up much more memory than expected.

**The goal should be to ensure that concurrent marking completes on time.** This can be achieved by : 

- **decreasing the allocation rate in the old generation**
- **giving the concurrent marking more time to complete.**

G1 gives you several options to handle this situation better:

- If there are <u>a significant number of humongous objects on the Java heap</u>, then `gc+heap=info` logging shows the number next to humongous regions. After every garbage collection, the best option is to <u>try to reduce the number of objects</u>. You can achieve this by **increasing the region size using the `-XX:G1HeapRegionSize` option**. The currently selected heap region size is printed at the beginning of the log.
- **Increase the size of the Java heap**. This typically <u>increases the amount of time marking has to complete</u>.
- **Increase the number of concurrent marking threads by setting `-XX:ConcGCThreads` explicitly**.
- **Force G1 to start marking earlier**. G1 automatically determines the Initiating Heap Occupancy Percent (IHOP) threshold based on earlier application behavior. If the application behavior changes, these predictions might be wrong. There are two options: <u>Lower the target occupancy for when to start space-reclamation by increasing the buffer used in an adaptive IHOP calculation by **modifying `-XX:G1ReservePercent`**</u>; or, <u>disable the adaptive calculation of the IHOP by **setting it manually using `-XX:-G1UseAdaptiveIHOP` and `-XX:InitiatingHeapOccupancyPercent`**.</u>

Other causes than Allocation Failure for a Full GC typically indicate that either the application or some external tool causes a full heap collection. <u>If the cause is `System.gc()`, and there is no way to modify the application sources, the effect of Full GCs can be mitigated by using `-XX:+ExplicitGCInvokesConcurrent` or let the VM completely ignore them by setting `-XX:+DisableExplicitGC`</u>. External tools may still force Full GCs; they can be removed only by not requesting them.

## Humongous Object Fragmentation

<u>A Full GC could occur before all Java heap memory has been exhausted due to the necessity of finding a contiguous set of regions for them</u>. 

Potential options in this case are **increasing the heap region size by using the option `-XX:G1HeapRegionSize` to decrease the number of humongous objects**, or **increasing size of the heap**.

In extreme cases, there might not be enough contiguous space available for G1 to allocate the object even if available memory indicates otherwise. This would lead to a VM exit if that Full GC can not reclaim enough contiguous space. As a result, there are no other options than either decreasing the amount of humongous object allocations as mentioned previously, or increasing the heap.



## Tuning for Latency

This section discusses hints to improve G1 behavior in case of common latency problems that is, if the pause-time is too high.



### Unusual System or Real-Time Usage

For every garbage collection pause, the `gc+cpu=info` log output contains a line including information from the operating system with a breakdown about where during the pause-time has been spent. An example for such output is `User=0.19s Sys=0.00s Real=0.01s`.

User time is time spent in VM code, *system time* is the time spent in the operating system, and *real time* is the amount of absolute time passed during the pause. If the system time is relatively high, then most often the environment is the cause.

Common known issues for high system time are:

- The VM allocating or giving back memory from the operating system memory may cause unnecessary delays. Avoid the delays by setting minimum and maximum heap sizes to the same value using the options `-Xms` and `-Xmx`, and pre-touching all memory using `-XX:+AlwaysPreTouch` to move this work to the VM startup phase.
- Particularly in Linux, coalescing of small pages into huge pages by the *Transparent Huge Pages (THP)* feature tends to stall random processes, not just during a pause. Because the VM allocates and maintains a lot of memory, there is a higher than usual risk that the VM will be the process that stalls for a long time. Refer to the documentation of your operating system on how to disable the Transparent Huge Pages feature.
- Writing the log output may stall for some time because of some background task intermittently taking up all I/O bandwidth for the hard disk the log is written to. Consider using a separate disk for your logs or some other storage, for example memory-backed file system to avoid this.

Another situation to look out for is real time being a lot larger than the sum of the others this may indicate that the VM did not get enough CPU time on a possibly overloaded machine.



### Reference Object Processing Takes Too Long

Information about the time taken for processing of Reference Objects is shown in the *Ref Proc* and *Ref Enq* phases. During the Ref Proc phase, G1 updates the referents of Reference Objects according to the requirements of their particular type. In Ref Enq, G1 enqueues Reference Objects into their respective reference queue if their referents were found dead. If these phases take too long, then consider enabling parallelization of these phases by using the option `-XX:+ParallelRefProcEnabled`.



### Young-Only Collections Take Too Long

Young-only and, in general any young collection roughly takes time proportional to the size of the young generation, or more specifically, the number of live objects within the collection set that needs to be copied. If the *Evacuate Collection Set* phase takes too long, in particular, the *Object Copy* sub-phase, decrease `-XX:G1NewSizePercent`. This decreases the minimum size of the young generation, allowing for potentially shorter pauses.

Another problem with sizing of the young generation may occur if application performance, and in particular the amount of objects surviving a collection, suddenly changes. This may cause spikes in garbage collection pause time. It might be useful to decrease the maximum young generation size by using `-XX:G1MaxNewSizePercent`. This limits the maximum size of the young generation and so the number of objects that need to be processed during the pause.



### Mixed Collections Take Too Long

Mixed collections are used to reclaim space in the old generation. The collection set of mixed collections contains young and old generation regions. You can obtain information about how much time evacuation of either young or old generation regions contribute to the pause-time by enabling the `gc+ergo+cset=trace` log output. Look at the *predicted young region* time and *predicted old region* time for young and old generation regions respectively.

If the predicted young region time is too long, then see [Young-Only Collections Take Too Long](https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector-tuning.htm#GUID-5FD5BDB1-DB7D-4E31-9296-19C0A28F810C) for options. Otherwise, to reduce the contribution of the old generation regions to the pause-time, G1 provides three options:

- Spread the old generation region reclamation across more garbage collections by increasing `-XX:G1MixedGCCountTarget`.
- Avoid collecting regions that take a proportionally large amount of time to collect by not putting them into the candidate collection set by using -`XX:G1MixedGCLiveThresholdPercent`. In many cases, highly occupied regions take a lot of time to collect.
- Stop old generation space reclamation earlier so that G1 won't collect as many highly occupied regions. In this case, increase `-XX:G1HeapWastePercent`.

Note that the last two options decrease the amount of collection set candidate regions where space can be reclaimed for the current space-reclamation phase. This may mean that G1 may not be able to reclaim enough space in the old generation for sustained operation. However, later space-reclamation phases may be able to garbage collect them.



### High Update RS and Scan RS Times

To enable G1 to evacuate single old generation regions, G1 tracks locations of *cross-region references*, that is references that point from one region to another. The set of cross-region references pointing into a given region is called that region's *remembered set*. The remembered sets must be updated when moving the contents of a region. Maintenance of the regions' remembered sets is mostly concurrent. For performance purposes, G1 doesn't immediately update the remembered set of a region when the application installs a new cross-region reference between two objects. Remembered set update requests are delayed and batched for efficiency.

G1 requires complete remembered sets for garbage collection, so the *Update RS* phase of the garbage collection processes any outstanding remembered set update requests. The *Scan RS* phase searches for object references in remembered sets, moves region contents, and then updates these object references to the new locations. Depending on the application, these two phases may take a significant amount of time.

Adjusting the size of the heap regions by using the option `-XX:G1HeapRegionSize` affects the number of cross-region references and as well as the size of the remembered set. Handling the remembered sets for regions may be a significant part of garbage collection work, so this has a direct effect on the achievable maximum pause time. Larger regions tend to have fewer cross-region references, so the relative amount of work spent in processing them decreases, although at the same time, larger regions may mean more live objects to evacuate per region, increasing the time for other phases.

G1 tries to schedule concurrent processing of the remembered set updates so that the Update RS phase takes approximately `-XX:G1RSetUpdatingPauseTimePercent` percent of the allowed maximum pause time. By decreasing this value, G1 usually performs more remembered set update work concurrently.

Spurious high Update RS times in combination with the application allocating large objects may be caused by an optimization that tries to reduce concurrent remembered set update work by batching it. If the application that created such a batch happens just before a garbage collection, then the garbage collection must process all this work in the Update RS times part of the pause. Use `-XX:-ReduceInitialCardMarks` to disable this behavior and potentially avoid these situations.

Scan RS Time is also determined by the amount of compression that G1 performs to keep remembered set storage size low. The more compact the remembered set is stored in memory, the more time it takes to retrieve the stored values during garbage collection. G1 automatically performs this compression, called remembered set coarsening, while updating the remembered sets depending on the current size of that region's remembered set. Particularly at the highest compression level, retrieving the actual data can be very slow. The option `-XX:G1SummarizeRSetStatsPeriod` in combination with `gc+remset=trace` level logging shows if this coarsening occurs. If so, then the `X` in the line `Did <X> coarsenings` in the *Before GC Summary* section shows a high value. The `-XX:G1RSetRegionEntries` option could be increased significantly to decrease the amount of these coarsenings. Avoid using this detailed remembered set logging in production environments as collecting this data can take a significant amount of time.



## Tuning for Throughput

G1's default policy tries to maintain a balance between throughput and latency; however, there are situations where higher throughput is desirable. Apart from decreasing the overall pause-times as described in the previous sections, the frequency of the pauses could be decreased. The main idea is to increase the maximum pause time by using `-XX:MaxGCPauseMillis`. The generation sizing heuristics will automatically adapt the size of the young generation, which directly determines the frequency of pauses. If that does not result in expected behavior, particularly during the space-reclamation phase, increasing the minimum young generation size using `-XX:G1NewSizePercent` will force G1 to do that.

In some cases, `-XX:G1MaxNewSizePercent`, the maximum allowed young generation size, may limit throughput by limiting young generation size. This can be diagnosed by looking at region summary output of `gc+heap=info` logging. In this case the combined percentage of Eden regions and Survivor regions is close to `-XX:G1MaxNewSizePercent` percent of the total number of regions. Consider increasing`-XX:G1MaxNewSizePercent` in this case.

Another option to increase throughput is to try to decrease the amount of concurrent work in particular, concurrent remembered set updates often require a lot of CPU resources. Increasing `-XX:G1RSetUpdatingPauseTimePercent` moves work from concurrent operation into the garbage collection pause. In the worst case, concurrent remembered set updates can be disabled by setting `-XX:-G1UseAdaptiveConcRefinement` `-XX:G1ConcRefinementGreenZone=``2G` `-XX:G1ConcRefinementThreads=``0`. This mostly disables this mechanism and moves all remembered set update work into the next garbage collection pause.

Enabling the use of large pages by using `-XX:+UseLargePages` may also improve throughput. Refer to your operating system documentation on how to set up large pages.

You can minimize heap resizing work by disabling it; set the options `-Xms` and `-Xmx` to the same value. In addition, you can use `-XX:+AlwaysPreTouch` to move the operating system work to back virtual memory with physical memory to VM startup time. Both of these measures can be particularly desirable in order to make pause-times more consistent.

## Tuning for Heap Size

Like other collectors, G1 aims to size the heap so that the time spent in garbage collection is below the ratio determined by the `-XX:GCTimeRatio` option. Adjust this option to make G1 meet your requirements.



## Tunable Defaults

This section describes the default values and some additional information about command-line options that are introduced in this topic.

Table 10-1 Tunable Defaults G1 GC

| Option and Default Value                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `-XX:+G1UseAdaptiveConcRefinement` `-XX:G1ConcRefinementGreenZone=`*<ergo>*`-XX:G1ConcRefinementYellowZone=`*<ergo>*`-XX:G1ConcRefinementRedZone=`*<ergo>*`-XX:G1ConcRefinementThreads=`*<ergo>* | The concurrent remembered set update (refinement) uses these options to control the work distribution of concurrent refinement threads. G1 chooses the ergonomic values for these options so that `-XX:G1RSetUpdatingPauseTimePercent` time is spent in the garbage collection pause for processing any remaining work, adaptively adjusting them as needed. Change with caution because this may cause extremely long pauses. |
| `-XX:+ReduceInitialCardMarks`                                | This batches together concurrent remembered set update (refinement) work for initial object allocations. |
| `-XX:-ParallelRefProcEnabled`                                | This determines whether processing of java.lang.Ref.* instances should be done in parallel by multiple threads. |
| `-XX:G1RSetUpdatingPauseTimePercent=10`                      | This determines the percentage of total garbage collection time G1 should spend in the Update RS phase updating any remaining remembered sets. G1 controls the amount of concurrent remembered set updates using this setting. |
| `-XX:G1SummarizeRSetStatsPeriod=0`                           | This is the period in a number of GCs that G1 generates remembered set summary reports. Set this to zero to disable. Generating remembered set summary reports is a costly operation, so it should be used only if necessary, and with a reasonably high value. Use `gc+remset=trace` to print anything. |
| `-XX:GCTimeRatio=12`                                         | This is the divisor for the target ratio of time that should be spent in garbage collection as opposed to the application. The actual formula for determining the target fraction of time that can be spent in garbage collection before increasing the heap is `1 / (1 + GCTimeRatio)`. This default value results in a target with about 8% of the time to be spent in garbage collection. |



Note: `<ergo>` means that the actual value is determined ergonomically depending on the environment.



# JVM Performance Tuning：Java 虚拟机调优

## JVM 调优目标：减少 STW

**JVM Performance Tuning 的目的是减少 GC**，特别是减少 Full GC（Often a Full GC (Major GC) is much slower because it involves all live objects）

**为什么要减少 GC？因为 GC 会 Stop-The-World** ：

- *Stop-The-World* also known as *STW*, *Stop the World Event*, *Stop-The-World Pause*, *STW Pause*
- **All application threads are stopped until the operation (GC) completes**
- 这也就造成了 GC Pause（Garbage Collection Pause）

为什么要使用 STW（Stop-The-World）机制？

如果在 GC 没结束前，正在运行的 Threads 就已经结束了，那么 Stack 占用的空间就会被释放掉，与此同时，Stack 存储的 Local Variables 也就被销毁掉了。

如果没有 STW 机制，那么在 Threads 还没结束前，就被 GC 标记为 Garbage 的 Object 实际上就为 `null` 了。这样的话，GC 需要回收的 Object 为 `null` ，就要重新再找一次，会出现问题。（更多 STW 相关，可以参考 [另一篇 JVM 笔记](https://github.com/LearnDifferent/my-notes/blob/full/JVM%E7%AC%94%E8%AE%B0-%E6%9E%81%E5%AE%A2%E6%97%B6%E9%97%B4-%E6%B7%B1%E5%85%A5%E6%8B%86%E8%A7%A3Java%E8%99%9A%E6%8B%9F%E6%9C%BA.md)）

所以，需要 STW 来防止 Object 有时候是 Garbage，有时候又不是 Garbage 的情况。

> Garbage（垃圾）：When an object can no longer be reached from any pointer in the running program, it is considered "garbage" and ready for collection

**The goal of tuning your heap size is to minimize the time that your JVM spends doing garbage collection** while maximizing the number of clients that WebLogic Server can handle at a given time. A best practice is to tune the time spent doing garbage collection to within 5% of execution time.

参考资料：

- [Tuning Java Virtual Machines (JVMs)](https://docs.oracle.com/cd/E15523_01/web.1111/e13814/jvm_tuning.htm#PERFM150)
- [Java Garbage Collection Basics](https://www.oracle.com/technetwork/tutorials/tutorials-1873457.html)

## JVM 指令与工具

> 这里使用的是 JDK 11，相关内容可以查看 [Oracle 官方工具文档](https://docs.oracle.com/en/java/javase/11/tools/index.html) ，下文主要涉及官方文档中的 [Monitoring Tools and Commands](https://docs.oracle.com/en/java/javase/11/tools/monitoring-tools-and-commands.html) 和 [Troubleshooting Tools and Commands](https://docs.oracle.com/en/java/javase/11/tools/troubleshooting-tools-and-commands.html)

查看当前 Java 进程的 pid：

```bash
jps
```

监视和管理控制台（图形化界面）：

```bash
jconsole
```

查看某个 pid 的线程状况：

```bash
jstack $pid
```

---

将某个 pid 的 Heap 信息，另存为一个自定义 filename 的文件：

```bash
jmap -dump:file=$filename $pid
```

这个 dump 出来的文件，需要其他工具来查看。具体的工具，可以查看：[A Guide to Java Profilers](https://www.baeldung.com/java-profilers)

如果想 output the contents on the console（在控制台直接输出结果），可以使用：

```bash
jmap -histo $pid # 输出该进程的 Heap 信息
jmap -histo:live $pid # count only live objects
```

更多 `jmap` 相关，可以查看 [官方文档](https://docs.oracle.com/en/java/javase/11/tools/jmap.html#GUID-D2340719-82BA-4077-B0F3-2803269B7F41)

---

打印某个 pid 进程的信息（百分比模式），可以设置打印的 interval（间隔时间，单位是毫秒）：

```bash
jstat -gcutil $pid $interval
```

比如，间隔 1000 毫秒，打印 pid 为 995 的 Java 进程的信息：

```bash
jstat -gcutil 995 1000
```

上面的命令 ，可能会每隔 1000 豪秒就打印一次的如下信息（部分截取）：

```
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT    CGC    CGCT     GCT   
  0.00 100.00  57.58  11.06  96.30  90.84      3    0.029     0    0.000     2    0.003    0.032
  0.00 100.00  57.58  11.06  96.30  90.84      3    0.029     0    0.000     2    0.003    0.032
  0.00 100.00  57.58  11.06  96.30  90.84      3    0.029     0    0.000     2    0.003    0.032
  0.00 100.00  57.58  11.06  96.30  90.84      3    0.029     0    0.000     2    0.003    0.032
```

其中的 Column 及其 Description：

- `S0`: Survivor space 0 utilization as a percentage of the space's current capacity.
- `S1`: Survivor space 1 utilization as a percentage of the space's current capacity.
- `E`: Eden space utilization as a percentage of the space's current capacity.
- `O`: Old space utilization as a percentage of the space's current capacity.
- `M`: Metaspace utilization as a percentage of the space's current capacity.
- `CCS`: Compressed class space utilization as a percentage.
- `YGC`: Number of young generation GC events. (Young GC = Minor GC)
- `YGCT`: Young generation garbage collection time.
- `FGC`: Number of full GC events.
- `FGCT`: Full garbage collection time.
- `GCT`: Total garbage collection time.

上面使用的是 `-gcutil` 参数，主要显示的是百分比。还可以：

- `-gc` 具体的值
- `-gcnew` New generation statistics
- `-gcold` Old generation size statistics
- 其他参数可以参考 [官方文档](https://docs.oracle.com/en/java/javase/11/tools/jstat.html#GUID-5F72A7F9-5D5A-4486-8201-E1D1BA8ACCB5)

参考资料：

- [Oracle - JDK 11 - 官方工具文档](https://docs.oracle.com/en/java/javase/11/tools/index.html)
- [【java】jvm指令与工具jstat/jstack/jmap/jconsole/jps/visualVM](https://www.bilibili.com/video/BV1QJ411P78Q)
- [A Guide to Java Profilers](https://www.baeldung.com/java-profilers)



## Selecting a Collector

> 查看当前 Collector：`java -XX:+PrintCommandLineFlags -version`

Unless your application has rather strict pause-time requirements, first run your application and allow the VM to select a collector.

If necessary, adjust the heap size to improve performance. If the performance still doesn't meet your goals, then use the following guidelines as a starting point for selecting a collector:

- If the application has a small data set (up to approximately 100 MB), then select the serial collector with the option -XX:+UseSerialGC.
- If the application will be run on a single processor and there are no pause-time requirements, then select the serial collector with the option -XX:+UseSerialGC.
- If (a) peak application performance is the first priority and (b) there are no pause-time requirements or pauses of one second or longer are acceptable, then let the VM select the collector or select the parallel collector with -XX:+UseParallelGC.
- If response time is more important than overall throughput and garbage collection pauses must be kept shorter than approximately one second, then select a concurrent collector with -XX:+UseG1GC or -XX:+UseConcMarkSweepGC.

These guidelines provide only a starting point for selecting a collector because performance is dependent on the size of the heap, the amount of live data maintained by the application, and the number and speed of available processors. Pause-time is particularly sensitive to these factors, so the threshold of one second mentioned previously is only approximate. The parallel collector will experience a pause time longer than one second on many heap size and hardware combinations. Conversely, the concurrent collectors may not be able to keep pauses shorter than one seconds in some cases.

If the recommended collector doesn't achieve the desired performance, then first attempt to adjust the heap and generation sizes to meet the desired goals. If performance is still inadequate, then try a different collector: Use the concurrent collector to reduce pause-time, and use the parallel collector to increase overall throughput on multiprocessor hardware.