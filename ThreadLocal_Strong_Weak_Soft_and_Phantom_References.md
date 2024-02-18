# ThreadLocal 

参考资料：[深入JDK源码全面解析ThreadLocal原理 - 马士兵](https://www.bilibili.com/video/BV1ZY4y1P799) （内容相同的视频：[巧用弱引用解决ThreadLocal内存泄漏问题 - 马士兵](https://www.bilibili.com/video/BV19H4y1U7Jc/)）

## 强引用、软引用、弱引用和虚引用

**强引用 Strong Reference**

像 `Object obj = new Object();` 这种引用指向一个 `Object` 对象的，就属于强引用 Strong Reference。

只有当没有引用指向这个 `Object` 对象了，这个 `Object` 对象才会被清空，比如 `obj = null;` 。

---

**软引用 Soft Reference**

软硬用是一个 Java 类型 `SoftReference` ，使用方法如下：

```java
// 10mb 的 byte 数组
SoftReference<byte[]> sr = new SoftReference<>(new byte[1024 * 1024 * 10]);
```

此时，上面的 `sr` 还是强引用 Strong Reference，指向的是 `new` 出来的这个 `SoftReference` 对象。

而这个 `SoftReference` 对象才是 <u>软引用 Soft Reference</u>，指向的是 10mb 的 byte 数组。可以使用 `get()` 方法获取软引用 Soft Reference 的对象：

```java
byte[] bytes = sr.get();
```

如果启动的时候，加上 `-Xmx20M` 的 VM 参数来限定 JVM 的大小为 20MB。

当我们使用 `sr.get()` 获取软引用 Soft Reference 里面的 10MB 的数组时，如果当前 JVM 内存占用小于 20MB，就能成功取到软引用 Soft Reference 指向的 10MB 数组。

如果 JVM 内存占用大于 20MB，此时不会立刻出现 OOM 错误，而是会释放软引用 Soft Reference 里面的对象，此时 `sr.get()` 得到的是 `null` ，也就是 10MB 的数组被清除了。

如果有多个软引用 Soft Reference，比如此时多个出一个 12MB 的软引用 `SoftReference<byte[]> newSr = new SoftReference<>(new byte[1024 * 1024 * 12]);` ，那么前一个 `sr` 会被清理，也就是 `sr.get()` 为 `null` ，而后一个 `newSr.get()` 能够正常获取到 12MB 的数组。即，清理软引用时，会从最旧的软引用 Soft Reference 开始清理。

当然，任何时候，如果软引用 Soft Reference 释放后，还是超过了 JVM 的最大内存空间，就会抛出 OOM 错误。

软引用的使用场景：

- 软引用 Soft Reference 适用于 *缓存* 场景
- 假设我们需要频繁使用一个图片文件，如果每次都从硬盘中读取该图片文件，用完后就释放，这样 IO 消耗很大
- 我们可以使用软引用来缓存这个图片，需要的时候就从软引用 Soft Reference 中 `get()` 这个图片
- 当内存不够的时候，JVM 会帮我们自动清理这个软引用里面的图片，等我们需要这个图片的时候，再通过 IO 读取图片文件的方式，存到软引用中（判断如果 `get()` 的结果是 `null` ，就从硬盘中读取）

---

**弱引用 Weak Reference**

跟软引用类似，弱引用的 Java 类型是 `WeakReference`，使用方法如下：

```java
WeakReference<Object> wr = new WeakReference<>(new Object());
```

上面的 `wr` 指向的 `new WeakReference()` 还是强引用 Strong Reference， 而 `new WeakReference()` 指向的 `new Object()` 是弱引用 Weak Reference。

当发生 GC 时，无论怎么样，弱引用 Weak Reference 都会被立刻清除，除非弱引用指向的对象，有其他强引用也指向它。也就是如下情况：

```java
// 情况 1：new Object 只有弱引用时，GC 会直接清理掉这个 new 出来的 Object：
WeakReference<Object> wr1 = new WeakReference<>(new Object());
// 能正常打印
System.out.println(wr1.get());
System.gc();
// 软引用被清理，打印 null
System.out.println(wr1.get());

// 情况 2：new Object 除了弱引用之外，还有有强引用时候，GC 不会直接清理这个 Object
Object obj = new Object();
WeakReference<Object> wr2 = new WeakReference<>(obj);
// 能打印
System.out.println(wr2.get());
System.gc();
// 还存在强引用 obj，所以也能打印
System.out.println(wr2.get());
```

打印的结果：

```
java.lang.Object@cb644e
null
java.lang.Object@13805618
java.lang.Object@13805618
```

## ThreadLocal 基础

先看看 Spring 的 `@Transactional` 注解，它可以保证事务。

假设在一个 `@Transactional` 注解的方法下，有两个方法，这两个方法都是注入的类的方法。

如果要保证事务，那么这两个方法的数据库连接（Connection），肯定都是要同一个才行。而数据库连接（Connection）是从数据库的连接池那里取的。

所以，当这个事务之内，任意一个方法，首次拿到数据库连接 Connection 的时候，会将 Connection 放到当前线程的 ThreadLocal 里面，接下来这个事务内的其他方法就都从 ThreadLocal 里面拿 Connection，这就保证了事务内用的是同一个数据库连接。

---

**ThreadLocal 的线程隔离**

假设有个 `ThreadLocal<String> tl = new ThreadLocal<>();` 表示每个线程都有一个自己的字符串。

现在新建一个线程，专门储存当前的线程的名称：

```java
new Thread(() -> {
    String currentThreadName = Thread.currentThread().getName();
    tl.set(currentThreadName);
}).start();
```

上面这个 `tl.set()` 的源码如下：

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    // 以当前线程为参数，获取 ThreadLocalMap，
    // 这个 ThreadLocalMap 实际上是当前线程的成员变量 threadLocals
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 这个 ThreadLocalMap 的 key 是当前的 ThreadLocal 实例，value 就是需要存储的值
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}

ThreadLocalMap getMap(Thread t) {
    // 每个线程都有自己的 ThreadLocalMap 类型的 threadLocals
    return t.threadLocals;
}
```

也就是说，每个线程都有自己的 `ThreadLocalMap threadLocals` 属性，这样就可以保证拿的时候也是拿自己线程里面的内容。

在 ThreadLocalMap 中，key 是当前 ThreadLocal 的实例，也就是 `tl` 指向的 `new ThreadLocal<>()` 实例对象，而 value 才是需要保存在线程中的值。

对任意一个 Thread 来说，它有一个 ThreadLocalMap，该 Thread 的 ThreadLocalMap 中，存储的内容如下：

| key | value |
| ---- | ---- |
| 某个ThreadLocal | 对应存储的值 |
| 另一个ThreadLocal | 另一个对应存储的值 |
| ...... | ...... |

