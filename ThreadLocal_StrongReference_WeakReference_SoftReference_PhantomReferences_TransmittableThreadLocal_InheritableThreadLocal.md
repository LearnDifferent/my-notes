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

## InheritableThreadLocal

> 下文整理自 AI

`ThreadLocal` 与 `InheritableThreadLocal` 的对比：

```java
public class InheritableThreadLocalExample {
    
    private static final ThreadLocal<String> THREAD_LOCAL = new ThreadLocal<>();
    private static final InheritableThreadLocal<String> INHERITABLE_THREAD_LOCAL = new InheritableThreadLocal<>();

    public static void main(String[] args) {
        THREAD_LOCAL.set("[THREAD_LOCAL] Value from parent thread");
        INHERITABLE_THREAD_LOCAL.set("[INHERITABLE_THREAD_LOCAL] Value from parent thread");

        Thread childThread1 = new Thread(() -> {
            System.out.println("Child thread1 value: " + THREAD_LOCAL.get());
            THREAD_LOCAL.remove();
        });

        Thread childThread2 = new Thread(() -> {
            System.out.println("Child thread2 value: " + INHERITABLE_THREAD_LOCAL.get());
            INHERITABLE_THREAD_LOCAL.remove();
        });

        childThread1.start();
        childThread2.start();
    }
}
```

打印结果：

```
Child thread1 value: null
Child thread2 value: [INHERITABLE_THREAD_LOCAL] Value from parent thread
```

打印的结果说明，如果使用 `ThreadLocal`，那么子线程无法获取父线程的 `ThreadLocal` 的值。相反， `InheritableThreadLocal` 就可以。

---

问： `InheritableThreadLocal` 如果在子线程被修改，那父线程的数据也会变吗？

答：

不会。在 Java 中，`InheritableThreadLocal` 允许子线程“继承”父线程的局部变量值，但这种“继承”是通过复制父线程中的值到子线程来实现的。一旦这个值被复制，父线程和子线程中的 `InheritableThreadLocal` 实例就持有了两份独立的数据。因此，如果子线程修改了它持有的 `InheritableThreadLocal` 中的数据，这个修改不会影响到父线程中的数据，反之亦然。

工作原理：

- 当一个新的线程（子线程）被创建时，Java 虚拟机会检查创建它的线程（父线程）是否有任何 `InheritableThreadLocal` 变量。
- 如果有，那么这些变量的当前值会被复制到新线程中。这个复制过程是值的复制，而不是引用的复制

让我们通过一个简单的示例来说明这一点：

```java
public class InheritableThreadLocalExample {

    private static final InheritableThreadLocal<String> INHERITABLE_THREAD_LOCAL = new InheritableThreadLocal<>();

    public static void main(String[] args) {
        String value = "Original value";
        INHERITABLE_THREAD_LOCAL.set(value);

        new Thread(() -> {
            String childThreadValue = INHERITABLE_THREAD_LOCAL.get();
            System.out.println("Before modification in child thread: " + childThreadValue);

            // 修改子线程中的变量值
            childThreadValue = "Modified value";
            System.out.println("After modification in child thread: " + childThreadValue);
        }).start();

        // 给子线程一点时间进行修改
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // 查看父线程中的变量值是否被修改
        System.out.println("Value in parent thread: " + INHERITABLE_THREAD_LOCAL.get());
    }
}
```

输出结果如下：

```
Before modification in child thread: Original value
After modification in child thread: Modified value
Value in parent thread: Original value
```

也就是说，使用 `InheritableThreadLocal` 的时候，父线程的线程变量会通过 <u>值传递</u> 的方式复制给子线程。

注意：虽然是值传递，但是如果是可变对象（Mutable Objects）的话，传递的是内存地址。所以如果是可变对象，那么子线程是可以对可变对象进行修改的。

这就涉及到两个概念：

- **数据隔离**：虽然 `InheritableThreadLocal` 允许父线程和子线程共享变量，但如果变量指向的是可变对象（如列表或映射），则需要注意，父线程和子线程实际上共享同一个对象实例。这可能导致意外的数据污染，即一个线程对对象的修改会影响到另一个线程。
- **深拷贝**：为了避免上述的数据污染问题，当需要在父子线程之间共享可变对象时，应该在 `InheritableThreadLocal` 的 `childValue` 方法中实现对象的深拷贝，以确保父线程和子线程操作的是两个独立的对象副本。

对于可变对象来说，如果希望子线程不是单纯复制对象的内存地址，而是 <u>深拷贝</u> 的话：

```java
import java.util.ArrayList;

public class InheritableThreadLocalExample {

    // 使用 InheritableThreadLocal 并重写 childValue 方法以实现深拷贝
    // 如果没有重写 childValue 方法，那么子线程对 InheritableThreadLocal 的修改是会影响到父线程的 InheritableThreadLocal
    private static final InheritableThreadLocal<ArrayList<String>> INHERITABLE_THREAD_LOCAL =
            new InheritableThreadLocal<ArrayList<String>>() {
                @Override
                protected ArrayList<String> childValue(ArrayList<String> parentValue) {
                    // 这里实现了 ArrayList 的深拷贝
                    return parentValue == null ? null : new ArrayList<>(parentValue);
                }
            };

    public static void main(String[] args) {
        // 父线程中设置初始值
        ArrayList<String> list = new ArrayList<>();
        list.add("Initial Value");
        INHERITABLE_THREAD_LOCAL.set(list);

        System.out.println("Parent thread initial value: " + INHERITABLE_THREAD_LOCAL.get());

        Thread childThread = new Thread(() -> {
            System.out.println("Child thread value before modification: " + INHERITABLE_THREAD_LOCAL.get());

            // 子线程中修改列表
            INHERITABLE_THREAD_LOCAL.get().add("Modified by child thread");
            System.out.println("Child thread value after modification: " + INHERITABLE_THREAD_LOCAL.get());
        });

        childThread.start();

        // 等待子线程完成，确保输出有序
        try {
            childThread.join();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // 查看父线程中的列表是否受到影响
        System.out.println("Parent thread value after child thread execution: " + INHERITABLE_THREAD_LOCAL.get());
    }
}
```

输出结果：

```
Parent thread initial value: [Initial Value]
Child thread value before modification: [Initial Value]
Child thread value after modification: [Initial Value, Modified by child thread]
Parent thread value after child thread execution: [Initial Value]
```

在例子里，使用了 `InheritableThreadLocal` 来传递一个 `ArrayList<String>` 对象从父线程到子线程，并在 `childValue` 方法中实现了深拷贝。这样，子线程中对列表的修改不会影响父线程中的列表，保持了数据的隔离性。

`childValue` 方法：

- `InheritableThreadLocal` 类中的 `childValue` 方法是实现这种继承行为的关键。
- 当一个新的子线程被创建时，`InheritableThreadLocal` 会自动为这个新线程调用 `childValue` 方法，以确定子线程中的初始值。
- 这个方法接收的参数是父线程中的值，并返回一个值，该值将作为子线程中的初始值。

默认情况下，`childValue` 方法直接返回其参数，即子线程继承的值与父线程中的值相同。 但是可以通过覆盖 `childValue` 方法来自定义子线程应该接收的值，这为不同的继承策略提供了灵活性。

---

> 下文来自 AI

问： `InheritableThreadLocal` 和 `TransmittableThreadLocal` 的关联是？

答：

`InheritableThreadLocal` 和 `TransmittableThreadLocal` 都是解决 Java 多线程编程中线程间上下文传递问题的工具，但它们解决这一问题的方式和适用的场景有所不同。

InheritableThreadLocal：

- `InheritableThreadLocal` 是 Java 标准库提供的一个类，它扩展自 `ThreadLocal`。`ThreadLocal` 用于为每个线程提供一个变量的副本，使得每个线程可以独立地修改自己的副本而不影响其他线程。而 `InheritableThreadLocal` 的特点是允许子线程访问在父线程中设置的值。简单来说，当一个线程创建一个子线程时，`InheritableThreadLocal` 会将父线程中的变量副本传递给子线程。
- `InheritableThreadLocal` 适用于简单的父子线程场景，但在现代复杂的多线程应用中，尤其是涉及线程池的使用时，`InheritableThreadLocal` 会遇到问题。因为线程池中的线程是被重用的，这意味着线程池中的线程并不总是作为新创建的子线程执行，它们不会从创建它们的父线程继承 `InheritableThreadLocal` 变量。

TransmittableThreadLocal:：

- `TransmittableThreadLocal` 是由阿里巴巴开源的一个库 [Transmittable Thread Local (TTL)](https://github.com/alibaba/transmittable-thread-local) 提供的，用于解决 `InheritableThreadLocal` 在异步编程和线程池使用场景中的局限性。`TransmittableThreadLocal` 在 `InheritableThreadLocal` 的基础上，提供了一种机制，能够在使用线程池等并发组件时，正确地传递线程局部变量，包括在任务提交给线程池执行前后，以及线程间的任务调度时。
- `TransmittableThreadLocal` 通过包装 `Runnable` 和 `Callable` 任务，捕获在提交任务时线程上下文（`ThreadLocal`/`InheritableThreadLocal` 的值），然后在任务执行前将这些上下文值恢复到执行任务的线程上，任务执行后再清理这些上下文值。这样，即使在复杂的并发和异步执行环境下，也能保证上下文信息（如用户身份、事务上下文等）的正确传递和隔离。

关联与区别：

- **关联**：`TransmittableThreadLocal` 在 `InheritableThreadLocal` 的基础思想上进行了扩展，以适应更复杂的多线程使用场景，尤其是在异步编程和线程池的使用上。
- **区别**：`InheritableThreadLocal` 适用于简单的父子线程场景，但在线程池重用线程的场景下存在局限。`TransmittableThreadLocal` 解决了这一局限性，提供了一种在现代多线程和异步编程环境中正确传递线程局部变量的解决方案。

总结来说，`TransmittableThreadLocal` 是对 `InheritableThreadLocal` 功能的增强，用于解决在并发编程中线程局部变量传递的问题，特别是在涉及线程池等并发组件的复杂场景下。

---

> 下文来自 AI 问答

问：InheritableThreadLocal 是怎么实现继承父线程中线程局部变量的功能的？

答：

`InheritableThreadLocal` 实现父线程向子线程传递线程局部变量的功能，主要依赖于 Java 的 `Thread` 类中的继承性上下文机制。具体实现方式如下：

Thread 类中的 InheritableThreadLocalMap：

- Java 的 `Thread` 类有两个与 `ThreadLocal` 相关的内部字段：一个是 `threadLocals`，用于存储普通的 `ThreadLocal` 变量；另一个是 `inheritableThreadLocals`，用于存储 `InheritableThreadLocal` 变量。这两个字段都是 `ThreadLocal.ThreadLocalMap` 类型，但只有 `inheritableThreadLocals` 被用来实现父子线程之间的变量传递。

创建子线程时的变量传递过程：

- 当一个新的线程被创建时，Java 虚拟机会调用 `Thread` 类的构造函数。在这个构造函数中，会检查创建新线程的当前线程（即父线程）是否包含 `inheritableThreadLocals`。如果包含，那么这个 `inheritableThreadLocals` 将被复制到新线程中，实现了从父线程到子线程的变量传递。这个复制过程是通过调用 `InheritableThreadLocal` 类中的 `childValue` 方法来完成的，该方法允许对继承的值进行修改或替换。

InheritableThreadLocal 的 childValue 方法

- `InheritableThreadLocal` 类提供了一个受保护的 `childValue` 方法，该方法允许开发者定制子线程从父线程继承的变量值。默认情况下，`childValue` 方法只是简单地返回父线程中的值。然而，开发者可以通过覆盖这个方法来实现自定义的继承逻辑，例如，根据特定的需求修改子线程继承的值。

注意事项：虽然 `InheritableThreadLocal` 提供了一种方便的方式来实现线程间的数据传递，但在使用线程池等并发编程环境中需要小心，因为线程池中的线程通常会被重用，这可能导致预期之外的数据共享问题。为了解决这个问题，可以考虑使用阿里巴巴开源的 `TransmittableThreadLocal`，它在 `InheritableThreadLocal` 的基础上提供了更加灵活和安全的线程局部变量传递机制，特别适合于并发和异步编程环境。

---

> 下文来自 AI 问答

问： `InheritableThreadLocal` 具体怎么配置？

答：

`InheritableThreadLocal` 在 Java 中是一个内建的类，用于创建线程局部变量，其值可以从父线程传递到子线程。这与普通的 `ThreadLocal` 类似，但 `InheritableThreadLocal` 允许子线程访问在父线程中设置的值。由于 `InheritableThreadLocal` 是 Java 标准库的一部分，它本身不需要通过配置来使用，而是通过编码实现的。

然而，在 Spring Boot 或其他框架中，如果你想要确保异步执行的任务能够继承父线程的上下文（例如，安全上下文、数据库事务上下文等），你可能需要一些额外的配置，尤其是当使用 `@Async` 注解或者配置自定义的 `TaskExecutor` 时。以下是一些如何在 Spring 环境中使用类似 `InheritableThreadLocal` 功能的方法：

1. 使用 `@Async` 与 `InheritableThreadLocal`

Spring 框架没有直接提供一个配置选项来启用 `InheritableThreadLocal`。当使用 `@Async` 注解进行异步方法调用时，由于方法可能会在不同的线程中执行，这可能导致 `InheritableThreadLocal` 中的值不会自动传递给这些异步方法。一种解决方法是自定义一个 `Executor`，它能够包装任务以传递 `ThreadLocal` 的值。

2. 自定义 `Executor` 以使用 `InheritableThreadLocal`

你可以创建一个自定义的 `Executor` 或者 `TaskExecutor` 实现，这个实现会使用 `InheritableThreadLocal` 来确保父线程中的上下文能被子线程继承。以下是一个简单的示例：

```java
import java.util.concurrent.Executor;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.core.task.TaskDecorator;

public class InheritableThreadLocalExecutor extends ThreadPoolTaskExecutor {

    public InheritableThreadLocalExecutor() {
        super();
        // 设置 TaskDecorator 来传递 InheritableThreadLocal
        this.setTaskDecorator(new ContextCopyingDecorator());
    }

    private static class ContextCopyingDecorator implements TaskDecorator {
        @Override
        public Runnable decorate(Runnable runnable) {
            // 在这里捕获父线程的 InheritableThreadLocal 值
            Map<String, Object> context = MyContextHolder.getContext().deepCopy();
            return () -> {
                try {
                    // 在子线程中设置上下文
                    MyContextHolder.getContext().set(context);
                    runnable.run();
                } finally {
                    // 清理操作
                    MyContextHolder.getContext().clear();
                }
            };
        }
    }
}
```

在这个示例中，`MyContextHolder` 是一个假设的类，用于管理你的 `InheritableThreadLocal` 上下文。`ContextCopyingDecorator` 是一个 `TaskDecorator`，它在任务执行前后对上下文进行设置和清理。

3. 在 Spring Boot 中配置自定义 `Executor`

一旦你定义了自定义的 `Executor`，你可以在 Spring Boot 的配置类中将其注册为 Bean，以便 Spring 使用它来处理 `@Async` 注解标记的方法：

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        return new InheritableThreadLocalExecutor();
    }
}
```

通过这种方式，你可以确保在使用 Spring 的 `@Async` 功能时，父线程中的上下文（通过 `InheritableThreadLocal` 管理）能够正确地传递到异步执行的任务中。

> 注意事项：使用 `InheritableThreadLocal` 虽然方便，但在某些情况下可能会引入意外的行为，尤其是在复杂的并发场景或者使用线程池（线程复用）的情况下。确保你清楚地了解了其使用场景，并在必要时进行适当的上下文清理。
