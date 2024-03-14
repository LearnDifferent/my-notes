# ThreadLocal & Strong Reference & Soft Reference & Weak Reference & Phantom Reference & TransmittableThreadLocal & InheritableThreadLocal

参考资料：[深入JDK源码全面解析ThreadLocal原理 - 马士兵](https://www.bilibili.com/video/BV1ZY4y1P799) （内容相同的视频：[巧用弱引用解决ThreadLocal内存泄漏问题 - 马士兵](https://www.bilibili.com/video/BV19H4y1U7Jc/)）

## 强引用 Strong Reference、软引用 Soft Reference 和弱引用 Weak Reference

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

---

**ThreadLocalMap 存储的 ThreadLocal 是 WeakReference 弱引用**

上文中的 ThreadLocalMap 的 `map.set(this, value);` 方法的源码里面有：

```java
private void set(ThreadLocal<?> key, Object value) {
    // 省略...
    tab[i] = new Entry(key, value);
    // 省略...
}
```

而 `new Entry(key, value)` 的 `Entry` 的源码部分截取如下：

```java
static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
}
```

也就是说，构造器 `Entry(ThreadLocal<?> k, Object v)` 因为继承了 `WeakReference` 弱引用，所以构造器实际上是将 `ThreadLocal` 作为参数构造了 `WeakReference` 。

所以 `ThreadLocalMap` 的 key 存储的 `ThreadLocal` 实际上是 WeakReference 弱引用。

---

**ThreadLocalMap 里面的 ThreadLocal 使用 WeakReference 弱引用是为了防止内存泄漏，但是最好使用 `remove()` 方法移除当前 ThreadLocal 里面 ThreadLocalMap 的所有 key 和 value**

以前的 JDK 是将 `ThreadLocalMap` 里面的当前 `ThreadLocal` 实例设置为 Strong Reference 强引用的，但是这样会产生问题。

因为正常情况下，为了回收 `ThreadLocal`，一般会将 `ThreadLocal<String> tl = new ThreadLocal<>();` 的这个 `tl` 设置设置为 `null` 。

但是这样只是让 `tl` 和 `new ThreadLocal<>()` 断开了引用，`ThreadLocalMap` 里面存储的 key（也就是 `ThreadLocal`）仍然指向了 `new ThreadLocal<>()` 实例。

这也就造成了内存泄漏。如果有多个这样的 `ThreadLocal` 一直被很多个线程的 `ThreadLocalMap` 里面的 key 指向着，那么就可能会出现 OOM 内存溢出的问题。

`ThreadLocalMap` 里面的 key 使用 WeakReference 弱引用的话，这个 key 指向的 `ThreadLocal` 实例就可以实现自动清除。

但是这只是一个安全机制，只能保证 `ThreadLocalMap` 的 key 能被移除，而 value 其实还不会被移除。所以要记得使用 `ThreadLocal` 的 `remove()` 方法，移除当前 `ThreadLocal` 里面的 `ThreadLocalMap` 的所有 key 和 value。

## 虚引用 Phantom Reference

> Phantom Reference 虚引用，也称幽灵引用或者幻影引用

Phantom Reference 使用方法如下：

```java
ReferenceQueue<Object> referenceQueue = new ReferenceQueue<>();
PhantomReference<Object> phantomReference = new PhantomReference<>(new Object(), referenceQueue);
```

`new PhantomReference<>(new Object(), referenceQueue)` 中，`referenceQueue` 这个 Queue 是用来装 `new` 出来的 `PhantomReference` 对象的。

也就是说，有一个 Queue 是 `referenceQueue`，`new PhantomReference` 被塞进这个 Queue 里面，而这个 `new PhantomReference` 将 `new Object()` 变为了 Phantom Reference。

---

Phantom Reference 在业务开发上基本用不到，它主要是给 GC 使用的。

我们的 JVM 里面有堆内存，堆内存里面，有些对象会指向堆外内存，这些堆外内存是不归 JVM 直接管理的。

这些堆内存里面，指向堆外内存的对象，在 GC 的时候，会被放到特殊的 Queue 里面，再通知 JVM 去处理。

假设一个实际的应用场景，比如我们写一个网络程序，需要使用网卡的功能。

OS 会将网卡的数据拷贝到 OS 的内存中，而我们的 JVM 如果不能直接操作 OS 内存中的网卡数据，就要再拷贝一次，将网卡数据也拷贝到 JVM 中。

直接内存 Direct Memory 和零拷贝 Zero-Copy：

 - 为了直接操作 OS 的内存，我们可以在 JVM 中的堆内存里面
 - 指定一个对象，这个对象可以直接指向 OS 的内存区域
 - 直接内存 Direct Memory 存储的就是这种类型的对象
 - 这种机制被叫做零拷贝 Zero-Copy

当我们用完了网卡的数据，想清理的时候，JVM 里面的堆内存的对象可以直接清理，但是堆外内存，也就是 OS 内存里面的网卡数据就不能 JVM 直接清理了。

所以使用了 Phantom Reference 后，JVM 会将这个对象放在 Queue 中，需要清理的时候，会将 Queue 拿出来，依次清理里面的堆内存对象，并发出指令去清理其引用的堆外内存的对象。

在 `java.nio` 包下面有 Direct Memory 和 Zero-Copy 以及 Phantom Reference 的应用。

比如我们可以这样 `ByteBuffer buffer = ByteBuffer.allocateDirect(1024);` 分配一个直接内存。`ByteBuffer.allocateDirect()` 的源码如下：

```java
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}
```

这里面 `new` 出来的 `DirectByteBuffer` 有一个 `public Cleaner cleaner() { return cleaner; }` 方法。

这个 `Cleaner` 类型实际上就是 `extends` 了 `PhantomReference<Object>` 的一个虚引用 Phantom Reference，清理的时候就可以将堆内存的对象，和堆外内存的对象一起清理了。

## TransmittableThreadLocal

> 下文来自 AI

问：单纯使用 ThreadLocal 会有什么问题？

答：

单纯使用 `ThreadLocal` 在多线程程序中尤其是在涉及线程池的情况下会遇到一些问题。`ThreadLocal` 提供了一种线程局部变量的存储机制，每个线程都可以通过它来存储自己的变量副本，这些变量对其他线程是隔离的。然而，当线程被线程池复用时，这些局部变量的处理就变得复杂了。以下是使用 `ThreadLocal` 可能遇到的一些问题：

1. 内存泄露

`ThreadLocal` 可能导致内存泄露。`ThreadLocal` 的实现是通过在 `Thread` 对象中保存一个 `ThreadLocal.ThreadLocalMap` 来实现的。如果一个 `ThreadLocal` 对象不再被使用，而线程继续存活，那么由于 `ThreadLocalMap` 持有 `ThreadLocal` 的强引用，这将导致 `ThreadLocal` 相关的对象无法被垃圾回收，从而可能导致内存泄露。这在使用静态的 `ThreadLocal` 变量时尤其需要注意。

2. 值污染

在使用线程池等会复用线程的情况下，如果线程局部变量没有在使用后被及时清理（例如调用 `ThreadLocal.remove()`），那么这个线程在之后被重新使用时，线程局部变量的值可能会“污染”后续的任务。也就是说，后续的任务可能会错误地访问到之前任务在 `ThreadLocal` 中遗留的数据。

3. 复杂的并发环境下的数据不一致

在复杂的并发环境下，特别是在微服务架构中，请求可能需要通过多个服务和多个线程进行处理。如果仅使用 `ThreadLocal` 来传递上下文信息（如用户认证信息），那么在异步调用或线程切换时，上下文信息可能会丢失，导致无法正确处理请求或数据不一致。

4. 性能问题

虽然 `ThreadLocal` 通常不会直接引起性能问题，但是不恰当的使用（如频繁地创建和销毁 `ThreadLocal` 对象，或者在每次使用后未调用 `remove()` 方法）可能会间接影响应用程序的性能，尤其是在高并发的环境下。

解决方案：

- **及时清理**：确保在每次使用 `ThreadLocal` 变量后及时调用 `remove()` 方法进行清理，避免值污染。
- **使用 `TransmittableThreadLocal`**：对于需要在线程之间传递上下文信息的场景，可以使用 `TransmittableThreadLocal` 替代 `ThreadLocal`。`TransmittableThreadLocal` 保证了在使用线程池等并发组件时，上下文信息能够被正确传递和管理。
- **慎重使用静态 `ThreadLocal`**：静态 `ThreadLocal` 变量更容易导致内存泄露，因此应该慎重使用，并确保及时清理。

总之，虽然 `ThreadLocal` 是解决多线程程序中数据隔离问题的有力工具，但在使用时需要注意以上问题，并采取相应措施避免这些问题。

---

> 下文来自 AI

问：[这里](https://github.com/yangzongzhuan/RuoYi-Cloud/blob/master/ruoyi-common/ruoyi-common-core/src/main/java/com/ruoyi/common/core/context/SecurityContextHolder.java) 为什么要使用 `TransmittableThreadLocal` ？

```java
package com.ruoyi.common.core.context;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import com.alibaba.ttl.TransmittableThreadLocal;
import com.ruoyi.common.core.constant.SecurityConstants;
import com.ruoyi.common.core.text.Convert;
import com.ruoyi.common.core.utils.StringUtils;

/**
 * 获取当前线程变量中的 用户id、用户名称、Token等信息 
 * 注意： 必须在网关通过请求头的方法传入，同时在HeaderInterceptor拦截器设置值。 否则这里无法获取
 *
 * @author ruoyi
 */
public class SecurityContextHolder
{
    private static final TransmittableThreadLocal<Map<String, Object>> THREAD_LOCAL = new TransmittableThreadLocal<>();

    public static void set(String key, Object value)
    {
        Map<String, Object> map = getLocalMap();
        map.put(key, value == null ? StringUtils.EMPTY : value);
    }

    public static String get(String key)
    {
        Map<String, Object> map = getLocalMap();
        return Convert.toStr(map.getOrDefault(key, StringUtils.EMPTY));
    }

    public static <T> T get(String key, Class<T> clazz)
    {
        Map<String, Object> map = getLocalMap();
        return StringUtils.cast(map.getOrDefault(key, null));
    }

    public static Map<String, Object> getLocalMap()
    {
        Map<String, Object> map = THREAD_LOCAL.get();
        if (map == null)
        {
            map = new ConcurrentHashMap<String, Object>();
            THREAD_LOCAL.set(map);
        }
        return map;
    }

    public static void setLocalMap(Map<String, Object> threadLocalMap)
    {
        THREAD_LOCAL.set(threadLocalMap);
    }

    public static Long getUserId()
    {
        return Convert.toLong(get(SecurityConstants.DETAILS_USER_ID), 0L);
    }

    public static void setUserId(String account)
    {
        set(SecurityConstants.DETAILS_USER_ID, account);
    }

    public static String getUserName()
    {
        return get(SecurityConstants.DETAILS_USERNAME);
    }

    public static void setUserName(String username)
    {
        set(SecurityConstants.DETAILS_USERNAME, username);
    }

    public static String getUserKey()
    {
        return get(SecurityConstants.USER_KEY);
    }

    public static void setUserKey(String userKey)
    {
        set(SecurityConstants.USER_KEY, userKey);
    }

    public static String getPermission()
    {
        return get(SecurityConstants.ROLE_PERMISSION);
    }

    public static void setPermission(String permissions)
    {
        set(SecurityConstants.ROLE_PERMISSION, permissions);
    }

    public static void remove()
    {
        THREAD_LOCAL.remove();
    }
}
```

答：

`TransmittableThreadLocal` 是阿里巴巴开源的一个 Java 库 [Transmittable ThreadLocal (TTL)](https://github.com/alibaba/transmittable-thread-local) 的一部分，它扩展了 Java 标准库中的 `ThreadLocal`，使其不仅能够保持线程局部变量的值，而且还能在使用线程池等会复用线程的场景中，正确地传递这些线程局部变量。

在并发编程中，`ThreadLocal` 用于在当前线程中保存变量，这样不同的线程可以保持自己独立的变量副本，互不干扰。这在处理用户会话或事务的上下文信息时非常有用。然而，当使用线程池等JUC并发工具时，线程是被复用的，这意味着一个线程可能会依次执行多个任务。由于 `ThreadLocal` 无法自动处理线程间的值传递，当一个新任务提交给线程池时，这个任务可能看到之前任务留下的上下文信息，这可能导致数据错乱或安全问题。

`TransmittableThreadLocal` 解决了这个问题。它提供了一种机制，能够在任务提交给线程池执行时，自动捕获从父线程继承的 `ThreadLocal` 值，并在任务执行前后，分别恢复和清理这些值。这样，即使在并发环境下，也能保证线程局部变量的正确传递和隔离。

在你提供的 `SecurityContextHolder` 类中使用 `TransmittableThreadLocal` 而不是普通的 `ThreadLocal`，是为了确保在使用线程池等并发执行环境时，当前线程变量中的用户ID、用户名称、Token等敏感信息能够正确地管理和传递。这对于维护一个安全的、多用户的系统尤其重要，特别是在微服务架构中，服务之间的调用往往涉及到异步处理和任务调度，`TransmittableThreadLocal` 确保了上下文信息（如安全凭证）的正确传递，避免了安全漏洞和数据一致性问题。

简而言之，使用 `TransmittableThreadLocal` 是为了在复杂的并发和异步执行环境中，安全、正确地处理线程局部变量，特别是那些关键的上下文信息，如用户身份标识和权限信息。
