# 总体划分

![Figure 5-1. The internal architecture of the Java virtual machine.](https://www.artima.com/insidejvm/ed2/images/fig5-1.gif)

[JVM 的运行时数据区域和直接内存](https://blog.csdn.net/clover_lily/article/details/80087162)：

1. JVM 运行时数据区（Runtime Data Areas）：
    * 由 JVM 管理的区域
2. 直接内存（Direct Memory）：
    * JVM 之外的内存，开发人员自己分配 / 回收内存。

When a JVM runs a program, it needs memory to store many things, including <u>bytecodes and other information it extracts from loaded class files, objects the program instantiates, parameters to methods, return values, local variables, and intermediate results of computations</u>. 

JVM organizes the memory it needs <u>to execute a program into several *runtime data areas*</u>. 

**Some runtime data areas are shared among all of an application's threads and others are unique to individual threads.**

参考资料：[Chapter 5 of Inside the Java Virtual Machine by Bill Venners](https://www.artima.com/insidejvm/ed2/jvm2.html)

---

```java
public class test{

    public static void main(String[] args) {
        int a = 1; // 是 main 方法的 Local Variable（局部变量），存在 Stack（栈）内，是会被运行的部分。注意，也只有运行的时候，才会进入 Stack 内。

      	// Teacher 所涉及的 Teacher.class 存放在 Metaspace
      	// john 是 main 方法的 Local Variable，也是一个 Reference，是可以被操作、运行的，所以放在 Stack 中
      	// new Teacher() 生成了 Instance（实例化的对象），该 Instance 存放在 Heap 中
        Teacher john = new Teacher();

        john.stu = new Student(); // john.stu 是 Teacher.class（类）的 Global Variable（全局变量/成员变量）。这里表示，让 Teacher john 的 stu 变量指向 new Student();
    }


    class Teacher {

        String name = "John"; // 实例字段的值存在 Heap 内（& 类的结构信息存在方法区 Method Area 内）
        int age = 40; // 是 global variable，存在 Heap（& Method Area）中
        boolean gender_male = true; // Heap（& Method Area）
        Student stu; // Heap（& Method Area）这个值会指向 Student 对象


        static boolean isHuman = true; // 静态字段（& 类的结构信息存在方法区 Method Area 内）


        public void shout() {
            System.out.println("Teacher is shouting.");
        } 
    }


    class Student {
        String name = "Emily";
        int age = 18;
        boolean gender_male = false;
    }

}
```

# 线程共享部分

**Some Runtime Rata Areas are shared among all threads**

线程共享：

- 所有线程都能访问这块内存数据
- 生命周期贯穿虚拟机 / GC（Garbage Collection）
- 线程共享 = **线程不安全** 

**Each instance of the JVM has one Metaspace and one Heap**. These areas are shared by all threads running inside the VM.

When the VM loads a class file, it parses information about a type from the binary data contained in the class file:

- It **places this type information into the Metaspace** (Method Area)
- As the program runs, the VM **places all objects the program instantiates onto the Heap**

![Figure 5-2. Runtime data areas shared among all threads.](https://www.artima.com/insidejvm/ed2/images/fig5-2.gif)

参考资料：[Chapter 5 of Inside the Java Virtual Machine by Bill Venners](https://www.artima.com/insidejvm/ed2/jvm2.html)

## 元空间 / 方法区（Metaspace / Method Area）

### Metaspace / Method Aread 基础

Metaspace / Method Area（元空间 / 方法区）是被线程共享（线程不安全）的，JDK 7 之前被称为永久带，JDK 8 之后为元空间。

[Metaspace (Method Area) is used to manage memory for **class metadata** ](https://wiki.openjdk.java.net/display/HotSpot/Metaspace):

- Class metadata are allocated when classes are loaded
- Their lifetime is usually scoped to that of the loading classloader - when a loader gets collected, all class metadata it accumulated are released in bulk

Metaspace 主要存储元数据信息：

- `static` 静态变量（static variable / スタティック変数）
- `final` 常量（constant / 定数 | ていすう）
- `Class` 类信息（class 类、interface 接口、enum 枚举、annotation 注解）：
	- 类的构造方法
	- 完整有效的包名+类名
	- 父类的信息
	- 修饰符：public、abstract、final……
	- Field（域）的信息：名称、类型、修饰符
	- Method（方法）的信息：名称、返回类型、参数的数量和类型（按顺序）、修饰符、字节码等
- Runtime Constant Pool（运行时的常量池）

---

```java
public class Test {
    public static Teacher teacher = new Teacher();
}

class Teacher {
}
```

上面的代码中，`static` 修饰的静态变量 `teacher` 是存在于 Metaspace 中的。这个静态变量 `teacher` 实际上存储的是一串内存地址。该内存地址指向被 `new` 出来的 `Teacher` 对象在 Heap 中的位置。

### Runtime Constant Pool 相关知识

Runtime Constant Pool:

- Each class file has a **constant pool** , and each class or interface loaded by the JVM has an internal version of its constant pool called the **Runtime Constant Pool** .
- The **Runtime Constant Pool** is an implementation-specific data structure that <u>maps to the constant pool in the class file</u>.
- Thus, after a Type is initially loaded, <u>all the symbolic references from the Type will reside in the type's Runtime Constant Pool</u>.
- [When creating a class or interface, if the construction of the run-time constant pool requires more memory than can be made available in the method area of the Java Virtual Machine, the Java Virtual Machine throws an `OutOfMemoryError`.](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.5)

Constant Pool:

> 比如：String str = new String("abc"); 会 new 一个新的对象，str 指向该对象。
>
> 而 String str = "abc"; 会将该字符串直接放入常量池中，str 直接指向常量池中的 "abc"。

- A class file keeps all its symbolic references in one place, the **constant pool** .
- [Simply put, a **constant pool** contains the constants that are needed to run the code of a specific class](https://www.baeldung.com/jvm-constant-pool).
- Basically, it's a runtime data structure similar to the symbol table. It is a per-class or per-interface runtime representation in a Java class file.
- The content of the constant pool consists of symbolic references generated by the compiler.
	- [常量池包含两部分：**字面值**（Literal）和**符号引用**（Symbolic Reference）](https://segmentfault.com/a/1190000004541386)
	- <u>Literal</u> 可以理解为 Java 中定义的字符串常量、final 常量等
	- <u>Symbolic Reference</u> 指的是一些字符串，这些字符串表示当前类引用的外部类、方法、变量等的引用地址的抽象表示形式。在类被 JVM 装载并第一次使用这些 Symbolic Reference 时，这些符号引用（Symbolic Reference）将会解析为直接引用
- These references are names of variables, methods, interfaces, and classes referenced from the code.
- The JVM uses them to link the code with other classes it depends on.

[Symbolic Reference（符号引用）具体包含](https://zhuanlan.zhihu.com/p/370870361)：

- 被模块导出或者开放的包（Package）
- 类和接口的全限定名（Fully Qualified Name）
- 字段的名称和描述符（Descriptor）
- 方法的名称和描述符
- 方法句柄和方法类型（Method Handle、Method Type、Invoke Dynamic）
- 动态调用点和动态常量（Dynamically-Computed Call Site、Dynamically-Computed Constant）

### 设置 Metaspace / Method Aread 的大小

[**设置 Metaspace / Method Aread 的大小**](https://www.cnblogs.com/ruoli-0/p/14275977.html[)

**JDK7及以前**

- 通过 `-xx:Permsize` 来设置永久代初始分配空间。默认值是20.75M。
- `-XX:MaxPermsize` 来设定永久代最大可分配空间。32 位机器默认是 64M，64 位机是 82M。
- 当 JVM 加载的类信息容量超过了这个值，会报异常 OutofMemoryError:PermGen space。

![img](https://img2020.cnblogs.com/blog/2123988/202101/2123988-20210114111828173-264799550.png)

**JDK8以后**

Metaspace 的大小可以使用 **参数 `-XX:MetaspaceSize` 和 `-XX:MaxMetaspaceSize` 指定** ：

- `-XX:MetaspaceSize` 设置 Metaspace 的<u>初始大小</u>
	- 对于一个 64 位的服务器端 JVM 来说，其默认值为 21MB
	- 一旦触及这个大小，就会触发 FullGC 并卸载无用的 Class（即，这些 Class 对应的 ClassLoader 不再存活）
- 在经过了 FullGC 后，Metaspace 的<u>初始大小</u>会被重置。新的 size 大小取决于 FullGC 释放了多少空间：
	- 如果 GC 释放的空间不足，则适当提高该值，不过也不能超过 `-XX:MaxMetaspaceSize` 指定的值
	- 如果 GC 释放空间过多，则适当降低该值
- 如果 `-XX:MetaspaceSize` 设置的过低，触发了多次 FullGC，那么应该将其设置为更高的值

如果 Metaspace 发生溢出，JVM 会抛出异常`OutOfMemoryError:Metaspace` 



## 堆内存 Java Heap

### Heap 基础

The Java heap is where the objects of a Java program live. It is a repository for live objects, dead objects, and free memory

Heap 分为：

- Young Generation（年轻代）：占据 Heap 的 1/3，且 Young Generation 还可以分为：
	- Eden：默认占据 Young Generation 的 8/10
	- Survivor 0：缩写为 S0
		- 可以是 from，也可以是 to
		- 默认占据 Young Generation 的 1/10
	- Survivor 1：缩写 S1，其余同上
- Old Generation（老年代）：占据 Heap 的 2/3

### Object 在 Heap 中的生命周期

Object 在 Heap 中的生命周期：

1. 当 `new` 一个 Object 出来的时候，这个 Object 优先 be allocated in Eden
2. 当 Eden 放不下后，[Execution Engine](https://www.geeksforgeeks.org/execution-engine-in-java/)（JVM 字节码执行引擎）会开启一个 MinorGC 垃圾收集线程，来清理整个 Young Generation
3. 在 Minor GC 过后依旧存留的 Object，就会放到当前的 Survivor（to）中

	- 在 Object 被放到当前的 Survivor（to）后，当前的 Survivor（to）就变为了 Survivor（from）

	- 同时，之前的 Survivor（from）就变为了 Survivor（to）

	- 只要记住，「to」是空的。只要存放了 Object 后，「to」就会变为「from」
4. 在 Survivor Space 中的 Object，每经历一次 Minor GC 后如果存活了下来，就会从「from」转移到「to」，且该 Object 的 **age** 就会 increase 1

	- 简单来说，age (分代年龄) means number of collection survived
	- age 存储在 Object Header 中
5. If the **age** is above the current **tenuring threshold** , it would <u>gets tenured into (be promoted to) Old Generation</u>

	- tenuring threshold 的参数是 `-XX:MaxTenuringThreshold` ，默认值为 15

	- The Java command line parameter `-XX:MaxTenuringThreshold` specifies for how many minor GC cycles an object will stay in the survivor spaces until it finally gets tenured into the old space.
6. 当 Old Gen 放满了之后，还要继续放 Object 的话，Execution Engine 就会开启一个 Full GC 的线程，对整个 Heap 进行垃圾回收处理
7. 如果 Old Gen 在进行了 Full GC 后还是满的，此时又有新的 Object 要放入 Old Gen 中，就会触发 `Out Of Memory Error`

### Object 在 Heap 中的其他情况

Object 要进入 Old Gen，除了 Object 的 age 超过 `-XX:MaxTenuringThreshold` 的数值外，还有其他情况，比如：

- 当 Object 的 size 超过了 `XX:PretenureSizeThreshold` 设置的大小时，会直接进入 Old Gen
- The Object could also be promoted to the Old Gen directly if the Survivor Space gets full (overflow)

Premature Tenuring (Premature Promotion) ：

- 也叫 *动态计算晋升年龄阈值* 或 *动态对象年龄判定*  ，因为 the actual <u>tenuring threshold</u> is <u>dynamically adjusted</u> by JVM
- When the Survivor Space（可能是 S0 也可能是 S1，总之是当前存放 Object 的 "from"） usage has reached target ratio defined by `-XX:TargetSurvivorRatio`  
	- The Serial collector, uses `-XX:TargetSurvivorRatio=50` as the default
- 那么此时，在该 Survivor Space ("from") 中的这批 Objects 里面，如果有 Objects **大于等于** 这批 Objects 的<u>特定年龄（age）</u>，就会直接进入 Old Gen：
	- [age 从小到大的 Objects 占据的空间，如果大于 Survivor（S0 或 S1）的 TargetSurvivorRatio，那么就把大于等于该 age 的 Objects，放入到 Old Gen](https://segmentfault.com/a/1190000039805691)
	- 如果 <u>年龄 1 + 年龄 2 + 年龄 3 + 年龄 N</u> 的 Objects 加起来的空间，大于 `-XX:TargetSurvivorRatio` 设定的百分比时，<u>年龄 N</u> 和 <u>年龄 N 以上</u> 的 Objects 进入 Old Gen
	- [如果出现这种情况](https://www.jianshu.com/p/989d3b06a49d)：<u>年龄 1 占用了 33%</u>，<u>年龄 2 的占用了 33%</u>，累加和超过默认的 TargetSurvivorRatio（50%），那么，<u>年龄 2 和年龄 3 的 Objects 都要到 Old Gen</u>

空间分配担保机制：

- 如果在发生 Minor GC 之前，JVM 检查到 <u>Old Gen剩余的可用的连续空间</u> **小于** <u>Young Gen 所有对象（包括 Garbage 对象）的总空间</u>
- 那么，如果此时 `HandlePromotionFailure=true` ，那么会继续检查 <u>Old Gen 剩余的可用连续空间</u> **是否大于** <u>历次晋升到 Old Gen 的 Object 的平均大小</u>
- **如果大于** ，则尝试进行一次 Minor GC（这次 Minor GC 可能会触发 Full GC）
- **如果小于或者 `HandlePromotionFailure=false`** ，则进行一次 Full GC

参考资料：

- [Do Not Set -XX:MaxTenuringThreshold to a Value Greater Than 15 (Doc ID 1283267.1)](https://support.oracle.com/knowledge/Middleware/1283267_1.html)
- [MaxTenuringThreshold - how exactly it works?](https://stackoverflow.com/a/13549337)
- [Execution Engine In Java](https://www.geeksforgeeks.org/execution-engine-in-java/) 或 [在 GitHub上查看](https://github.com/LearnDifferent/my-notes/blob/full/ExecutionEngineInJava.md)

### OOM

OOM：`java.lang.OutOfMemoryError: Java heap space`

```java
public class ObjectTest {

    public static void main(String[] args) {

        List<ObjectTest> list = new ArrayList<>();

        while (true) {
            list.add(new ObjectTest());
            System.out.println(list.size());
        }
    }
}
```

在上面的代码中：

- `List<ObjectTest> list = new ArrayList<>();` 表示 <u>`list` (local variable)</u> 引用了 <u>`ArrayList` (对象)</u>
- `list.add(new ObjectTest());` 表示 <u>`ArrayList` (局部变量 list 实际指向的 ArrayList 对象)</u> 引用了 <u>`ObjectTest` (对象)</u>

这样的话，所有 `list` 中的 `ObjectTest` 对象就永远都无法被 GC 回收：

- `list` 是当前的 GC Root，而 `list` 引用了 `ArrayList`，`ArrayList` 又引用了 `ObjectTest` 

- 也就是说，每次 `new` 出来的 `ObjectTest` 会被 `ArrayList` 引用，而 `ArrayList` 又会被 `list` （也就是 GC Root）引用，所以无法回收

> 如果想早点触发 OOM 的话，可以在 IDEA 中添加 VM Option `-Xms2m -Xmx8m` 后再运行。

如果想让这个 Java 程序，在触发 OOM 后，生成一个 Heap 信息，可以继续添加 `-XX:+HeapDumpOnOutOfMemoryError` 参数。当 OOM 后，会生成一个类似 `java_pid1234.hprof` 的文件，可以在 IDEA 双击打开，IDEA 会使用自带的 Profiler 来分析该文件也可以使用其他的工具打开改文件，可以参考[A Guide to Java Profilers](https://www.baeldung.com/java-profilers)。

# 线程独占部分

**Some Runtime Data Areas are unique to individual threads**

线程独占部分：

- 每个线程都有独立空间
- 生命周期等于线程的生命周期
- 线程独占 = 线程私有 = **线程安全 Thread Safety** （Thread-safe / スレッドセーフ）

As each new thread comes into existence, it gets its own *pc register* (program counter) and *Java stack*.

![Figure 5-3. Runtime data areas exclusive to each thread.](https://www.artima.com/insidejvm/ed2/images/fig5-3.gif)

> 图：Figure 5-3. Runtime data areas exclusive to each thread.

Figure 5-3 shows a snapshot of a virtual machine instance in which three threads are executing. At the instant of the snapshot, threads one and two are executing Java methods. Thread three is executing a native method.

In Figure 5-3, as in all graphical depictions of the Java stack in this book, the <u>stacks are shown growing downwards</u>. <u>The "top" of each stack is shown at the bottom of the figure</u>. 

Stack frames for currently executing methods are shown in a lighter shade. For threads that are currently executing a Java method, the <u>pc register indicates the next instruction to execute</u>. In Figure 5-3, such pc registers (the ones for threads one and two) are shown in a lighter shade. Because thread three is currently executing a native method, the contents of its pc register--the one shown in dark gray--is undefined.

参考资料：[Chapter 5 of Inside the Java Virtual Machine by Bill Venners](https://www.artima.com/insidejvm/ed2/jvm2.html)

## 虚拟机栈 Java Virtual Machine Stacks

### JVM Stack 基础

**Java Virtual Machine Stacks・仮想マシン・スタック**

A thread's **Java stack stores the state of Java (not native) method invocations for the thread** . 

The <u>state of a Java method invocation</u> includes its **local variables, the parameters with which it was invoked, its return value (if any), and intermediate calculations** .

> The state of native method invocations is stored in an implementation-dependent way in *native method stacks*, as well as possibly in registers or other implementation-dependent memory areas
>
> 参考：Native Method Stacks

JVM Stacks，可以理解简单理解为“线程栈”（Thread Stacks）：

- 每个 Thread 运行的时候，Stacks 都会开辟一个区域给 Thread，Thread 的每个方法，每个方法需要的每个 Local Variables 等，都会在 Stacks 中按照顺序执行

也就是说，**只要 Thread 开始运行，JVM 就会分配一个专属的内存空间给该 Thread。在 Thread 上运行的 Method 所需要的 Local Variables 及其他 Thread 相关的数据，也会被存在该内存空间中。该内存空间就是 Stack(s)。**

### Stack Frame 栈帧

[Stack Frames](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6)

The Java stack is composed of **stack frames** (or *frames*). 

**A stack frame is used to store data and partial results, as well as to perform dynamic linking, return values for methods, and dispatch exceptions.**

Stack frames are allocated from the [Java Virtual Machine stack](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.2) of the thread creating the frame. A stack frame contains the state of one Java method invocation:

- When a thread invokes a method, the JVM pushes a new frame onto that thread's Java stack
- When the method completes, the VM pops and discards the frame for that method, whether that completion is normal or abrupt (it throws an uncaught exception)

Each stack frames has its own array of [local variables](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6.1) , its own [operand stack](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6.2), and a [reference to the run-time constant pool](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.5) of the class of the current method. 

> A stack frame may be extended with additional implementation-specific information, such as debugging information.（展示异常错误信息的时候，可以使用 JVM 的栈）

The **sizes** of the **local variable array** and the **operand stack** are <u>determined at compile-time</u> and are <u>supplied along with the code for the method associated with the stack frame</u>. 

Thus **the size of the stack frame data structure** depends only on <u>the implementation of the JVM</u>, and **the memory for these structures** can be <u>allocated simultaneously on method invocation</u>.

Current Frame :

- Only one frame, **the stack frame for the executing method**, is <u>active at any point in a given thread of control</u>.
- This frame is referred to as the **current frame** , and its method is known as the **current method** . The class in which the current method is defined is the **current class** . 
- <u>Operations on local variables and the operand stack</u> are typically with <u>reference to the current frame</u>.
- A frame **ceases to be current** if <u>its method invokes another method</u> or if <u>its method completes</u>.
- **When a method is invoked, a new frame is created and becomes current when control transfers to the new method** .
- **On method return, the current frame passes back the result of its method invocation, if any, to the previous frame. The current frame is then discarded as the previous frame becomes the current one.**

Note that a stack frame created by a thread is <u>local to that thread</u> and <u>cannot be referenced by any other thread</u>.

---

### Local Variable Array / Local Variables 局部变量

[Local Variables](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6.1)

Each Stack Frame contains an array of variables known as its **Local Variables** .

The length of the local variable array of a frame is determined at compile-time and supplied in the binary representation of a class or interface along with the code for the method associated with the frame.

Local Variable 可以存储：

- A **single local variable** can hold a value of type `boolean`, `byte`, `char`, `short`, `int`, `float`, `reference`, or `returnAddress`. 
- A **pair of local variables** can hold a value of type `long` or `double`. 
- Note: A value of type `long` or type `double` <u>occupie two consecutive</u> local variables.

**Local Variables 是以 Array（数组 / 配列）的形式存在的，所以也被称为 Local Variable Array (LVA)** ：Local variables are addressed by indexing. The index of the first local variable is zero. An integer is considered to be an index into the local variable array if and only if that integer is between zero and one less than the size of the local variable array.

**The JVM uses local variables to pass parameters on method invocation** : 

- On class method invocation, any parameters are passed in consecutive local variables starting from local variable *0*. 
- On instance method invocation, local variable *0* is always used to pass a reference to the object on which the instance method is being invoked (`this` in the Java programming language). 
- Any parameters are subsequently passed in consecutive local variables starting from local variable *1*.

在下面的代码中：

```java
public class Test {
    public static void main(String[] args) {
        Teacher teacher = new Teacher();
    }
}

class Teacher {
}
```

执行  `main` 方法的 Stack Frame，会有一个 `teacher` 的 Local Variable，该 Local Variable 会存储一个内存地址。该内存地址，就是 `new` 出来的 `Teacher` 对象在 Heap 中的位置。

也就是说， **JVM Stack 中的 Stack Frame 里面的 Local Variable，存放的是指向 Heap 中的 Object 的内存地址**

### Operand Stack 操作数栈

[Operand Stack](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6.2)

操作数栈：

* 定义：临时存放并操作 value（数值）的内存空间
* 存放 value：JVM 先把某个 value 放入 Operand Stack 中
* 操作 value：如果需要赋值某个 Local Variable，就将之前存入的 value 该 Local Variable 中
* 操作 value：如果需要计算，Operand Stack 就会弹出最近的 2 个 value，然后根据加减乘除对这 2 个 value 进行运算操作，得到结束后重新押回 Operand Stack 中

Each frame contains a last-in-first-out (LIFO) stack known as its **operand stack** . The maximum depth of the operand stack of a frame is determined at compile-time and is supplied along with the code for the method associated with the frame.

> Where it is clear by context, we will sometimes refer to the operand stack of the current frame as simply the operand stack.

The operand stack is <u>empty when the frame that contains it is created</u>. 

**JVM supplies instructions to load constants or values from local variables or fields onto the operand stack. Other JVM instructions take operands from the operand stack, operate on them, and push the result back onto the operand stack. The operand stack is also used to prepare parameters to be passed to methods and to receive method results.**

> For example, the [*iadd*](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.iadd) instruction adds two `int` values together. It requires that the `int` values to be added be the top two values of the operand stack, pushed there by previous instructions. Both of the `int` values are popped from the operand stack. They are added, and their sum is pushed back onto the operand stack. Subcomputations may be nested on the operand stack, resulting in values that can be used by the encompassing computation.

Each entry on the operand stack can <u>hold a value of any JVM type</u>, including a value of type `long` or type `double`.

**Values from the operand stack must be operated upon in ways appropriate to their types** . It is not possible, for example, to push two `int` values and subsequently treat them as a `long` or to push two `float` values and subsequently add them with an [*iadd*](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.iadd) instruction. 

> A small number of JVM instructions (the [*dup*](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.dup) instructions and [*swap*](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.swap)) operate on run-time data areas as raw values without regard to their specific types; these instructions are defined in such a way that they cannot be used to modify or break up individual values. These restrictions on operand stack manipulation are enforced through `class` file verification.

At any point in time, an operand stack has an associated depth, where a value of type `long` or `double` contributes two units to the depth and a value of any other type contributes one unit.

### Dynamic Linking 动态引用

[Dynamic Linking](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6.3)

<u>Each Stack Frame contains a reference to the run-time constant pool for the type of the current method to support **dynamic linking** of the method code</u>. 

The `class` file code for a method refers to methods to be invoked and variables to be accessed via symbolic references. **Dynamic linking translates these symbolic method references into concrete method references** , loading classes as necessary to resolve as-yet-undefined symbols, and translates variable accesses into appropriate offsets in storage structures associated with the run-time location of these variables.

This late binding of the methods and variables makes changes in other classes that a method uses less likely to break this code.

## 程序计数器 Program Counter Register

If the thread is executing a Java method (not a native method), <u>the value of the pc register indicates the next instruction to execute</u>. 

假设在运行一个线程的某个方法时候，有优先级更高的线程抢占了 CPU 资源，此时之前的线程被挂起，等 CPU 资源空闲后再继续，为了继续线程的时候能回到之前执行的方法，就需要 *程序计数器*

其实就是一个指向 *方法区* 的指针，该指针具体指向程序当前运行的位置。
```
0: aload_1
1: invokevirtual
4: pop
5: return
```

比如上面的字节码中的 `0: aload_1` ：

- `0` 就是字节码指令的行号（位置）
- `aload_1` 就是需要运行的字节码指令
- 「字节码指令」对应的是：「指令方法」在方法区内的「Memory Address（内存地址）」

[PC Register 为什么存放的不是「指令本身」而是「指令的存放地址」？](https://www.zhihu.com/question/318129637/answer/637300541)

- 因为除了程序计数器（PC, PC Register）外还有个指令寄存器（IR, Instruction Register）
- IR 是存放当前执行指令的，而 PC 指向的是下条执行指令的地址
- 所以，PC 用地址取值的方式，便于执行跳转指令

[PC Register](https://app.yinxiang.com/shard/s72/nl/16998849/c39de305-2e8a-4989-bf7b-ca4ec847d4d1/)：

> 程序计数器其实就是一个指针，它指向了我们程序中下一句需要执行的指令，它也是内存区域中唯一一个不会出现 OutOfMemoryError 的区域，而且占用内存空间小到基本可以忽略不计。这个内存仅代表当前线程所执行的字节码的行号指示器，字节码解析器通过改变这个计数器的值选取下一条需要执行的字节码指令。如果执行的是 native 方法，那这个指针就不工作了。

## 本地方法栈 Native Method Stack

Native Method Stack 是存放 JVM 底层的调用 C 和 C++ 实现的 native 方法的区域。

这些 native 方法也需要内存空间，所以在调用到这些方法的时候，JVM 会腾出一块 Native Method Stack 给这些方法。

只要带了 `native` 关键字，说明 Java 超过了 Java 的作用范围，会进入 Native Method Stack（本地方法栈），然后调用 JNI。

 JNI：

- Java Native Interface / 本地方法接口
- 用于登记 native 方法
- 在执行的时候，通过 JNI，加载本地方法库中的方法

> [Not all JVMs support native methods, however, those that do typically create a per thread native method stack.](https://app.yinxiang.com/shard/s72/nl/16998849/6c2c243c-cb85-491a-839b-09be980d50e2/)

> [本地方法栈(Native Method Stack)和Java虚拟机栈类似，区别在于Java虚拟机栈是为了Java方法服务的，而本地方法栈是为了native方法服务的。在虚拟机规范中并没有对本地方法实现所采用的编程语言与数据结构采取强制规定，因此不同的JVM虚拟机可以自己实现自己的native方法。](https://app.yinxiang.com/shard/s72/nl/16998849/74009fbe-a516-41e2-8920-f48cc4593957/)



# 草稿

## Stack

摘抄：[Java Virtual Machine (JVM) Stack Area](https://www.geeksforgeeks.org/java-virtual-machine-jvm-stack-area/)

Java Virtual Machine Stacks・

> For every thread, JVM creates a separate stack at the time of thread creation. The memory for a Java Virtual Machine stack does not need to be contiguous. The Java virtual machine only performs two operations directly on Java Stacks: it pushes and pops frames. And stack for a particular thread may be termed as *Run – Time Stack*.
>
> Each and every method call performed by that thread is stored in the corresponding *run time stack* including parameters, local variables, intermediate computations, and other data.
>
> After completing a method, corresponding entry from the stack is removed. After completing all method calls the stack becomes empty and that empty stack is destroyed by the JVM just before terminating the thread.
>
> The data stored in the stack is available for the corresponding thread and not available to the remaining threads. Hence we can say local data is thread safe. Each entry in the stack is called *Stack Frame* or Activation Record.

![](https://media.geeksforgeeks.org/wp-content/uploads/JVM.jpg)

Stack Frame Structure

> The stack frame basically consists of three parts: Local Variable Array, Operand Stack & Frame Data.
>
> When JVM invokes a Java method, first it checks the class data to determine the number of words (size of the local variable array and operand stack, which are measured in words for each individual method) required by the method in the local variables array and operand stack. It creates a stack frame of the proper size for invoked method and pushes it onto the Java stack.

1. 局部变量表 Local Variable Table/Array (LVA):
	* The local variables part of stack frame is organized as a zero-based array of words.
	* It contains all parameters and local variables of the method.
	* 包含了<u>**本地变量（Local Variables/Vars）、输入参数和输出参数（Parameters）以及方法内的变量**</u>
2. 栈操作 Operand Stack (OS):
	* JVM uses operand stack as work space like rough work or we can say for storing intermediate calculation’s result.
	* Operand stack is organized as array of words like local variable array. But this is not accessed by using index like local variable array rather it is accessed by some instructions that can push the value to the operand stack and some instructions that can pop values from operand stack and some instructions that can perform required operations.
	* **记录出栈、入栈的操作**
3. 栈帧数据 Frame Data (FD):
	* It contains all symbolic reference (constant pool resolution) and normal method return related to that particular method.
	* It also contains a reference to Exception table which provide the corresponding catch block information in the case of exceptions.

***

下面的是草稿，等待整理……

一些名词：栈帧结构、方法索引（method index）、输入输出参数（Parameters）、本地变量（Local vars）、类（Class）、父帧（Return Frame）、子帧（Next Frame）

> 栈分为<u>栈顶</u>和<u>栈底</u>，每一个栈压住的程序正在执行的方法。

> 栈帧：是一个内存区块，是一个数据集，是一个有关方法和运行期数据集

> 如果栈满了，就会出现 StackOverflowError。

栈 + 堆 + 方法区的交互关系：

* 栈帧里面的引用，指向了堆里面的对象具体实例
* 对象具体实例里面的`final`常量，指向方法区

知识点：

* 栈 Stack：

	* 定义：我们编写的每一个 method 都会放到 Stack 里面运行
	* Student stu = new Student();
	* stu 为 *Stack / 栈*，是会被执行操作的部分
		* Stack 引用 Heap / Stack -> Heap
	* 平时说的 Stack 指的就是 Java Virtual Machine Stacks（JVM Stacks）

* 栈帧 Stack Frame：

	* 在 Stack 内，表示被执行的一个方法。
	* 如果该 Stack Frame 调用了另外一个方法，就会生成新的 Stack Frame。
	* 以此类推，所有 Stack Frame 就会像 stack 一样，先进后出（最后被调用的方法，最先被执行）。

* 局部变量表 Local Variable Table 与栈帧 Stack Frame：

	* 当一个方法被执行时，会生成一个 Stack Frame，这个 Stack Frame 会生成一个 Local Variable Table，把所有的 Local Variable 压缩进来。
	* 这就是局部变量 Local Variable 也属于栈内到原因

* 操作数栈 Operand Stack 与栈帧 Stack Frame：

	* [当一个方法刚刚开始执行时，其操作数栈是空的，随着方法执行和字节码指令的执行，会从局部变量表或对象实例的字段中复制常量或变量写入到操作数栈，再随着计算的进行将栈中元素出栈到局部变量表或者返回给方法调用者，也就是出栈/入栈操作。一个完整的方法执行期间往往包含多个这样出栈/入栈的过程。](https://zhuanlan.zhihu.com/p/45354152)
	* [操作数栈(Operand Stack)也常称为操作栈，它是一个后入先出栈(LIFO)。同局部变量表一样，操作数栈的最大深度也在编译的时候写入到方法的Code属性的max_stacks数据项中。操作数栈的每一个元素可以是任意Java数据类型，32位的数据类型占一个栈容量，64位的数据类型占2个栈容量,且在方法执行的任意时刻，操作数栈的深度都不会超过max_stacks中设置的最大值。](https://zhuanlan.zhihu.com/p/45354152)
	* [**The operand stack is used during the execution of byte code instructions** in a similar way that general-purpose registers are used in a native CPU. Most JVM byte code spends its time manipulating the operand stack by pushing, popping, duplicating, swapping, or executing operations that produce or consume values. Therefore, instructions that move values between the array of local variables and the operand stack are very frequent in byte code. For example, a simple variable initialization results in two byte codes that interact with the operand stack.](https://app.yinxiang.com/shard/s72/nl/16998849/6c2c243c-cb85-491a-839b-09be980d50e2/)

* 动态引用 Dynamic Linking 与栈帧 Stack Frame：

	* 动态引用：存在于 Stack Frame 内，用于找到方法区 Method Area / Metaspace 里面的方法代码

* Return Address（返回值地址、方法出口）  与栈帧 Stack Frame：

	* 一个 Stack Frame /运行的方法在结束后，会得到一个值，这个值要返回到之前的方法内：
		* main 方法的 Stack Frame 里面有一个 int result = calculate();
		* 此时会生成 calculate() 方法的 Stack Frame，然后优先计算出结果。这个结果需要返回到 main 方法内，赋值给 main 方法的 int result
	* Return Address 就是在 Stack Frame 划出一块区域，用于储存需要 return 的前一个 stack 的内存地址

## Heap

> 栈主要管运行，堆主要管存储对象

* **Heap** 主要**存储对象**
* **Stack** 存储的主要是**对象的引用类型**，也就是**对象的地址**，最终要指向 Heap 实际存在的对象
* 注意！JDK 6 之后，JVM 通过逃逸分析（エスケープ解析/Escape analysis），发现一个对象在声明之后，只有在它当前运行的函数中调用：
	* JVM 就会在 **Stack** 上申请空间放置这个对象，而不是 **Heap** 上
	* 函数执行完毕后，会直接清理这个对象，这样可以减轻 GC 的压力

例子：User u = new User();

* `new` 关键字会在 Heap 中创建新的 User 对象
* `u` 则会在 Stack 上创建引用（Reference），该引用内的地址信息（也就是 *存放地址* ），会指向 Heap 中的 User 对象

堆内存也是被线程共享的。

* 堆的定义：
	* Student stu = new Student();
	* new Student() 为 *Heap / 堆*，是被引用到部分
		* Stack 引用 Heap / Stack -> Heap
* 作用：存放对象实例 Instance。几乎所有的对象、数组都存在这里。

> 详细请看 GC 部分的

## 对比 JVM Stack 和 Heap

Stack 主要管运行，Heap 主要管存储。

Stack 存放的内容和程序运行相关，主要存储函数运行过程中的临时变量。

* **Stack** 存储的是**对象的引用类型**，也就是**对象的地址**，最终要指向 Heap 实际存在的对象
* **Heap** 主要**存储对象**
* 注意！JDK 6 之后，JVM 通过逃逸分析（エスケープ解析/Escape analysis），发现一个对象在声明之后，只有在它当前运行的函数中调用：
	* JVM 就会在 **Stack** 上申请空间放置这个对象，而不是 **Heap** 上
	* 函数执行完毕后，会直接清理这个对象，这样可以减轻 GC 的压力

例子：User u = new User();

* `new` 关键字会在 Heap 中创建新的 User 对象
* `u` 则会在 Stack 上创建引用（Reference），该引用内的地址信息，会指向 Heap 中的 User 对象

> 栈内存不存在垃圾回收问题

栈内存，主要管理程序的运行，生命周期和线程同步：

* 线程结束的时候，栈内存也就释放了（注意，main 也是一个进程）
* 首先在 Stack 内压入 main 方法，然后压入方法（比如 `test()`）
* 在取出来的过程中，如果 main 方法也被取出来，stack 空了之后，程序就结束了

## JVM 的位置

JRE（包含了 JVM）在 OS 之上，OS 在硬件体系之上。

## JVM 的体系结构

运作流程：

* 最开始的文件是 `.java`，通过 `java -c` 命令编译成 `.class` 文件，然后通过 `Class Loader`/类加载器，加载到 JVM 的 运行时数据区/`Runtime Data Area`。

Runtime Data Area：

- Method Area/方法区
- (Java) Stack Area
- Native Method Stack
- Heap Area
- PC 计数器

> 垃圾回收只会发生在 Method Area 和 Heap，JVM 调优基本上就是调整这两个地方。

## 类加载器和双亲委派机制

ClassLoader 的作用：

* 加载 class 文件

> 类是抽象的，对象是具体的（new 关键字）。

双亲委派机制（親委譲モデル/Parents Delegation Model）：APP->EXC->BOOT（最终执行）

* AppClassLoader 的父类是 ExtClassLoader，而 ExtClassLoader 的父类是 BootClassLoader
* 如果在最高的 BOOT 执行了，就不会去底下的执行
* 如果 BOOT 和 EXC 加载器都没有的情况下，就使用当前的 APP 加载器

流程：

1. 类加载器收到类加载的请求
2. 将这个请求向上委托给父加载类，指导启动类加载器/根加载器/Boot
3. 启动类加载器检查是否能够加载当前这个类，能加载就使用当前的加载器。否则，抛出异常，通知子加载器进行加载
4. 重复上一步

> 这就是 Class Not Found 的原理

## sandbox/沙箱/沙盒安全机制

基本组件：

- 字节码校验器/Bytecode verifier：确保 Java 类文件遵循 Java 语言规范，保护内存。像 java. 和 javax. 这样的类就不会经过 bytecode verifer
- 类加载器/class loader：
	1. 使用双亲委派机制，防止恶意代码干涉善意代码，也就是不让你修改 Java 原来的代码
	2. 守护了信任的类库的边界
	3. 将代码归入保护域，确定了代码可以进行哪些操作

类加载器内有：
- 存取控制器/access controller：可以控制核心 API 堆操作系统的存取权限，也就是可以操作我们的 OS（比如使用 `new Robot()`）
- 安全管理器/security manager：实现权限控制，比 access controller 优先级高
- 安全软件包/security package： java.security 下的类和扩张包下的类，允许用户为应用增加新的安全特性



## 三种 JVM

1. Sun 的 HotSpot
2. BEA 的 JRockit
3. IBM 的 J9 VM

## GC

> GC 的作用区域只有方法区和堆
>
> 基本上回收都是新生区

## 堆（Heap）

Heap：一个 JVM 只有一个堆内存。堆内存的大小是可以调节的。

类加载器读取了类文件后，会把类、方法、常量和变量放进堆中，堆会帮我们保存引用类型的真实对象。

JDK 8 之前堆内存中分为三个区域：

- Young Generation Space（新生区）：
	1. Eden Space（伊甸园区）
	2. Survivor Space
		1. Survivor Space 0
			* 可以简写为 S0，中文是：幸存 0 区
			* 根据情况，可以叫做 From Survivor Space 或 To Survivor Space
    	2. Survivor Space 1
			* 可以简写为 S1，中文是：幸存 1 区
			* 根据情况，可以叫做 From Survivor Space 或 To
- Old Generation Space / Tenured Space（养老区/老年区/老年代）
- Permanent Generation(non-heap)：永久区/永久存储区，其实不在堆上（JKD 8 后叫元空间）

> 在 JDK 8 以后，永久存储区改名为元空间。

幸存区是新生区和养老区之间的过渡，其中的幸存 0 区和幸存 1 区会动态交互。

垃圾回收主要：

* 轻 GC（Minor GC/Young GC） -> 新生区（主要是在伊甸园区）
* 重 GC（Major GC/Full GC） -> 养老区

> 如果堆内存满了，就会 OOM/OutOFMemoryError


新生区：
- 类：诞生和成长的地方，大部分情况下还会死亡
- 伊甸园区：所有的对象都是在这里 new 出来的
- 幸存区（From, to）：99% 的对象是临时对象，在幸存区就会死去，不会进入养老区

元空间/永久区（方法区在这里，方法区里面的常量池肯定也在这里）：
- 用来存放 JDK 自身携带的 Class 对象、interface 元数据。
- 也就是用来存储 Java 运行时的环境或类信息
- 这个区域不存在垃圾回收。关闭 JVM 就会释放这个区域的内存
- 一个启动类加载了大量第三方 jar 包或者 tomcat 部署了太多的应用，或者加载了大量动态生成的反射类，才会在这个区域出现 OOM

元空间，逻辑上存在于堆中，物理上不存在于堆中，所以也被叫做「非堆」。（物理上新生区和老年区基本占用了全部堆内存）

默认情况下，分配的总内存是电脑内存的 1/4，初始化的内存是电脑内存的 1/64

如果出现了 OOM：
1. 用参数调优，扩大堆内存查看结果
2. 分析内存，看一下哪个地方有问题（需要使用专业工具）

## 使用 JProfiler

如果一个项目出现 OOM 故障：
1. 通过 Debug，一行行分析代码
2. 通过内存快照分析工具 MAT 或 JProfiler，查看第几行代码出错

在 IDEA 中安装插件，并下载对应 OS 的安装包。

使用命令行 `-Xms1m -Xmx8m -XX:+HeapDumpOnOutOfMemoryError`，来产生 dump 文件。然后打开 JProfiler 进行分析。

`-Xms` 是设置化初始化内存大小
`-Xmx` 设置最大分配内存
`-XX:+PrintGCDetails`：打印 GC 垃圾回收信息
`-XX:+HeapDumpOnOutOfMemoryError`：OOM Dump

## JMM / Java memory model

Java 内存模型

作用：缓存一致性协议，用于定义数据读写的规则

JVM 有一个主内存，每个线程在使用数据的时候，要先去主内存拷贝一份数据到自己的线程工作内存区域中。只要是相互拷贝的，就有可能出现数据不一致的情况

JMM 定义了主内存/Main Memory 和线程工作内存之间的抽象关系。

线程之间的共享变量存储在 Main Memory 中，每个线程都有一个私有的本地内存/Local Memory

要解决共享对象可见性的问题，可以加 volatile 关键字或加锁。

JMM 是抽象概念，对八种指令的使用，制定了相应规则。

指令重排，原子性问题

---

本文的一些参考资料：

- [【狂神说Java】JVM快速入门篇](https://www.bilibili.com/video/BV1iJ411d7jS)
- [全套JVM视频教程，全网播放超百万的jvm教程（深入理解Java虚拟机）](https://www.bilibili.com/video/BV1DA411G7fR)
- [网易Java高级系列直播课（2月）](https://study.163.com/course/courseLearn.htm?courseId=1209696848#/learn/live?lessonId=1280291272&courseId=1209696848)























