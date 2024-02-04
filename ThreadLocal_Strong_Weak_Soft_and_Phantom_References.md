# ThreadLocal 

参考资料：[深入JDK源码全面解析ThreadLocal原理 - 马士兵](https://www.bilibili.com/video/BV1ZY4y1P799) （内容相同的视频：[巧用弱引用解决ThreadLocal内存泄漏问题 - 马士兵](https://www.bilibili.com/video/BV19H4y1U7Jc/)）

## 强引用、软引用、弱引用和虚引用

**强引用 Strong Reference**

像 `Object obj = new Object();` 这种引用指向一个 `Object` 对象的，就属于强引用。

只有当没有引用指向这个 `Object` 对象了，这个 `Object` 对象才会被清空，比如 `obj = null;` 。

---

**软引用 Soft Reference**

软硬用是一个 Java 类型 `SoftReference` ，使用方法如下：

```java
// 10mb 的 byte 数组
SoftReference<byte[]> sr = new SoftReference<>(new byte[1024 * 1024 * 10]);
```

此时，上面的 `sr` 还是强引用，指向的是 `new` 出来的这个 `SoftReference` 对象。

而这个 `SoftReference` 对象才是 <u>软引用</u>，指向的是 10mb 的 byte 数组。可以使用 `get()` 方法获取软引用的对象：

```java
byte[] bytes = sr.get();
```

如果启动的时候，加上 `-Xmx20M` 的 VM 参数来限定 JVM 的大小为 20MB。

当我们使用 `sr.get()` 获取软引用里面的 10MB 的数组时，如果当前 JVM 内存占用小于 20MB，就能成功取到软引用指向的 10MB 数组。

如果 JVM 内存占用大于 20MB，此时不会立刻出现 OOM 错误，而是会释放软引用里面的对象，此时 `sr.get()` 得到的是 `null` ，也就是 10MB 的数组被清除了。

如果有多个软引用，比如此时多个出一个 12MB 的软引用 `SoftReference<byte[]> newSr = new SoftReference<>(new byte[1024 * 1024 * 12]);` ，那么前一个 `sr` 会被清理，也就是 `sr.get()` 为 `null` ，而后一个 `newSr.get()` 能够正常获取到 12MB 的数组。即，清理软引用时，会从最旧的软引用开始清理。

当然，任何时候，如果软引用释放后，还是超过了 JVM 的最大内存空间，就会抛出 OOM 错误。

软引用的使用场景：

- 软引用 Soft Reference 适用于 *缓存* 场景
- 假设我们需要频繁使用一个图片文件，如果每次都从硬盘中读取该图片文件，用完后就释放，这样 IO 消耗很大
- 我们可以使用软引用来缓存这个图片，需要的时候就从软引用中 `get()` 这个图片
- 当内存不够的时候，JVM 会帮我们自动清理这个软引用里面的图片，等我们需要这个图片的时候，再通过 IO 读取图片文件的方式，存到软引用中（判断如果 `get()` 的结果是 `null` ，就从硬盘中读取）
