# ThreadLocal 

参考资料：[深入JDK源码全面解析ThreadLocal原理 - 马士兵](https://www.bilibili.com/video/BV1ZY4y1P799) （内容相同的视频：[巧用弱引用解决ThreadLocal内存泄漏问题 - 马士兵](https://www.bilibili.com/video/BV19H4y1U7Jc/)）

## 强引用、软引用、弱引用和虚引用

**强引用 Strong Reference**

像 `Object obj = new Object();` 这种引用指向一个 `Object` 对象的，就属于强引用。

只有当没有引用指向这个 `Object` 对象了，这个 `Object` 对象才会被清空，比如 `obj = null;` 。
