这是极客时间的教程「深入拆解Java虚拟机」的笔记。

# Java 虚拟机是怎么运行 Java 字节码的？

为什么使用 JVM 来运行 Java 字节码？

- 可以轻松实现 Java 代码的跨平台执行
- JVM 提供了一个托管平台，提供内存管理、垃圾回收、编译时动态校验等功能
- 使用 JVM 能够让我们的编程工作更轻松、高效节省公司成本，提示社会化的整体快发效率，我们只关注和业务相关的程序逻辑的编写，其他业务无关但对于编程同样重要的事情交给 JVM 来处理

---

从虚拟机的视角，Java 代码首先会被编译为 class 文件，然后被加载到 Java 虚拟机中。加载后的 Java 类被存放于 **方法区（Method Area）** 中。

在实际运行时，虚拟机会执行  **方法区（Method Area）** 内的代码。

Java 虚拟机还会在内存中划分出 **堆（Heap）** 和 **栈（Stack）** 来存储运行时数据。

JVM 中的 Stack 可以细分为：

- 面向 Java 方法的 Java 方法栈
- 面向本地方法（native 方法）的本地方法栈
- 存放各个线程执行位置的程序计数器（PC寄存器/PC Register）

在运行的过程中，每当调用进入一个 Java 方法，JVM 就会在当前线程的 **Java 方法栈** 中生成一个 **栈帧（Stack Frame）** ，用来存放局部变量和字节码的操作数：

- 这个 *栈帧（Stack Frame）* 的大小是提前计算好的，而且 JVM 不要求 *栈帧（Stack Frame）* 在内存空间里连续分布
- 当退出当前执行的方法时，不管是正常返回还是异常返回，JVM 都会弹出当前线程的当前 *栈帧（Stack Frame）*，并将之舍弃

硬件视角来看，Java 字节码无法直接执行，因此 JVM 需要将字节码翻译成机器码。在 HotSpot 中，翻译过程有两种形式：
1. 解释执行：
    - 将逐条字节码翻译成机器码
    - 无需等待编译
2. 即使编译/Just-In-Time compilation/JIT：
    - 将一个方法中包含的所有字节码编译成机器码后再执行
    - 实际运行速度更快

> **interpreter** : A VM module which implements method calls by individually executing bytecodes. The interpreter has a limited set of highly stylized stack frame layouts and register usage patterns, which it uses for all method activations. The Hotspot VM generates its own interpreter at start-up time.

> **JIT compilers** : An on-line compiler which generates code for an application (or class library) during execution of the application itself. ("JIT" stands for "just in time".) A JIT compiler may create machine code shortly before the first invocation of a Java method. Hotspot compilers usually allow the interpreter ample time to "warm up" Java methods, by executing them thousands of times. This warm-up period allows a compiler to make better optimization decisions, because it can observe (after initial class loading) a more complete class hierarchy. The compiler can also inspect branch and type profile information gathered by the interpreter.

HotSpot 在翻译时默认采用混合模式，它会先解释执行字节码，而后将其中反复执行的热点代码，以方法为单位进行即时编译：

- 我们假设程序符合二八定律，20% 的代码占据了 80% 的计算资源
- 对于不常用的代码，就不用花费时间编译成机器码，直接采用解释执行即可
- 对于小部分热点代码，将其编译成机器码就可以提高运行时的速度

# Java 的基本类型

## JVM 中的 boolean 类型

boolean 类型在 JVM 中被映射为整数类型（int）： **`true` 被映射为 1，而 `false` 被映射为 0**

**Java 代码中的逻辑运算以及条件跳转，都是用整数相关的字节码来实现的**

## Java 的基本类型

除 boolean 类型之外，Java 还有另外 7 个基本类型：

- 整数类型 byte、short、char、int 和 long
- 浮点类型 float 和 double

它们拥有不同的值域，大的值域会包含小的值域，所以值域小的基本类型不需要强制转换就能转为值域大的基本类型。

需要注意：

- **所有基本类型的默认值在内存中均为 0**
- 无符号的类型只有 boolean 和 char
	- char 的取值范围则是 [0, 65535]
		- 正常情况 char 类型的值为非负数
		- 这种特性十分有用，比如说作为数组索引等
	- boolean 的取值是 0 或 1
- 浮点类型（float）在运算或比较时，需要考虑 +0.0F、-0.0F 以及 NaN 的情况：在向量化比较的时候需要考虑这个特性

## Java 基本类型的大小

Java 虚拟机每调用一个 Java 方法，便会创建一个栈帧。

> 这里只讨论供解释器使用的解释栈帧（interpreted frame）

栈帧有两个主要的组成部分：

- （广义的）局部变量区
	- 局部变量
	- 实例方法的 `this` 指针
	- 方法所接收的参数
- 字节码的操作数栈

在 Java 虚拟机规范中，**局部变量区等价于一个数组，并且可以用正整数来索引** ：

- 除了 long、double 需要用两个数组单元来存储之外，其他基本类型以及引用类型的值均占用一个数组单元
	- boolean、byte、char、short、int 及引用类型，在解释执行的方法栈帧中占用的空间大小是一致的
	- 在 32 位的 HotSpot 中，这些类型在栈上将占用 4 个字节；而在 64 位的 HotSpot 中，他们将占 8 个字节
- 这种情况仅存在于局部变量，而不会出现在存储于堆中的字段或者数组元素上
	- byte、char 以及 short 这三种类型的字段或者数组单元在堆上占用的空间分别为一字节、两字节，以及两字节
	- 也就是说，跟这些类型的值域相吻合
	- 因此，当我们将一个 int 类型的值，存储到这些类型的字段或数组时，相当于做了一次隐式的掩码操作
	- boolean 字段和 boolean 数组比较特殊：
		- 在 HotSpot 中，boolean 字段占用一字节，而 boolean 数组则直接用 byte 数组来实现
		- 为了保证堆中的 boolean 值是合法的，HotSpot 在存储时显式地进行掩码操作
		- 也就是说，只取最后一位的值存入 boolean 字段或数组中
- 产生区别的主要原因：变长数组在堆中不好控制，所以就选择浪费一些空间，以便访问时直接通过下标来计算地址

总结一下，就是 **虽然除了 long 和 double 外，其他基本类型与引用类型在栈上占用的空间大小一致，但它们在堆中占用的大小却不同**

> 在内存中，数据类型是不做区分的。Java程序想要把它解读成什么类型，它就是什么类型。

承上启下：

- 在将 boolean、byte、char 以及 short 的值存入字段或者数组单时，JVM 会进行掩码操作
- 在读取（加载）时，JVM 则会将其扩展为 int 类型

加载：JVM 的算数运算几乎全部依赖于操作数栈， **我们需要将堆中的 boolean、byte、char 以及 short 加载到操作数栈上，而后将栈上的值当成 int 类型来运算**

- 对于 boolean、char 这两个无符号类型来说，加载伴随着零扩展。举个例子，char 的大小为两个字节。在加载时 char 的值会被复制到 int 类型的低二字节，而高二字节则会用 0 来填充。
- 对于 byte、short 这两个类型来说，加载伴随着符号扩展。举个例子，short 的大小为两个字节。在加载时 short 的值同样会被复制到 int 类型的低二字节。如果该 short 值为非负数，即最高位为 0，那么该 int 类型的值的高二字节会用 0 来填充，否则用 1 来填充。

# JVM 如何加载 Java 类

## 两大类型和字节流

Java 语言的类型：

- 基本类型（primitive types）
	- 基本类型由 JVM 预先定义好
- 引用类型（reference types）
	- 泛型参数：会在编译过程中被擦除
	- 数组类：由 JVM 直接生成
	- 类：有对应的字节流
	- 接口：有对应的字节流

字节流的形式：

- 由 Java 编译器生成的 class 文件
- 在程序内部直接生成
- 从网络中获取（例如网页中内嵌的小程序 Java applet）字节流

不同形式的字节流都会被加载到 JVM 中，成为类或接口。

> 为了叙述方便，下面会用 *类* 来统称来统称它们。

无论是直接生成的数组类，还是加载的类，JVM 都需要对其进行链接和初始化。

JVM 将字节流转化为 Java 类的过程：

1. 加载
2. 链接
3. 初始化

## 加载

**加载是指查找字节流，并且据此创建类的过程**

**数组类没有字节流，所以由 JVM 直接生成，而其他类，JVM 需要借助 Class Loader（类加载器）来完成查找字节流的过程**

类加载器相当于建筑设计师，类就相当于房型。

而所有“建筑师”的鼻祖，就是 **启动类加载器（boot class loader）** ：

- 启动类加载器是由 C++ 实现的，没有对应的 Java 对象
- 在 Java 中只能用 null 来指代
- 相当于，建筑师的鼻祖不希望被打扰，所以没有留下联系方式

除了 boot class loader， **其他的类加载器都是 `java.lang.ClassLoader` 的子类** ，所以有对应的 Java 对象。

**加载需要借助类加载器，在 JVM 中，类加载器使用了双亲委派模型，即接收到加载请求时，会先将请求转发给父类加载器**

双亲委派机制：

- 每当一个类加载器接收到加载请求时，它会先将请求转发给父类加载器
- 在父类加载器没有找到所请求的类的情况下，该类加载器才会尝试去加载
- 相当于建筑师接单后，需要先给其师傅过目，师傅不接手，才能自己来做

> 在 Java 9 之前，启动类加载器负责加载最为基础、最为重要的类，比如存放在 JRE 的 lib 目录下 jar 包中的类（以及由虚拟机参数 `-Xbootclasspath` 指定的类）。

> 除了启动类加载器之外，另外两个重要的类加载器是扩展类加载器（extension class loader）和应用类加载器（application class loader），均由 Java 核心类库提供。

> 扩展类加载器的父类加载器是启动类加载器。它负责加载相对次要、但又通用的类，比如存放在 JRE 的 lib/ext 目录下 jar 包中的类（以及由系统变量 java.ext.dirs 指定的类）

> Java 9 引入了模块系统，[并且略微更改了上述的类加载器](https://docs.oracle.com/javase/9/migrate/toc.htm#JSMIG-GUID-A868D0B9-026F-4D46-B979-901834343F9E) —— 扩展类加载器被改名为平台类加载器（platform class loader）

> Java SE 中除了少数几个关键模块，比如说 java.base 是由启动类加载器加载之外，其他的模块均由平台类加载器所加载。

除了由 Java 核心类库提供的类加载器外，我们还可以加入自定义的类加载器，来实现特殊的加载方式。

举例来说，我们可以对 class 文件进行加密，加载时再利用自定义的类加载器对其解密。

**除了加载功能之外，类加载器还提供了命名空间的作用** ：

- 在 JVM 中，类的唯一性是由类加载器实例以及类的全名一同确定的
- 即便是同一串字节流，经由不同的类加载器加载，也会得到两个不同的类
- 在大型应用中，我们往往借助这一特性，来运行同一个类的不同版本

也就是说，如果你剽窃了另一个建筑师的设计作品，那么只要你标上自己的名字，这两个房型就是不同的

## 链接

**链接，是指将创建成的类合并至 JVM 中，使之能够执行的过程**

JVM 不会直接使用 `.class` 文件，类加载链接的目的就是在 JVM 中创建相应的类结构，并将其存储在元空间

链接的三个阶段：

1. 验证
	- 目的：确保被加载类能够满足 JVM 的约束条件
	- 相当于，建筑设计必须被审核通过，才能建造房子
2. 准备
	- 目的：为被加载类的静态字段分配内存
		- Java 代码中对静态字段的具体初始化，会在稍后的初始化阶段中进行
		- 过了这个阶段，算是盖好了毛坯房，虽然结构已经完整，但是在没有装修之前不能住人
	- 除了分配内存外，部分 JVM 还会在此阶段构造其他跟类层次相关的数据结构
		- 构造与该类相关联的方法表
		- 比如：用来实现虚方法的动态绑定的方法表
3. 解析
	- 在 class 文件被加载至 JVM  之前，这个类无法知道其他类及其方法、字段所对应的具体地址，甚至不知道自己方法、字段的地址
	- 因此，每当需要引用这些成员时，Java 编译器会生成一个符号引用
	- 在运行阶段，这个符号引用一般都能够无歧义地定位到具体目标上
		- 对于一个方法调用，编译器会生成一个包含目标方法所在类的名字、目标方法的名字、接收参数类型以及返回值类型的符号引用，来指代所要调用的方法
	- 解析阶段的目的：将这些符号引用 解析 成为实际引用
		- 如果符号引用指向一个未被加载的类，或者未被加载类的字段或方法，那么解析阶段将触发这个类的加载（但未必触发这个类的链接以及初始化）
		- 在盖房子的语境下，不管 A 的房子是否实际存在，都可以用这个说法指代 A 的房子
		- 实际引用就相当于 B 建筑师的电话，真的需要建房子的时候，就联系 B 建筑师去盖房子
	- 解析阶段为非必须的，JVM  规范并没有要求在链接过程中完成解析
		- 它仅规定了：如果某些字节码使用了符号引用，那么在执行这些字节码之前，需要完成对这些符号引用的解析

## 初始化

**初始化，则是为标记为常量值的字段赋值，以及执行 `< clinit >` 方法的过程**

- 在 Java 代码中，如果要初始化一个静态字段，我们可以在声明时直接赋值，也可以在静态代码块中对其赋值
- 如果直接赋值的静态字段被 final 所修饰，并且它的类型是基本类型或字符串时，那么该字段便会被 Java 编译器标记成常量值（Constant Value），其初始化直接由 JVM 完成
- 除此之外的直接赋值操作，以及所有静态代码块中的代码，则会被 Java 编译器置于同一方法中，并把它命名为 `< clinit >`

**JVM 会通过加锁来确保类的 `< clinit >` 方法仅被执行一次**

- 只有当初始化完成之后，类才正式成为可执行的状态
- 相当于，装修房子后，才能住人

那么，类的初始化何时会被触发呢？JVM 规范枚举了下述多种触发情况：

1. 当虚拟机启动时，初始化用户指定的主类；
2. 当遇到用以新建目标类实例的 new 指令时，初始化 new 指令的目标类；
3. 当遇到调用静态方法的指令时，初始化该静态方法所在的类；
4. 当遇到访问静态字段的指令时，初始化该静态字段所在的类；
5. 子类的初始化会触发父类的初始化；
6. 如果一个接口定义了 default 方法，那么直接实现或者间接实现该接口的类的初始化，会触发该接口的初始化；
7. 使用反射 API 对某个类进行反射调用时，初始化这个类；
8. 当初次调用 MethodHandle 实例时，初始化该 MethodHandle 指向的方法所在的类。

---

因为类的初始化仅会被执行一次，所以这个特性被用来实现 *单例的延迟初始化* ：

```java
public class Singleton {
    private Singleton() {}

    private static class LazyHolder {
        static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return LazyHolder.INSTANCE;
    }
}
```

只有当调用 `Singleton.getInstance` 时，程序才会访问 `LazyHolder.INSTANCE `，才会触发对 `LazyHolder` 的初始化（当遇到访问静态字段的指令时，初始化该静态字段所在的类），继而新建一个 `Singleton` 的实例。

由于类初始化是线程安全的，并且仅被执行一次，因此程序可以确保多线程环境下有且仅有一个 `Singleton` 实例。

# JVM是如何执行方法调用的？

## 重载与重写

重载与重写：

- 重载（Overload）指的是方法名相同而参数类型不相同的方法之间的关系

- 重写（Override）指的是方法名相同并且参数类型也相同的方法之间的关系

**重载的方法在编译过程中即可完成识别** ，具体到每一个方法调用，Java 编译器会根据所传入参数的声明类型（注意与实际类型区分）来选取重载方法

选取重载方法的过程共分为三个阶段：

1. 在不考虑对基本类型自动装拆箱（auto-boxing，auto-unboxing），以及可变长参数的情况下选取重载方法
2. 如果在第 1 个阶段中没有找到适配的方法，那么在允许自动装拆箱，但不允许可变长参数的情况下选取重载方法
3. 如果在第 2 个阶段中没有找到适配的方法，那么在允许自动装拆箱以及可变长参数的情况下选取重载方法

如果 Java 编译器在同一个阶段中找到了多个适配的方法，那么就根据形式参数类型的继承关系来选取：

```java
public class Test {

    public static void main(String[] args) {
        Test test = new Test();

        // null 既可以匹配 A 方法中声明为 Object 的形式参数，
        // 也可以匹配 B 方法中声明为 String 的形式参数
        // 由于 String 是 Object 的子类，因此 Java 编译器会认为 B 更贴切
        test.invoke(null, 1); // 调用 B
        test.invoke(null, 1, 2); // 调用 B

        // 只有手动绕开可变长参数的语法，才会调用 A
        test.invoke(null, new Object[]{1});
    }


    void invoke(Object obj, Object... args) {
        System.out.println("这是 A");
    }

    void invoke(String s, Object obj, Object... args) {
        System.out.println("这是 B");
    }
}
```

除了同一个类中的方法，重载也可以作用于这个类所继承而来的方法。也就是说，如果子类定义了与父类中非私有方法同名的方法，而且这两个方法的参数类型不同，那么在子类中，这两个方法同样构成了重载。

---

如果子类定义了与父类中非私有方法同名的方法，而且这两个方法的参数类型相同，那么这两个方法之间又是什么关系？

```java
class Test {

    public void foo1() {
        System.out.println("父类的非静态，非私有");
    }

    static void foo2() {
        System.out.println("父类的静态方法");
    }
}

// Test 的子类
class TestChild extends Test{

    // 子类的方法重写了父类中的方法
    @Override
    public void foo1() {
        System.out.println("子类的非静态，非私有");
    }

    // 子类中的方法隐藏了父类中的方法
    static void foo2() {
        System.out.println("子类的静态方法");
    }
}
```

## 静态绑定（static binding）和动态绑定（dynamic binding）

**JVM 识别方法的关键** ：

- 方法描述符（method descriptor）：
	- 方法的参数类型
	- 方法的返回类型
- 类名
- 方法名

在同一个类中，如果同时出现多个名字相同且描述符也相同的方法，那么 JVM 会在类的验证阶段就报错。

JVM 与 Java 语言不同，它并不限制「名字与参数类型相同，但返回类型不同的方法」出现在同一个类中。

也就是说，下面的代码在 Java 语言中不成立，但是在 JVM 中可以成立：

```java
// 名字都为 foo，参数类型都为 int，返回值不同
// 在 JVM 是可以成立的
class Test {
    boolean foo(int a) {
        System.out.println("返回值为 boolean");
        return a == 0;
    }

    int foo(int b) {
        System.out.println("返回值为 int");
        return b;
    }
}
```

对于调用这些方法的字节码来说，由于字节码所附带的方法描述符包含了返回类型，因此 JVM 能够准确地识别目标方法。

**JVM 中关于方法重写（Override）的判定同样基于方法描述符** 。如果子类定义了与父类中非私有、非静态方法同名的方法，那么只有当这两个方法的参数类型以及返回类型一致，JVM 才会判定为重写。

> Java 的重写（Override）与 JVM  中的重写（Override）并不一致

**对于 Java 语言中为 Override 而 JVM 中不是 Override 的情况，编译器会通过生成 Bridge Methods（[桥接方法](https://docs.oracle.com/javase/tutorial/java/generics/bridgeMethods.html)）来实现 Java中的重写语义**

---

**由于 Java 编译器在编译阶段已经区分了重载（Overload）的方法，所以可以认为 JVM 中不存在重载（Overload）这一概念** 。因此，在某些文章中重载也被称为静态绑定（static binding），或者编译时多态（compile-timepolymorphism，而重写则被称为动态绑定（dynamic binding）

这个说法在 JVM 语境下并非完全正确，因为某个类中的重载方法可能被它的子类所重写，因此 **Java 编译器会将所有对非私有实例方法的调用，编译为需要动态绑定的类型**

**Static Binding 和 Dynamic Binding** ：

- Static Binding（静态绑定）：在解析时便能够直接识别目标方法的情况
- Dynamic Binding（动态绑定）：需要在运行过程中根据调用者的动态类型来识别目标方法的情况

---

Java 字节码中与调用相关的指令共有五种：

1. invokestatic：用于调用静态方法
2. invokespecial：用于调用私有实例方法、构造器，以及使用 super 关键字调用父类的实例方法或构造器，和所实现接口的默认方法
3. invokevirtual：用于调用非私有实例方法
4. invokeinterface：用于调用接口方法
5. invokedynamic：用于调用动态方法

参考代码如下：

```java
import java.util.Random;

interface Customer {
    boolean isVip();
}

class Merchant {
    public double finalPrice(double originalPrice, Customer customer) {
        return originalPrice * 0.8d;
    }
}

class BlackMerchant extends Merchant {

    @Override
    public double finalPrice(double originalPrice, Customer customer) {

        if (customer.isVip()) { // invokeinterface
            return originalPrice * expensiveRate(); // invokestatic
        } else {
            return super.finalPrice(originalPrice, customer); // invokespecial
        }

    }

    // 熟客，使用更高的价格
    public static double expensiveRate() {
        return new Random() // invokespecial
                .nextDouble() // invokevirtual
                + 0.8d;
    }
}
```

**对于 `invokestatic` 以及 `invokespecial` 而言，JVM 能直接识别具体的目标方法** ，所以在 JVM  中，静态绑定包括：

- 用于调用静态方法的 `invokestatic` 指令
- 用于调用构造器、私有实例方法以及超类非私有实例方法的 `invokespecial` 指令

而 **对于 `invokevirtual` 以及 `invokeinterface` 而言，在绝大部分情况下，虚拟机需要在执行过程中，根据调用者的动态类型，来确定具体的目标方法**

**唯一的例外在于，如果虚拟机能够确定目标方法有且仅有一个，比如说目标方法被标记为 `final`，那么它可以不通过动态类型，直接确定目标方法**

## 调用指令的符号引用

在编译过程中，我们并不知道目标方法的具体内存地址。因此， **在 class 文件中，Java 编译器会暂时用符号引用来指代目标方法** ，这一符号引用包括：

- 目标方法所在的类或接口的名字
- 目标方法的方法名和方法描述符

符号引用存储在 class 文件的常量池之中（可以使用 `javap -v $类名.class` 来查看 Constant pool）。根据目标方法是否为接口方法，这些引用可分为

- 接口符号引用
- 非接口符号引用

**在执行调用指令前，JVM 会解析符号引用，并替换为实际引用**

对于非接口符号引用，假定该符号引用所指向的类为 C，则 JVM 会按照如下步骤进行查找：

1. 在 C 中查找符合名字及描述符的方法

2. 如果没有找到，在 C 的父类中继续搜索，直至 Object 类

3. 如果没有找到，在 C 所直接实现或间接实现的接口中搜索

	- 这一步搜索得到的目标方法必须是非私有、非静态的

	- 并且，如果目标方法在间接实现的接口中，则需满足 C 与该接口之间没有其他符合条件的目标方法

	- 如果有多个符合条件的目标方法，则任意返回其中一个

从这个解析算法可以看出，静态方法也可以通过子类来调用。此外，子类的静态方法会隐藏（注意与重写区分）父类中的同名、同描述符的静态方法。

对于接口符号引用，假定该符号引用所指向的接口为 I，则 JVM 会按照如下步骤进行查找：

1. 在 I 中查找符合名字及描述符的方法
2. 如果没有找到，在 Object 类中的公有实例方法中搜索。
3. 如果没有找到，则在 I 的超接口中搜索。这一步的搜索结果的要求与非接口符号引用步骤 3 的要求一致

经过上述的解析步骤之后，符号引用会被解析成实际引用：

- 对于可以静态绑定的方法调用而言，实际引用是一个指向目标方法的指针
- 对于需要动态绑定的方法调用而言，实际引用则是一个方法表的索引（辅助动态绑定的信息）

## Virtual Method 虚方法调用

**虚方法（virtual method ）调用包括 `invokevirtual` 指令和 `invokeinterface` 指令** ：

- 如果这两种虚方法调用指令所声明的目标方法，被标记为 `final`，那么 JVM 会采用 static binding（静态绑定）
- 否则（绝大部分情况），JVM 将采用 dynamic binding（动态绑定），在运行过程中根据调用者的动态类型，来决定具体的目标方法

> 静态绑定的「虚方法」，比静态绑定的「非虚方法」，更加耗时

JVM 中采取了一种用空间换取时间的策略实现 dynamic binding（动态绑定）。它为每个类生成一张方法表，用以快速定位目标方法

## Method Table 方法表

> 类加载的准备阶段，它除了为静态字段分配内存之外，还会构造与该类相关联的方法表

**JVM 中的 static binding（动态绑定），是通过 Method Table（方法表）这一数据结构来实现的** :

- Method Table 中每一个重写方法的索引值，与父类方法表中被重写的方法的索引值一致
- 在解析虚方法调用时，JVM 会记录下所声明的目标方法的索引值，并且在运行过程中根据这个索引值查找具体的目标方法。

下面将以 `invokevirtual` 所使用的虚方法表（virtual method table，vtable）为例介绍方法表的用法。`invokeinterface` 所使用的接口方法表（interface method table，itable）稍微复杂些，但原理是类似的。

**方法表本质上是一个数组，每个数组元素指向一个当前类及其祖先类中非私有的实例方法** ，这些方法可能是 *具体的、可执行的方法* ，也可能是 *没有相应字节码的抽象方法*

方法表满足两个特质：

1. 子类方法表中包含父类方法表中的所有方法
2. 子类方法在方法表中的索引值，与它所重写的父类方法的索引值相同

之前提到过：

> 方法调用指令中的符号引用会在执行之前解析成实际引用：
>
> 1. 对于静态绑定的方法调用而言，实际引用将指向具体的目标方法
>
> 2. 对于动态绑定的方法调用而言，实际引用则是方法表的索引值（实际上并不仅是索引值）

**动态绑定：在执行过程中，JVM 将获取调用者的实际类型，并在该实际类型的虚方法表（virtual method table，vtable）中，根据索引值获得目标方法**

例子：

```java
abstract class Passenger {

    // 出境
    abstract void leaveCountry();

    @Override
    public String toString() {
        return super.toString();
    }
}

class citizens extends Passenger {

    @Override
    void leaveCountry() {
        System.out.println("进入本国人通道");
    }
}

class nonCitizen extends Passenger {

    @Override
    void leaveCountry() {
        System.out.println("进入外国人通道");
    }

    void shopping() {
        System.out.println("免税店购物");
    }
}
```

上面的代码，可以获得方法表：

| Index | `Passenger` 的方法表 | 备注 |
| - | ------------------------- | -------------------------------- |
| 0 | `Passenger.toString()` | `toString` 方法的索引值需要与 `Object` 类中同名方法的索引值一致 |
| 1 | `Passenger.leaveCountry()` | 这是抽象方法，不可执行 |

| Index | `Citizens` 的方法表       |
| ----- | ------------------------- |
| 0     | `Passenger.toString()`    |
| 1     | `Citizens.leaveCountry()` |

| Index | `nonCitizen` 的方法表       |
| ----- | --------------------------- |
| 0     | `Passenger.toString()`      |
| 1     | `NonCitizen.leaveCountry()` |
| 2     | `NonCitizen.shopping()`     |

> 这里 JVM 的工作类似于出境的导航员，每当有人出境，就会询问是本国人还说外国人（获取动态类型），然后根据国籍，选择“外国人对应的指导手册”或“本国人对应的指导手册”（获取动态类型的方法表）。
>
> 指导手册的第 1 页可能会写应该到哪条通道办理出境手续（用 1 作为索引来查找方法表所对应的目标方法）。

使用了方法表的动态绑定与静态绑定相比，仅仅多出几个内存解引用操作：

- 访问栈上的调用者
- 读取调用者的动态类型
- 读取该类型的方法表
- 读取方法表中某个索引值所对应的目标方法

相对于创建并初始化 Java 栈帧来说，这几个内存解引用操作的开销简直可以忽略不计。那么我们是否可以认为虚方法调用对性能没有太大影响呢？

其实是不能的，上述优化的效果看上去十分美好，但实际上仅存在于解释执行中，或者即时编译代码的最坏情况中。这是因为即时编译还拥有另外两种性能更好的优化手段：

- 内联缓存（inlining cache）
- 方法内联（method inlining）

## Inlining Cache 内联缓存

在针对多态的优化手段中，我们通常会提及以下三个术语：

1. 单态（monomorphic）指的是仅有一种状态的情况
2. 多态（polymorphic）指的是有限数量种状态的情况。二态（bimorphic）是多态的其中一种
3. 超多态（megamorphic）指的是更多种状态的情况。

通常我们用一个具体数值来区分多态和超多态。在这个数值之下，我们称之为多态。否则，我们称之为超多态。

对于内联缓存来说，我们也有对应的单态内联缓存、多态内联缓存和超多态内联缓存：

- 单态内联缓存 [Monomorphic Inline Caching](https://en.wikipedia.org/wiki/Inline_caching)
	- 含义：只缓存了一种动态类型以及它所对应的目标方法
	- 实现：比较所缓存的动态类型，如果命中，则直接调用对应的目标方法
- 多态内联缓存 [Polymorphic Inline Caching](https://en.wikipedia.org/wiki/Inline_caching)
	- 含义：缓存了多个动态类型及其目标方法
	- 实现：将所缓存的动态类型与当前动态类型逐个进行比较，如果命中，则调用对应的目标方法
- 超多态内联缓存 [Megamorphic Inline Caching](https://en.wikipedia.org/wiki/Inline_caching#Megamorphic_inline_caching)
	- 放弃使用缓存，直接使用方法表来动态绑定

一般来说，我们会将更加热门的动态类型放在前面。在实践中，大部分的虚方法调用均是单态的，也就是只有一种动态类型。为了节省内存空间， **JVM 只采用单态内联缓存** 。

---

**JVM 中的即时编译器会使用 Inlining Cache（内联缓存）来加速动态绑定** ：

- JVM 所采用的单态 Inlining Cache，能 **缓存 Virtual Method（虚方法）调用中，调用者的动态类型，及该类型所对应的目标方法**
- 在之后的执行过程中， **如果碰到已缓存的动态类型，则直接调用已经缓存的该类型所对应的目标方法**
- **否则，JVM 将该 Inlining Cache 劣化为超多态内联缓存，也就是说，在今后的执行过程中直接使用方法表进行动态绑定**

> 这相当于导航员记住了上一个出境乘客的国籍和对应的通道，例如本国人，走了左边通道出境。那么下一个乘客想要出境的时候，导航员会先问是不是本国人，是的话就走左边通道。如果不是的话，只好拿出外国人的小册子，翻到第 1 页，再告知查询结果：右边

如上所述，当内联缓存没有命中的情况下，JVM 需要重新使用方法表进行动态绑定。

对于内联缓存中的内容，可以有两种选择：

1. 替换 Monomorphic Inline Caching（单态内联缓存）中的记录
	- 这种做法就好比 CPU 中的数据缓存，它对数据的局部性有要求，即在替换内联缓存之后的一段时间内，方法调用的调用者的动态类型应当保持一致，从而有效地利用内联缓存
	- 因此，在最坏情况下，我们用两种不同类型的调用者，轮流执行该方法调用，那么每次进行方法调用都将替换内联缓存。也就是说，只有写缓存的额外开销，而没有用缓存的性能提升
2. 另外一种选择则是劣化为 Megamorphic（超多态状态）
	- 这也是 JVM 的具体实现方式
	- 处于这种状态下的内联缓存，实际上放弃了优化的机会
	- 它将直接访问方法表，来动态绑定目标方法
	- 与替换内联缓存纪录的做法相比，它牺牲了优化的机会，但是节省了写缓存的额外开销。

> 劣化为 Megamorphic Inline Caching，相当于：
>
> 来了一队乘客，其中外国人和本国人依次隔开，那么在重复使用的单态内联缓存中，导航员需要反复记住上个出境的乘客，而且记住的信息在处理下一乘客时又会被替换掉。
>
> 因此，倒不如一直不记，以此来节省脑细胞

虽然内联缓存附带内联二字，但是它并没有内联目标方法。

任何「方法调用」除非被内联，否则都会有固定开销。这些开销来源于保存程序在该方法中的执行位置，以及新建、压入和弹出新方法所使用的栈帧。

对于极其简单的方法而言，比如说 getter/setter，这部分固定开销占据的 CPU 时间甚至超过了方法本身。此外，在即时编译中，方法内联不仅能够消除方法调用的固定开销，而且还增加了进一步优化的可能性

# JVM是如何处理异常的？

异常处理的两大组成要素是 *抛出异常* 和 *捕获异常* 。这两大要素共同实现程序控制流的非正常转移。

抛出异常可分为：

- 显式抛异常：
	- 主体是应用程序
	- 在程序中使用 `throw` 关键字，手动将异常实例抛出
- 隐式抛异常：
	- 主体则是 JVM 
	- JVM 在执行过程中，碰到无法继续执行的异常状态，自动抛出异常
	- 举例来说，JVM 在执行读取数组操作时，发现输入的索引值是负数，故而抛出数组索引越界异常 `ArrayIndexOutOfBoundsException`

捕获异常涉及三种 Block（代码块）：

- `try` 代码块：用来标记需要进行异常监控的代码
- `catch` 代码块：
	- 用来捕获在 `try` 代码块中触发的某种指定类型的异常
	- 还定义了针对该异常类型的异常处理器
	- 如果有多个 `catch` ，JVM 会从上往下匹配异常处理器，所以上面（前面） `catch` 代码块所捕获的异常类型不能覆盖下面（后面）的，否则编译器会报错
- `finally` 代码块：必须运行的代码，主要用于避免跳过某些关键的清理代码，如果关闭已打开的系统资源

所有异常都是 `Throwable` 类或者其子类的实例。

`Throwable` 有两大直接子：

- `Error` ：
	- 涵盖程序不应捕获的异常
	- 当程序触发 Error 时，它的执行状态已经无法恢复，需要中止线程甚至是中止虚拟机
- `Exception` ：
	- 涵盖程序可能需要捕获并且处理的异常
	- `Exception` 有一个特殊的子类 `RuntimeException`，表示“程序虽然无法继续执行，但是还能抢救一下”的情况
	- 前边提到的数组索引越界便是其中的一种

`RuntimeException` 和 `Error` 属于 unchecked exception（非检查异常）。

其他的 `Exception` 属于 checked exception（检查异常），在触发时需要显式捕获，或者在方法头用 `throws` 关键字声明（一定要注意， `RuntimeException` 不需要显示捕获）。

> 通常情况下，程序中自定义的异常应为检查异常，以便最大化利用 Java 编译器的编译时检查

在构造异常实例的时候，因为 JVM 会生成该异常的 stack trace（栈轨迹），所以需要逐一访问当前线程的 Java 栈帧、记录下各种调试信息，包括栈帧所指向方法的名字，方法所在的类名、文件名，以及在代码中的第几行触发该异常。

在生成 stack trace（栈轨迹）时，JVM 会忽略掉异常构造器以及 `Throwable.fillInStackTrace` （填充栈帧的方法），直接从新建异常位置开始算起。此外，JVM 还会忽略标记为不可见的 Java 方法栈帧。

> 既然异常实例的构造十分昂贵，我们是否可以缓存异常实例，在需要用到的时候直接抛出呢？
>
> 从语法角度上来看，这是允许的。然而，该异常对应的栈轨迹并非 `throw` 语句的位置，而是新建异常的位置。
>
> 因此，这种做法可能会误导开发人员，使其定位到错误的位置。这也是为什么在实践中，我们往往选择抛出新建异常实例的原因。

**在编译生成的 Bytecode（字节码）中，每个方法都附带一个 Exception Table（异常表）** ：

- Exception Table 中的每一个条目，代表一个异常处理器，并且由 from 指针、to 指针、target 指针以及所捕获的异常类型构成
	- from 指针和 to 指针标示了该异常处理器所监控的范围（例如 `try` 代码块所覆盖的范围）
	- target 指针指向异常处理器的起始位置，例如 `catch` 代码块的起始位置
- 这些指针的值是 BCI（Bytecode Instrumentation），用以定位 Bytecode

当程序触发异常时，JVM  将从上至下遍历异常表中的所有条目：

1. 当触发异常的 Bytecode 的 BCI 在某个 Exception Table 的监控范围内
	1. JVM  会判断「该 Bytecode 抛出的异常」和「其对应的 Exception Table 的条目中需要捕获的异常」是否匹配
	2. 如果匹配，JVM 会将控制流转移至该条目 target 指针指向的异常处理器之中
2. 如果遍历完所有 Exception Table 的条目，仍未匹配到异常处理器
	1. 那么它会弹出当前方法对应的 Java 栈帧
	2. 并且在 Caller（调用者）中重复上述操作
3. 在最坏情况下，JVM 需要遍历当前线程 Java 栈上所有方法的 Exception Table

`finally` 代码块的编译比较复杂。当前版本 Java 编译器的做法，是复制 `finally` 代码块的内容，分别放在 `try-catch` 代码块所有正常执行路径以及异常执行路径的出口中。

可以理解为每个可能的位置都放一个 `finally` 代码块，以此保证 `finally` 必然被执行。

Java 代码中的 `catch` 代码块和 `finally` 代码块都有对应的 Exception Table：

- 假设 `catch` 代码块捕获的异常 A 触发了另一个新的异常 B
- 那么 `finally` 会捕获并且重新抛出新的 B 异常
- 也就是说，原本的 A 异常会被忽略

Java 7 引入了 Supressed 异常来解决“一个异常触发另一个异常后，初始的异常会被忽略”的问题。这个新特性允许开发人员将一个异常附于另一个异常之上。因此，抛出的异常可以附带多个异常的信息。

然而，Java 层面的 `finally` 代码块缺少指向所捕获异常的引用，所以这个新特性使用起来非常繁琐。

为此，Java 7 专门构造了一个名为 `try-with-resources` 的语法糖，在字节码层面自动使用 Supressed 异常。当然，该语法糖的主要目的并不是使用 Supressed 异常，而是精简资源打开关闭的用法。

除了 `try-with-resources` 语法糖之外，Java 7 还支持在同一 `catch` 代码块中捕获多种异常：`catch (SomeException | OtherException e) {}` 。实际实现非常简单，生成多个异常表条目即可。

# Java 对象的内存布局

参考资料：

- [Memory Layout of Objects in Java](https://www.baeldung.com/java-memory-layout)
- [HotSpot Glossary of Terms](https://openjdk.java.net/groups/hotspot/docs/HotSpotGlossary.html)
- [JAVA对象布局之对象头(Object Header)](https://segmentfault.com/a/1190000037643624)
- [Java Objects Inside Out](https://shipilev.net/jvm/objects-inside-out/#_field_packing)
- [JVM之压缩指针（CompressedOops）](https://juejin.cn/post/6844903768077647880)
- [码农会锁，synchronized 对象头结构(mark-word、Klass Pointer)、指针压缩、锁竞争，源码解毒、深度分析！](https://www.cnblogs.com/xiaofuge/p/13895226.html)

## Java 对象基础

JVM 构造对象的方式

- `new` 语句：通过调用构造器来初始化实例字段
- 反射机制：通过调用构造器来初始化实例字段
- `Object.clone` ：直接复制已有的数据，来初始化新建对象的实例字段
- 反序列化：直接复制已有的数据，来初始化新建对象的实例字段
- `Unsafe.allocateInstance` ：初始化实例字段

---

`new` 语句通过调用构造器来初始化实例字段，该语句编译而成的字节码包含：

- 用来请求内存的 `new` 指令
- 用来调用 constructor（构造器）的 `invokespecial` 指令

如下代码：

```java
public class Foo {
    public static void main(String[] args) {
        Foo foo = new Foo();
    }
}
```

先执行 `javac Foo.java` 编译，然后 `javap -c Foo` 对代码进行反汇编，得到：

```
Compiled from "Foo.java"
public class Foo {
  public Foo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class Foo
       3: dup
       4: invokespecial #3                  // Method "<init>":()V
       7: astore_1
       8: return
}
```

其中 `0: new` 就是请求内存空间，`4: invokespecial` 就是调用 constructor。

从反编译的字节码中也可以看出，如果一个类没有定义任何 constructor 的话， Java 编译器会自动添加一个无参数的 constructor，如上面字节码中的  `public Foo();`

然后，子类的 constructor 需要调用父类的 constructor：

- 如果父类存在无参数构造器的话，该调用可以是隐式的：
	- 隐式：Java 编译器会自动添加对父类构造器的调用
- 如果父类没有无参数构造器，那么子类的构造器则需要显式地调用父类带参数的构造器。显式调用又可分为两种：
	- 直接使用 `super` 关键字调用父类构造器
	- 使用 `this` 关键字调用同一个类中的其他构造器
- 只要调用了父类构造器，无论是直接的显式调用，还是间接的显式调用，都会成为构造器的第一条语句：
	- 目的：以便优先初始化继承而来的父类字段
	- 提醒：不过这可以通过调用其他生成参数的方法，或者字节码注入来绕开

总而言之，当我们调用任意一个 constructor 时，它都直接或者间接调用父类的 constructor，直至 `Object` 类。

这些 constructor 的调用者皆为同一对象，也就是通过 `new` 指令新建而来的对象，该对象的内存涵盖了所有父类中的实例字段（新建的实例会初始化相应的父类字段）。

也就是说，虽然子类无法访问父类的私有实例字段，或者子类的实例字段隐藏了父类的同名实例字段，但是子类的实例还是会为这些父类实例字段分配内存。

## 对象的内存布局：字段在内存中的具体分布

Ordinary Object Pointers (OOPs)：

- JVM uses a data structure called Ordinary Object Pointers ([OOPs](https://github.com/openjdk/jdk15/tree/master/src/hotspot/share/oops)) to represent pointers to objects. Specifically, pointers into the GC-managed heap
- Implemented as a native machine address, not a handle
- OOPs may be directly manipulated by compiled or interpreted Java code, because the GC knows about the liveness and location of oops within such code. (See GC map.)
- OOPs can also be directly manipulated by short spans of C/C++ code, but must be kept by such code within handles across every safepoint.

Object Header（对象头）：

- 每个 Java 对象都有一个 Object Header，同时 every OOP (Ordinary Object Pointer) points to an Object Header
- Object Header is a common structure at the beginning of every GC-managed heap object
- Object Header includes fundamental information about the heap object's layout, type, GC state, synchronization state, and identity hash code
- In arrays it is immediately followed by a length field

> Note that both Java Objects and VM-internal Objects have a common Object Header format

---

[HotSpot 虚拟机中，对象在内存中存储的布局可以分为三块区域](https://www.cnblogs.com/xiaofuge/p/13895226.html)：

- 对象头（Object Header）
- 实例数据（Instance Data）
- 对齐填充（Padding）

Object Header consists of two words （对象头由两部分组成）：

- Mark Word（标记字段）：
	- The first word of every object header
	- 用以存储 JVM 有关该对象的运行数据（如哈希码、GC 信息以及锁信息等）
	- Usually a set of bitfields including synchronization state and identity hash code. May also be a pointer (with characteristic low bit encoding) to synchronization related information. During GC, may contain GC state bits.
- Klass Pointer（类型指针）：
	- The second word of every Object Header
	- Points to another object (a metaobject) which describes the layout and behavior of the original object
	- 也就是指向该对象的类
	- For Java objects, the "klass" contains a C++ style "vtable"

## 压缩指针 CompressedOops

在 64 位的 JVM 中，Object Header（对象头）的Mark Word（标记字段）占 64 位，而 Klass Pointer（类型指针）又占了 64 位。

也就是说，每一个 Java 对象在内存中的额外开销就是 16 个字节。以 Integer 类为例，它仅有一个 int 类型的私有字段，占 4 个字节。因此，每一个 Integer 对象的额外内存开销至少是 400%。这也是为什么 Java 要引入基本类型的原因之一。

为了尽量较少对象的内存使用量，64 位 JVM 引入了 [CompressedOops](https://wiki.openjdk.java.net/display/HotSpot/CompressedOops)（压缩指针）的概念：

- 对应虚拟机选项 `-XX:+UseCompressedOops` ，默认是开启的
- 功能：将 GC-managed Heap（堆）中原本 64 位的 Java 对象指针（OOPs）压缩成 32 位
- 附带效果 1：对象头中的 Klass Pointer（类型指针）也会被压缩成 32 位，使得 Object Header 的大小从 16 字节降至 12 字节
- 附带效果 2：还可以作用于引用类型的字段，包括引用类型数组

---

CompressedOops 的原理：

打个比方，路上停着的全是房车，而且每辆房车恰好占据两个停车位。现在，我们按照顺序给它们编号。也就是说，停在 0 号和 1 号停车位上的叫 0 号车，停在 2 号和 3 号停车位上的叫 1 号车，依次类推。

原本的内存寻址用的是车位号：比如说我有一个值为 6 的指针，代表第 6 个车位（1 辆房车会占据 2 个车位），那么沿着这个指针可以找到 3 号房车。

现在我们规定指针里存的值是「房车号」，比如 「房车号 3」指代 3 号房车。当需要查找 3 号房车时，我便可以将该指针的值（即「房车号」）乘以 2 等于 6（表示根据 3 号“房车号”，找到 6 号“停车位”），再沿着 6 号“停车位”找到 3 号房车。

也就是说，因为 1 辆房车会占据 2 个车位，所以干脆将“1 辆房车占据的车位”，当做 1 个「房车号」。即，「2 个停车位」等于「1 个房车号」。

所以，32 位 CompressedOops（压缩指针）最多可以标记 2 的 32 次方辆车，对应着 2 的 33 次方个车位。

当然，房车也有大小之分。大房车占据的车位可能是三个甚至是更多。不过这并不会影响我们的寻址算法：我们只需跳过部分车号，便可以保持 `原本车号 * 2` 的寻址系统

上述模型有一个前提——每辆车都从偶数号车位停起，这个概念就是内存对齐

## Alignment & Padding

By default, the JVM adds enough padding to the object to make its size a multiple of 8

内存对齐：

- 虚拟机选项 `-XX:ObjectAlignmentInBytes` ，默认值为 8
- JVM 堆中对象的起始地址需要对齐至 8 的倍数
	- 如果想修改为至少 16 的倍数，可以使用： `-XX:ObjectAlignmentInBytes=16`

如果一个对象用不到 8N 个 Bytes（字节），那么空白的那部分空间就浪费掉了， **浪费掉的空间就是对象间的 padding（填充）**

> 默认情况下，JVM 中的 32 位 CompressedOops（压缩指针）可以寻址到 2 的 35 次方个字节，也就是 32GB 的地址空间（超过 32GB 则会关闭压缩指针）。
>
> 在对 CompressedOops 解引用时，我们需要将其左移 3 位，再加上一个固定 offset（偏移量可以为 0），便可以得到能够寻址32GB 地址空间的伪 64 位指针了。
>
> 此外，我们可以通过配置刚刚提到的内存对齐选项（`-XX:ObjectAlignmentInBytes`）来进一步提升寻址范围。但是，这同时也可能增加对象间填充，导致压缩指针没有达到原本节省空间的效果。
>
> 举例来说，如果规定每辆车都需要从偶数车位号停起，那么对于占据两个车位的小房车来说刚刚好，而对于需要三个车位的大房车来说，也仅是浪费一个车位。
>
> 但是如果规定需要从 4 的倍数号车位停起，那么小房车则会浪费两个车位，而大房车至多可能浪费三个车位。

**即使关闭了压缩指针，JVM 还是会进行内存对齐**。

此外， **内存对齐不仅存在于对象与对象之间（Object Alignment），也存在于对象中的字段之间（Field Alignments）**

比如说，JVM 要求 long 字段、double 字段，以及非压缩指针状态下的引用字段地址为 8 的倍数。

字段内存对齐的其中一个原因，是让字段只出现在同一 CPU 的缓存行中。如果字段不是对齐的，那么就有可能出现跨缓存行的字段。

也就是说，该字段的读取可能需要替换两个缓存行，而该字段的存储也会同时污染两个缓存行。这两种情况对程序的执行效率而言都是不利的。

## Field Packing 字段重排列：使字段内存对齐

字段重排列：

- JVM 重新分配字段的先后顺序，以达到内存对齐的目的
- When a class has multiple fields, the JVM may distribute those fields in such a way as to minimize padding waste

JVM 中有三种排列方法（对应 JVM 选项 `-XX:FieldsAllocationStyle` ，默认值为 1），但都会遵循如下两个规则：

1. 如果一个字段占据 C 个字节，那么该字段的 offset（偏移量）需要对齐至 NC（这里的 offset 指的是字段地址与对象的起始地址差值）
	- 以 long 类为例，它仅有一个 long 类型的实例字段
	- 在使用了压缩指针的 64 位虚拟机中，尽管对象头的大小为 12 个字节，该 long 类型字段的偏移量也只能是 16
	- 中间空着的 4 个字节便会被浪费掉
2. 子类所继承字段的偏移量，需要与父类对应字段的偏移量保持一致
	- 在具体实现中，JVM 还会对齐子类字段的起始位置
	- 对于使用了压缩指针的 64 位虚拟机，子类第一个字段需要对齐至 4N
	- 对于关闭了压缩指针的 64 位虚拟机，子类第一个字段则需要对齐至 8N

## 虚共享 False Sharing 和 @Contended 注解

The `@Contended` annotation is a hint for the JVM to isolate the annotated fields to avoid [false sharing](https://alidg.me/blog/2020/4/24/thread-local-random#false-sharing) ：

- `jdk.internal.vm.annotation.Contended` or `sun.misc.Contended` on Java 8
- `@Contended` 用于解决对象字段之间的虚共享问题
- `@Contended` 也会影响到字段的排列

False Sharing（虚共享）：

- 假设两个线程分别访问同一对象中不同的 `volatile` 字段，逻辑上它们并没有共享内容，因此不需要同步
- 然而，如果这两个字段恰好在同一个缓存行中，那么对这些字段的写操作会导致缓存行的写回，也就造成了实质上的共享

The `@Contended` annotation adds some paddings around each annotated field to isolate each field on its own cache line. 

Consequently, this will impact the memory layout.

> JVM 会让不同的 `@Contended` 字段处于独立的缓存行中，因此会有大量的空间被浪费掉

# GC 垃圾回收

## 垃圾回收的三种方式

当标记完所有的存活对象时，就可以进行死亡对象的回收工作。主流的基础回收方式可分为三种：

- Sweep（清除）:
	- 把死亡对象所占据的内存标记为空闲内存，并记录在一个 Free List（空闲列表）之中
		- Free List：A storage management technique in which unused parts of the Java object heap are chained one to the next, rather than having all of the unused part of the heap in a single block
	- 当需要新建 Object 时，内存管理模块便会从该空闲列表中寻找空闲内存，并划分给新建的 Object
	- 缺点 1：造成内存碎片——Heap 中的对象必须是连续分布的，因此可能出现总空闲内存足够，但是无法分配的极端情况
	- 缺点 2：分配效率较低——如果是一块连续的内存空间，就可以通过 pointer bumping（指针加法）来做分配。而对于空闲列表，JVM 需要逐个访问列表中的项，来查找能够放入新建 Object 的空闲内存
- Compact（压缩）：
	- 把存活的 Object 聚集到内存区域的起始位置，从而留下一段连续的内存空间。这种做法能够解决内存碎片化的问题
	- 代价是压缩算法的性能开销较大
- Copy（复制）：
	- 把内存区域分为两等分，分别用两个指针 from 和 to 来维护，并且只是用 from 指针指向的内存区域来分配内存
	- 当发生垃圾回收时，便把存活的 Object 复制到 to 指针指向的内存区域中，并且交换 from 指针和 to 指针的内容
	- Copy 这种回收方式同样能够解决内存碎片化的问题，但是它的缺点也极其明显——堆空间的使用效率极其低下

## JVM 中 Heap 的划分：Old Generation & Young Generation

因为大部分的 Java 对象只存活一小段时间，而存活下来的小部分 Java 对象则会存活很长一段时间，所以 JVM  采用「分代回收」

堆空间分为两代：

- 新生代（Young Generation）：
	- A region of the Java object heap that holds recently-allocated objects.
	- 用来存储新建的对象
	- 因为大部分的 Java 对象只存活一小段时间，所以可以频繁地采用耗时较短的垃圾回收算法，让大部分的垃圾都能够在新生代（Young Generation）被回收掉
- 老年代（Old Generation）：
	- A region of the Java object heap that holds object that have remained referenced for a while
	- 当对象存活时间够长时，就会被移动到老年代（Old Generation）
	- 因为大部分的垃圾已经在新生代（Young Generation）中被回收了，而在老年代（Old Generation）中的对象有大概率会继续存活
	- 当真正触发针对老年代（Old Generation）的回收时，则代表这个假设出错了，或者堆的空间已经耗尽了
	- 这时，JVM 往往需要做一次全堆扫描，耗时也将不计成本（现代的垃圾回收器都在并发收集的道路上发展，来避免这种全堆扫描的情况。）

新生代（Young Generation）：

- Eden 区：A part of the Java object heap where object can be created efficiently
- Survivor 区：分为 Survivor 0（S0）和 Survivor 1（S1），会动态变为 from 和 to：
	- Survivor 区 from（大小和 to 相同，用于存放 Object）
	- Survivor 区 to（大小和 from 相同，但是 to 是空的）

> **survivor space** : A region of the Java object heap used to hold objects. There are usually a pair of survivor spaces, and collection of one is achieved by copying the referenced objects in one survivor space to the other survivor space

默认情况下，JVM 采取的是一种动态分配的策略（参数 `-XX:+UsePSAdaptiveSurvivorSizePolicy` ），根据生成对象的速率，以及 Survivor 区的使用情况动态调整 Eden 区和 Survivor 区的比例：

- 可以通过参数 `-XX:SurvivorRatio` 来固定 Eden 区和 Survivor 区的比例
- 因为 Survivor 区的 to 一直为空，所以该比例越低浪费的堆空间将越高（Survivor 区比例越高，浪费的空间越多）

## TLAB

当我们调用 `new` 指令时，它会在 Eden 区中划出一块作为存储对象的内存。

由于堆空间是线程共享的，因此直接在这里边划空间是需要进行同步的。否则，将有可能出现两个对象共用一段内存的事故。相当于两个司机（线程）同时将车停入同一个停车位，而发生事故。

JVM 的解决方法是为每个司机预先申请多个停车位，并且只允许该司机停在自己的停车位上。

如果一个司机的停车位用完了，可以再申请多个停车位。

这项技术被称之为 `TLAB` （Thread Local Allocation Buffer，虚拟机参数 `-XX:+UseTLAB` ，默认开启）。

> **TLAB** : Thread-local allocation buffer. Used to allocate heap space quickly without synchronization. Compiled code has a "fast path" of a few instructions which tries to bump a high-water mark in the current thread's TLAB, successfully allocating an object if the bumped mark falls before a TLAB-specific limit address.

具体来说，每个线程可以向 JVM 申请一段连续的内存，比如 2048 字节，作为线程私有的TLAB。这个操作需要加锁，线程需要维护两个指针：

- 一个指向 TLAB 中空余内存的起始位置
- 一个则指向 TLAB 末尾
- （实际上可能有更多的指针，但这只有这两个比较重要）

接下来的 `new` 指令，就可以直接通过 bump the pointer（指针加法）来实现，也就是把「指向空余内存位置的指针」加上「所请求的字节数」。

> bump the pointer 实际上是 bump up the pointer，意思是“提高指针”，这里翻译为“指针加法”。
>
> 题外话：软件更新版本号的时候，也可以说 bump the version number

如果加法后空余内存指针的值仍小于或等于指向末尾的指针，则代表分配成功。

否则，TLAB 已没有足够的空间来满足本次新建操作。这个时候，便需要当前线程重新申请新的 TLAB。

## Minor GC

**当 Eden 区的空间耗尽了，就会触发一次 Minor GC，用于收集新生代（Young Generation）的垃圾。存活下来的对象，则会被送到 Survivor 区。**

新生代（Young Generation）共有两个 Survivor 区，我们分别用 from 和 to 来指代。其中 to 指向的 Survivior 区是空的。

当发生 Minor GC 时， <u>Eden 区</u> 和 <u>from 指向的 Survivor 区</u> 中存活的对象，会被复制到 <u>to 指向的Survivor 区</u> 中，然后 **交换 from 和 to 指针** ，以保证下一次 Minor GC 时，to 指向的 Survivor 区还是空的。

JVM 会记录 <u>Survivor 区</u> 中的对象一共被来回复制了几次：

- 如果一个对象被复制次数超过一定数值的 age 时（默认次数为 15，参数 `-XX:+MaxTenuringThreshold` ），那么该对象将被晋升（promote）至老年代（Old Generation）
- 如果 <u>单个Survivor 区</u>（S0 或 S1 中已经存储了 Object 的区域）已经被占用了一定的比例（参数 `-XX:TargetSurvivorRatio` ，默认值是 50，表示 50%），那么：age 从小到大的对象占据的空间，如果大于单个 Survivor 区域的一半（默认是一半），就把等于或大于该 age 的对象，晋升至老年代（Old Generation）

也就是说， **当发生 Minor GC 时，会使用「标记 - 复制算法」，将 Survivor 区中老的存活对象晋升到老年代（Old Generation），然后将剩下的存活对象和 Eden 区的存活对象复制到另一个 Survivor 区中**

理想情况下，Eden 区中的对象基本都死亡了，那么需要复制的数据将非常少，因此采用这种「标记 - 复制算法」的效果极好。

Minor GC 的另外一个好处是不用对整个堆进行垃圾回收。但是，它却有一个问题，那就是老年代（Old Generation）的对象可能引用新生代（Young Generation）的对象。

也就是说，在标记存活对象的时候，我们需要扫描老年代（Old Generation）中的对象。如果该对象拥有对新生代（Young Generation）对象的引用，那么这个引用也会被作为 GC Roots。

这样一来，好像又做了一次全堆扫描？解决方案就是 Card Table

## Card Table

因为 Minor GC 只针对新生代（Young Generation）进行垃圾回收，所以在枚举 GC Roots 的时候，“如果老年代（Old Generation）中的对象，拥有新生代（Young Generation）中的对象的引用”，就可能需要扫描整个老年代（Old Generation）。

**为了避免扫描整个老年代（Old Generation），HotSpot 引入了名为 Card Table 的技术，可以大致地标出可能存在“老年代（Old Generation）到新生代（Young Generation）引用”的内存区域**

Card Table（卡表）：

- A kind of remembered set that records where oops have changed in a generation
	- Remembered Set：A data structure that records pointers between generations
- 整个 Heap 会被划分为一个个大小为 512 字节的 Card（卡），Card Table 会储存每张 Card 的标识
- 每张 Card 的标识位，表示“对应的 Card 是否可能存有指向新生代（Young Generation）对象的引用”，如果可能存在，那么我们就认为这张卡是 dirty（脏的）

在 Minor GC 的时候，就不需要扫描整个老年代（Old Generation），而是 **在 Card Table 中找到 Dirty Card(s)，并将 Dirty Card 中的 Object（对象）加入到 Minor GC 的 GC Roots 中**

**After all Dirty Cards are scanned, the JVM will clear the identification bits of all Dirty Cards** （扫描完所有脏卡后，就将所有脏卡的标识位清零）

---

因为 Minor GC 需要对「存活的对象」进行「复制」操作。而「复制」操作需要更新「指向该对象的引用」。因此，在更新「引用」的同时，我们又会设置「引用所在的卡的标识位」。这个时候，我们可以确保 Dirty Card（脏卡）中必定包含「指向新生代（Young Generation）对象的引用」。

在 Minor GC 之前，我们并不能确保 Dirty Card 中是否包含「指向新生代（Young Generation）对象的引用」。其原因和「如何设置卡的标识位」有关：

首先，要想要保证每个可能有「指向新生代（Young Generation）对象引用」的卡都被标记为 Dirty Card，那么 JVM 就需要截获每个引用型实例变量的写操作，并作出对应的写标识位操作

这个操作在解释执行器中比较容易实现。但是在即时编译器生成的机器码中，则需要插入额外的逻辑。这也就是所谓的写屏障（write barrier，注意不要和 volatile 字段的写屏障混淆）

> **write barrier** : Code that is executed on every oop store. For example, to maintain a remembered set.

写屏障需要尽可能地保持简洁。这是因为我们并不希望在每条引用型实例变量的写指令后跟着一大串注入的指令。

因此，写屏障并不会判断更新后的引用是否指向新生代（Young Generation）中的对象，而是宁可错杀，不可放过，一律当成可能指向新生代（Young Generation）对象的引用。

这么一来，写屏障便可精简为下面的伪代码 [1]。这里右移 9 位相当于除以 512，JVM 便是通过这种方式来从地址映射到卡表中的索引的。最终，这段代码会被编译成一条移位指令和一条存储指令。

```
CARD_TABLE [this address >> 9] = DIRTY;
```

虽然写屏障不可避免地带来一些开销，但是它能够加大 Minor GC 的吞吐率（ 应用运行时间 /(应用运行时间 + 垃圾回收时间) ）。总的来说还是值得的。不过，在高并发环境下，写屏障又带来了虚共享（false sharing）问题。

> 在介绍对象内存布局中曾提到虚共享问题，讲的是几个 volatile 字段出现在同一缓存行里造成的虚共享。这里的虚共享则是卡表中不同卡的标识位之间的虚共享问题。

在 HotSpot 中，卡表是通过 byte 数组来实现的。对于一个 64 字节的缓存行来说，如果用它来加载部分卡表，那么它将对应 64 张卡，也就是 32KB 的内存。

如果同时有两个 Java 线程，在这 32KB 内存中进行引用更新操作，那么也将造成存储卡表的同一部分的缓存行的写回、无效化或者同步操作，因而间接影响程序性能。

为此，HotSpot 引入了一个新的参数 -XX:+UseCondCardMark，来尽量减少写卡表的操作。其伪代码如下所示：

```
if ( CARD_TABLE [this address >> 9] != DIRTY )
	CARD_TABLE [this address >> 9] = DIRTY;
```

## 引用计数法与可达性分析

> JVM 中的垃圾回收器采用可达性分析来探索所有存活的对象。它从一系列 GC Roots 出发，边标记边探索所有被引用的对象

垃圾回收：将已经分配出去的，但却不再使用的内存回收回来，以便能够再次分配

在 JVM 的语境下， **垃圾指的是死亡的对象所占据的堆空间** 。这里便涉及了一个关键的问题：如何辨别一个对象是存是亡？

**引用计数法（Reference Counting）** ：

- 为每个对象添加一个引用计数器，用来统计指向该对象的引用个数
- 一旦某个对象的引用计数器为 0，则说明该对象已经死亡，便可以被回收了

Reference Counting 具体实现：

- 如果有一个引用，被赋值为某一对象，那么将该对象的引用计数器 +1
- 如果一个指向某一对象的引用，被赋值为其他值，那么将该对象的引用计数器 -1
- 也就是说，JVM 需要截获所有的引用更新操作，并且相应地增减目标对象的引用计数器

除了需要额外的空间来存储计数器，以及繁琐的更新操作，引用计数法还有一个重大的漏洞，那便是无法处理循环引用对象。

举个例子，假设对象 a 与 b 相互引用，除此之外没有其他引用指向 a 或者 b。在这种情况下，a 和 b 实际上已经死了，但由于它们的引用计数器皆不为 0，在引用计数法的心中，这两个对象还活着。

因此，这些循环引用对象所占据的空间将不可回收，从而造成了 **内存泄露**

目前 JVM 的主流垃圾回收器采取的是 **可达性分析算法** ，这个算法的实质是：

- 将一系列 GCRoots 作为初始的 Live Set（存活对象合集）
- 然后从该 Live Set 出发，探索所有能够被该 Live Set 引用到的对象，并将其加入到该集合中
	- 这个过程称之为 mark（标记）
- 最终，未被探索到的对象便是死亡的，是可以回收的。

GC Roots 包括（但不限于）如下几种：

- Java 方法栈桢中的局部变量
- 已加载类的静态变量
- JNI handles
- 已启动且未停止的 Java 线程

> **GC Root (Garbage Collection Root)** : A pointer into the Java object heap from outside the heap. These come up, e.g., from static fields of classes, local references in activation frames, etc.

> **JNI** : The Java Native Interface - a specification and API for how Java code can call out to native C code, and how native C code can call into the Java VM

> **handle** : A memory word containing an oop. The word is known to the GC, as a root reference. C/C++ code generally refers to oops indirectly via handles, to enable the GC to find and manage its root set more easily. Whenever C/C++ code blocks in a safepoint, the GC may change any oop stored in a handle. Handles are either 'local' (thread-specific, subject to a stack discipline though not necessarily on the thread stack) or global (long-lived and explicitly deallocated). There are a number of handle implementations throughout the VM, and the GC knows about them all.

**可达性分析可以解决引用计数法所不能解决的循环引用问题** 。举例来说，即便对象 a 和 b 相互引用，只要从 GC Roots 出发无法到达 a 或者 b，那么可达性分析便不会将它们加入存活对象合集之中。

## Stop-the-world 以及安全点

>为了防止在标记过程中堆栈的状态发生改变，JVM 采取安全点机制来实现 Stop-the-world 操作，暂停其他非垃圾回收线程

虽然可达性分析的算法本身很简明，但是在实践中还是有不少其他问题需要解决的。

比如说，在多线程环境下，其他线程可能会更新已经访问过的对象中的引用，从而造成误报（将引用设置为 null）或者漏报（将引用设置为未被访问过的对象）。

误报并没有什么伤害，Java 虚拟机至多损失了部分垃圾回收的机会。漏报则比较麻烦，因为垃圾回收器可能回收事实上仍被引用的对象内存。一旦从原引用访问已经被回收了的对象，则很有可能会直接导致 Java 虚拟机崩溃。

怎么解决这个问题呢？在 JVM 里，传统的垃圾回收算法采用的是一种简单粗暴的方式，那便是 Stop-the-world，停止其他非垃圾回收线程的工作，直到完成垃圾回收。

这也就造成了垃圾回收所谓的暂停时间（GC pause）

JVM  中的 Stop-the-world 是通过 safepoint（安全点）机制来实现的。当 Java 虚拟机收到 Stop-the-world 请求，它便会等待所有的线程都到达安全点，才允许请求 Stop-the-world 的线程进行独占的工作。

> **Safepoint** : A point during program execution at which all GC roots are known and all heap object contents are consistent. From a global point of view, all threads must block at a safepoint before the GC can run. (As a special case, threads running JNI code can continue to run, because they use only handles. During a safepoint they must block instead of loading the contents of the handle.) From a local point of view, a safepoint is a distinguished point in a block of code where the executing thread may block for the GC. Most call sites qualify as safepoints. There are strong invariants which hold true at every safepoint, which may be disregarded at non-safepoints. Both compiled Java code and C/C++ code be optimized between safepoints, but less so across safepoints. The JIT compiler emits a GC map at each safepoint. C/C++ code in the VM uses stylized macro-based conventions (e.g., TRAPS) to mark potential safepoints.

Safepoint 的初始目的并不是让其他线程停下，而是找到一个稳定的执行状态。在这个执行状态下，JVM 的堆栈不会发生变化。这么一来，垃圾回收器便能够“安全”地执行可达性分析。

举个例子，当 Java 程序通过 JNI 执行本地代码时，如果这段代码不访问 Java 对象、调用 Java 方法或者返回至原 Java 方法，那么 Java 虚拟机的堆栈不会发生改变，也就代表着这段本地代码可以作为同一个安全点。

只要不离开这个安全点，JVM 便能够在垃圾回收的同时，继续运行这段本地代码。

由于本地代码需要通过 JNI 的 API 来完成上述三个操作，因此 Java 虚拟机仅需在 API 的入口处进行安全点检测（safepoint poll），测试是否有其他线程请求停留在安全点里，便可以在必要的时候挂起当前线程。

除了执行 JNI 本地代码外，Java 线程还有其他几种状态：解释执行字节码、执行即时编译器生成的机器码和线程阻塞。阻塞的线程由于处于 Java 虚拟机线程调度器的掌控之下，因此属于安全点。

其他几种状态则是运行状态，需要虚拟机保证在可预见的时间内进入安全点。否则，垃圾回收线程可能长期处于等待所有线程进入安全点的状态，从而变相地提高了垃圾回收的暂停时间。

对于解释执行来说，字节码与字节码之间皆可作为安全点。JVM 采取的做法是，当有安全点请求时，执行一条字节码便进行一次安全点检测。

执行即时编译器生成的机器码则比较复杂。由于这些代码直接运行在底层硬件之上，不受 Java 虚拟机掌控，因此在生成机器码时，即时编译器需要插入安全点检测，以避免机器码长时间没有安全点检测的情况。HotSpot 虚拟机的做法便是在生成代码的方法出口以及非计数循环的循环回边（back-edge）处插入安全点检测

为什么不在每一条机器码或者每一个机器码基本块处插入安全点检测呢？原因主要有两个：

1. 安全点检测本身也有一定的开销。不过 HotSpot 虚拟机已经将机器码中安全点检测简化为一个内存访问操作。在有安全点请求的情况下，Java 虚拟机会将安全点检测访问的内存所在的页设置为不可读，并且定义一个 segfault 处理器，来截获因访问该不可读内存而触发 segfault 的线程，并将它们挂起
2. 即时编译器生成的机器码打乱了原本栈桢上的对象分布状况。在进入安全点时，机器码还需提供一些额外的信息，来表明哪些寄存器，或者当前栈帧上的哪些内存空间存放着指向对象的引用，以便垃圾回收器能够枚举 GC Roots

由于这些信息需要不少空间来存储，因此即时编译器会尽量避免过多的安全点检测

不过，不同的即时编译器插入安全点检测的位置也可能不同。以 Graal 为例，除了上述位置外，它还会在计数循环的循环回边处插入安全点检测。其他的虚拟机也可能选取方法入口而非方法出口来插入安全点检测。

不管如何，其目的都是在可接受的性能开销以及内存开销之内，避免机器码长时间不进入安全点的情况，间接地减少垃圾回收的暂停时间。

除了垃圾回收之外，Java 虚拟机其他一些对堆栈内容的一致性有要求的操作也会用到安全点这一机制。

# Java Memory Model

JMM（Java Memory Model / Java 内存模型）：

- JMM describes how threads in the Java programming language interact through memory.
- Together with the description of single-threaded execution of code, the memory model provides the semantics of the Java programming language.

JMM specifies how and when different threads can see values written to shared variables by other threads, and how to synchronize access to shared variables when necessary.

## 指令重排：Java 内存访问重排序

Java 代码在运行的时候并不是严格按照代码语句顺序执行的， **编译器可能会对指令进行 Reordering（重排序）** 

> 大多数现代微处理器都会采用将指令乱序执行（out-of-order execution，简称 OoOE 或OOE）的方法，在条件允许的情况下，直接运行当前有能力立即执行的后续指令，避开获取下一条指令所需数据时造成的等待。
>
> 通过乱序执行的技术，处理器可以大大提高执行效率。
>
> 除了处理器，常见的 Java 运行时环境的 JIT 编译器也会做指令重排序操作，即生成的机器指令与字节码指令顺序不一致。

**所有的 Action（动作）都可以为了优化而被重排序，但是即时编译器、运行时和处理器都需要保证程序能够遵守 as-if-serial 属性。**

As-if-serial 语义：

- 在单线程情况下，要给程序一个顺序执行的假象
	- 也就是说，经过重排序的执行结果要与顺序执行的结果保持一致
- 如果两个操作之间存在数据依赖，那么即时编译器（和处理器）不能调整它们的顺序
- 否则，将会造成程序语义的改变

在下面的代码中，`y = 0 / 0` 可能会被重排序在 `x = 2` 之前执行，为了保证最终不致于输出 `x = 1` 的错误结果，JIT 在重排序时会在 `catch` 语句中插入错误代偿代码，将 `x` 赋值为2，将程序恢复到发生异常时应有的状态：

```java
public class Reordering {
    public static void main(String[] args) {
        int x, y;
        x = 1;
        try {
            x = 2;
            y = 0 / 0;
        } catch (Exception e) {
        } finally {
            System.out.println("x = " + x);
        }
    }
}
```

这种做法的确将异常捕捉的逻辑变得复杂了，但是 JIT 的优化的原则是，尽力优化正常运行下的代码逻辑，哪怕以 `catch` 块逻辑变得复杂为代价…… 毕竟，进入 `catch` 块内是一种“异常”情况的表现。

## JMM 与 happens-before

在单线程环境下，因为 *as-if-serial* 的保证，「指令重排」的结果是符合预期的。然而， **在多线程的情况下，可能会出现 Data Race（数据竞争）的情况** ，也就是多个线程读取内存中的数据导致数据不同步的问题。

不同硬件环境下「指令重排序」的规则是不同的。为此，JSR-1337 制定了 JMM（Java Memory Model / Java 内存模型），以 **提供一个统一的可参考的指令重排序（Reordering）规范，屏蔽平台差异性** 。从 Java 5 开始，JMM 成为 Java 语言规范的一部分。

**JMM 也可以让应用程序能够免于 Data Race（数据竞争）的干扰。** 

JMM 中最为重要的一个概念便是 **happens-before 关系** ：

- happens-before 关系用于描述两个操作的内存可见性
- happens-before 的前后两个操作不会被重排序，且后者对前者的内存可见

[Two actions can be ordered by a *happens-before* relationship](https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.5)：

- If one action *happens-before* another, then the first is visible to and ordered before the second
- 如果操作 X happens-before 操作 Y，那么 X 的结果对于 Y 可见

在同一个线程中，字节码的 program order（先后顺序）也暗含了 happens-before 关系：

- 在程序控制流路径中，靠前的字节码 happens-before 靠后的字节码
- 不过，这并不意味着前者一定在后者之前执行。实际上，如果后者没有观测前者的运行结果，即后者没有数据依赖于前者，那么它们可能会被重排序。

所有 happens-before 的规则如下：

- 程序次序法则：线程中的每个动作 A 都 happens-before 于该线程中的每一个动作 B，其中，在程序中，所有的动作 B 都能出现在 A之 后。
- 监视器锁法则：对一个监视器锁的解锁 happens-before 于每一个后续对同一监视器锁的加锁。
- volatile 变量法则：对 volatile 域的写入操作 happens-before 于每一个后续对同一个域的读写操作。
- 线程启动法则：在一个线程里，对 `Thread.start` 的调用会 happens-before 于每个启动线程的动作。
- 线程终结法则：线程中的任何动作都 happens-before 于其他线程检测到这个线程已经终结、或者从 `Thread.join` 调用中成功返回，或 `Thread.isAlive` 返回 `false` 。
- 中断法则：一个线程调用另一个线程的 `interrupt` happens-before 于被中断的线程发现中断。
- 终结法则：一个对象的构造函数的结束 happens-before 于这个对象 `finalizer` 的开始。
- 传递性：如果 A happens-before 于 B，且 B happens-before 于 C，则 A happens-before 于 C

---

在多线程中，解决 Data Race 问题的关键在于构造一个跨线程的 happens-before 关系，所以 JMM 对 `volatile` 的语义做了扩展：

- 保证 `volatile` 变量在一些情况下不会重排序
- `volatile` 的 64 位变量 `double` 和 `long` 的读取和赋值操作都是原子的

`volatile` 字段的 happens-before 关系指的是在两个不同线程中，`volatile` 的写操作 happens-before 之后对同一字段的读操作：

- 注意，这里的「之后」，指的是时间上的先后
- 也就是我这边写，你之后再读，这样就一定能读得到我刚刚写的值
- 而普通字段没有这个保证

![重排序示意表](https://p0.meituan.net/travelcube/94e93b3a7b49dc4c46b528fde1a03cd967665.png)

> 表中“第二项操作”的含义是指，第一项操作之后的所有指定操作。如，普通读不能与其之后的所有 volatile 写重排序。
>
> 另外，JMM 也规定了上述 volatile 和同步块的规则尽适用于存在多线程访问的情景。
>
> 例如，若编译器（这里的编译器也包括 JIT，下同）证明了一个 volatile 变量只能被单线程访问，那么就可能会把它做为普通变量来处理。
>
> 留白的单元格代表允许在不违反Java基本语义的情况下重排序。
>
> 例如，编译器不会对对同一内存地址的读和写操作重排序，但是允许对不同地址的读和写操作重排序。

JMM 还对 `final` 的语义做了扩展：

- 保证一个对象的构建方法结束前，所有 `final` 成员变量都必须完成初始化
- 前提是没有 `this` 引用溢出

参考资料： [Java内存访问重排序的研究](https://tech.meituan.com/2014/09/23/java-memory-reordering.html)

## Memory Barrier：JMM 的底层实现

**Memory Barrier（内存屏障，或内存栅栏，Memory Fence）是一种 CPU 指令，用于控制特定条件下的重排序和内存可见性** ：

- Java 编译器会根据 Memory Barrier 的规则禁止重排序
- 这也就是 JMM  禁止重排序的底层实现

对于即时编译器来说，它会针对每一个 happens-before 关系，向正在编译的目标方法中插入相应的读读、读写、写读以及写写 Memory Barrier（内存屏障）

这些 **Memory Barrier 会限制即时编译器的重排序操作**

以 `volatile` 字段访问为例，所插入的 Memory Barrier 将不允许 `volatile` 字段「写操作之前」的「内存访问」被重排序至其之后；也将不允许 `volatile` 字段读操作之后的内存访问被重排序至其之前

然后，即时编译器将根据具体的底层体系架构，将这些内存屏障替换成具体的 CPU 指令。以我们日常接触的 X86_64 架构来说，读读、读写以及写写内存屏障是空操作（no-op），只有写读内存屏障会被替换成具体指令

---

写缓存是处理器用来加速内存存储效率的一项技术： **在碰到内存写操作时，CPU 并不会等待该指令结束，而是直接开始下一指令，并且依赖于写缓存将更改的数据同步至 Main Memory（主内存）之中** ，这也就是 CPU Cache。

**强制刷新写缓存，会让当前线程写入 `volatile` 字段的值（以及写缓存中已有的其他内存修改），同步至 Main Memory（主内存）之中**

由于内存写操作同时会无效化其他处理器所持有的、指向同一内存地址的缓存行，因此可以认为其他处理器能够立即见到该 `volatile` 字段的最新值。

## Lock, volatile, final & Safe Publication

Lock（锁操作）具备 happens-before 关系：

- 解锁操作 happens-before 之后对同一把锁的加锁操作
- 实际上，在解锁时，Java 虚拟机同样需要强制刷新缓存，使得当前线程所修改的内存对其他线程可见

注意：锁操作的 happens-before 规则只适用于 **同一把锁**

也就是说，如果编译器能够（通过逃逸分析）证明某把锁仅被同一线程持有，那么它可以移除相应的加锁解锁操作。因此也就不再强制刷新缓存。

举个例子，即时编译后的 `synchronized (new Object()) {}` ，可能等同于空操作，而不会强制刷新缓存。

---

`volatile` 字段可以看成一种轻量级的、不保证原子性的同步，其性能往往优于（至少不亚于）锁操作。

然而，频繁地访问 `volatile` 字段也会因为不断地强制刷新缓存而严重影响程序的性能。

在 X86_64 平台上，只有 `volatile` 字段的写操作会强制刷新缓存。因此， **理想情况下 `volatile` 字段应当多读少写，并且应当只有一个线程进行写操作** 。

`volatile` 字段的另一个特性是即时编译器无法将其分配到寄存器里。换句话说， **`volatile` 字段的每次访问均需要直接从内存中读写** 。

> `volatile` 的变量是存在 Working Memory（工作内存）中的

> Working Memory 是 JMM 抽象出来的一个概念，可以映射到实际的CPU缓存

---

`final` 实例字段涉及新建对象的发布问题。

当一个对象包含 `final` 实例字段时，我们希望其他线程只能看到已初始化的 `final` 实例字段。

因此，即时编译器会在 `final` 字段的写操作后插入一个写屏障（Write Barrier），以防某些优化将新建对象的发布（即将实例对象写入一个共享引用中）重排序至 `final` 字段的写操作之前。

> 在 X86_64 平台上，写屏障是 no-op（空操作）

---

新建对象的 safe publication（安全发布）问题不仅仅包括 `final` 实例字段的可见性，还包括其他实例字段的可见性。

当发布一个已初始化的对象时，我们希望所有已初始化的实例字段对其他线程可见。否则，其他线程可能见到一个仅部分初始化的新建对象，从而造成程序错误。

扩展阅读：[Safe Object Publication in Java](https://vlkan.com/blog/post/2014/02/14/java-safe-publication/)

# JVM 是怎么实现 synchronized 的？

拓展阅读：

- [不可不说的Java“锁”事](https://tech.meituan.com/2018/11/15/java-lock.html) 或 [该文章的备份](https://github.com/LearnDifferent/my-notes/blob/full/Java%E7%9A%84%E9%94%81_Lock_JVM_%E7%BE%8E%E5%9B%A2%E6%8A%80%E6%9C%AF%E5%9B%A2%E9%98%9F_%E4%B8%8D%E5%8F%AF%E4%B8%8D%E8%AF%B4%E7%9A%84Java%E2%80%9C%E9%94%81%E2%80%9D%E4%BA%8B.md)
- [浅谈偏向锁、轻量级锁、重量级锁](https://juejin.cn/post/6844903550586191885)

## 抽象的锁实现

`synchronized` 可以对程序进行加锁：

- 声明一个 `synchronized` 代码块
- 直接标记静态方法或者实例方法

> 在 HotSpot 虚拟机中，monitor 是由 C++ 中 ObjectMonitor 实现

当声明 `synchronized` 代码块时，编译而成的字节码将包含 `monitorenter` 和 `monitorexit` 指令。

这两种指令均会消耗操作数栈上的一个引用类型的元素（也就是 `synchronized` 关键字括号里的引用），作为所要加锁解锁的锁对象。

```java
public class Foo {
    public void foo (Object obj) {
        synchronized (obj) {
            obj.hashCode();
        }
    }
}
```

上面的代码通过 `javap -c Foo` 可以得到：

```
Compiled from "Foo.java"
public class Foo {
  public Foo();
    Code:
       0: aload_0
       1: invokespecial #1  // Method java/lang/Object."<init>":()V
       4: return

  public void foo(java.lang.Object);
    Code:
       0: aload_1
       1: dup
       2: astore_2
       3: monitorenter
       4: aload_1
       5: invokevirtual #2  // Method java/lang/Object.hashCode:()I
       8: pop
       9: aload_2
      10: monitorexit
      11: goto          19
      14: astore_3
      15: aload_2
      16: monitorexit
      17: aload_3
      18: athrow
      19: return
    Exception table:
       from    to  target type
           4    11    14   any
          14    17    14   any
}
```

在字节码中，包含一个 `monitorenter` 指令以及多个 `monitorexit` 指令

这是因为 **JVM 需要确保 所获得的锁 在正常执行路径，以及异常执行路径上都能够被解锁**

可以对照字节码和 Exception table（异常处理表）来构造所有可能的执行路径，看看在执行了 `monitorenter` 指令之后，是否都有执行 `monitorexit` 指令。

---

当用 `synchronized` 标记方法时，Java 代码为：

```java
public class Foo {
    public synchronized void foo (Object obj) {
        obj.hashCode();
    }
}
```

使用 `javap -v Foo` 可以获得 `public synchronized void foo (Object obj)` 的字节码为：

```
public synchronized void foo(java.lang.Object);
    descriptor: (Ljava/lang/Object;)V
    flags: (0x0021) ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=1, locals=2, args_size=2
         0: aload_1
         1: invokevirtual #2  // Method java/lang/Object.hashCode:()I
         4: pop
         5: return
      LineNumberTable:
        line 3: 0
        line 4: 5
```

此时字节码中方法的 flags（访问标记）包含 `ACC_SYNCHRONIZED`

`ACC_SYNCHRONIZED`  表示在进入该方法时，JVM 需要进行 `monitorenter` 操作。而在退出该方法时，不管是正常返回，还是向调用者抛异常，JVM 均需要进行 `monitorexit` 操作。

这里 `monitorenter` 和 `monitorexit` 操作所对应的「锁对象」是隐式的：

- 对于实例方法来说，这两个操作对应的锁对象是 `this` 
- 对于静态方法来说，这两个操作对应的锁对象是则是所在类的 Class 实例

> 关于 `monitorenter` 和 `monitorexit` 的作用，可以抽象地理解为每个锁对象拥有一个锁计数器和一个指向持有该锁的线程的指针。

[monitorenter](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.monitorenter) ：

- Each object is associated with a monitor. A monitor is locked if and only if it has an owner.
- The thread that executes `monitorenter` attempts to gain ownership of the monitor associated with `objectref` (object reference), as follows:
- If the <u>entry count</u> of the monitor associated with `objectref` is 0, the thread enters the monitor and sets its entry count to 1. The thread is then the owner of the monitor
- If the thread already owns the monitor associated with `objectref`, it reenters the monitor, incrementing its <u>entry count</u>
- If another thread already owns the monitor associated with `objectref`, the thread blocks until the monitor's <u>entry count</u> is 0, then tries again to gain ownership

> 当执行 `monitorenter` 时，如果目标锁对象的计数器为 0，那么说明它没有被其他线程所持有。在这个情况下，JVM 会将该锁对象的持有线程设置为当前线程，并且将其计数器加 1。

> 在目标锁对象的计数器不为 0 的情况下，如果持有锁对象的线程是当前线程，那么 JVM 可以将其计数器加 1，否则需要等待，直至持有线程释放该锁。

[monitorexit](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.monitorexit) ：

- The thread that executes `monitorexit` must be the owner of the monitor associated with the instance referenced by `objectref` .
- The thread *decrements* the <u>entry count</u> of the monitor associated with `objectref`. 
- If as a result the value of the <u>entry count</u> is 0, the thread exits the monitor and is no longer its owner. Other threads that are blocking to enter the monitor are allowed to attempt to do so.

> 当执行 monitorexit 时，JVM 则需将锁对象的计数器减 1。当计数器减为 0 时，那便代表该锁已经被释放掉了。

之所以 **采用这种计数器的方式，是为了允许同一个线程重复获取同一把锁** 。

举个例子，如果一个 Java 类中拥有多个 `synchronized` 方法，那么这些 <u>方法之间的相互调用</u>，不管是直接的还是间接的，都<u>会涉及对同一把锁的重复加锁操作</u>。

因此，我们需要设计这么一个可重入的特性，来避免编程里的隐式约束。

这就是抽象的锁算法，下面就会介绍 JVM 具体的锁实现。

## Heavyweight Lock 重量级锁

> JVM 中 `synchronized` 关键字的实现，按照代价由高至低可分为重量级锁、轻量级锁和偏向锁三种

> Heavyweight Lock（重量级锁）会阻塞、唤醒请求加锁的线程。
>
> Heavyweight Lock 针对的是多个线程同时竞争同一把锁的情况。
>
> JVM 采取了 adaptive spinning（自适应自旋），来避免线程在面对非常小的 `synchronized` 代码块时，仍会被阻塞、唤醒的情况。

Heavyweight Lock（重量级锁）是 JVM  中最为基础的锁实现。在这种状态下，JVM 会 **阻塞加锁失败的线程，并且在目标锁被释放的时候，唤醒这些线程** 。

> Java 线程的阻塞以及唤醒，都是依靠操作系统来完成的。举例来说，对于符合 posix 接口的操作系统（如 macOS 和绝大部分的 Linux），上述操作是通过 pthread 的 **互斥锁（mutex）** 来实现的。此外，这些操作将涉及系统调用，需要从操作系统的用户态切换至内核态，其开销非常之大。

为了尽量避免昂贵的线程阻塞、唤醒操作，JVM 会 **在线程进入阻塞状态之前，以及被唤醒后竞争不到锁的情况下，进入自旋状态，在处理器上空跑并且轮询锁是否被释放** 。如果此时锁恰好被释放了，那么当前线程便无须进入阻塞状态，而是直接获得这把锁。

与线程阻塞相比，自旋状态可能会浪费大量的处理器资源。这是因为当前线程仍处于运行状况，只不过跑的是无用指令。它 **期望在运行无用指令的过程中，锁能够被释放出来** 。

可以用等红绿灯作为例子：Java 线程的阻塞相当于熄火停车，而自旋状态相当于怠速停车：

- 如果红灯的等待时间非常长，那么熄火停车相对省油一些
- 如果红灯的等待时间非常短，比如说我们在 `synchronized` 代码块里只做了一个整型加法，那么在短时间内锁肯定会被释放出来，因此怠速停车更加合适

然而，对于 JVM 来说，它并不能看到“红灯的剩余时间”，也就 **没办法根据等待时间的长短来选择自旋还是阻塞，所以 JVM 的方案是「自适应自旋」，即根据以往自旋等待时是否能够获得锁，来动态调整自旋的时间（循环数目）**

> **adaptive spinning** : An optimization technique whereby a thread spins waiting for a change-of-state to occur (typically a flag that represents some event has occurred - such as the release of a lock) rather than just blocking until notified that the change has occurred. The "adaptive" part comes from the policy decisions that control how long the thread will spin until eventually deciding to block.

就我们的例子来说，如果之前不熄火等到了绿灯，那么这次不熄火的时间就长一点；如果之前不熄火没等到绿灯，那么这次不熄火的时间就短一点。

**自旋状态还带来另外一个副作用，那便是不公平的锁机制。处于阻塞状态的线程，并没有办法立刻竞争被释放的锁。然而，处于自旋状态的线程，则很有可能优先获得这把锁**

## Lightweight Lock 轻量级锁

> 轻量级锁采用 CAS 操作，将锁对象的 Mark Word（标记字段）替换为一个指针。
>
> 该指针指向当前线 Thread 上的一块 Lock Record 空间。
>
> 该 Lock Record 存储着锁对象原本的 Mark Word。
>
> Lightweight Lock（轻量级锁）针对的是多个线程在不同时间段申请同一把锁的情况。

深夜的十字路口，车辆来往可能比较少，如果还设置红绿灯交替，那么很有可能出现四个方向仅有一辆车在等红灯的情况。

因此，红绿灯可能被设置为闪黄灯的情况，代表车辆可以通过，但司机需要注意观察。

JVM 也存在着类似的情形：<u>多个线程在不同的时间段请求同一把锁</u>，也就是说没有锁竞争。 **JVM 采用了轻量级锁，来避免重量级锁的阻塞以及唤醒** 。

---

Object Header（对象头）中 Mark Word（标记字段）的最后两位表示该 Object 的锁状态：

- 00 代表 Lightweight Lock（轻量级锁）
- 01 代表无锁或 Bias Lock（偏向锁）
- 10 代表 Heavyweight Lock（重量级锁）
- 11 则跟垃圾回收算法的标记有关

**Lightweight Lock 加锁操作** ：

1. 进行加锁操作时，JVM 会判断<u>是否已经是 Heavyweight Lock</u> （最后两位为“10”）

2. 如果不是 Heavyweight Lock，它会在<u>当前 Thread（线程）的当前 Stack Frame（栈桢）中划出一块空间，作为该 Lock 的 Lock Record</u>（锁记录）

3. JVM 会 <u>将 *锁对象* 的 Mark Word 拷贝到之前划出的 Lock Record 中</u>

4. 拷贝成功后，JVM 会尝试用 CAS（compare and swap）操作，<u>替换 *锁对象* 的 Mark Word</u>：

	1. 将 *锁对象的 Mark Word* 更新为 *指向 Lock Record 的指针*
	2. 将 Lock Record 里的 *owner 指针* 指向 *锁对象的 Mark Word*

5. 如果这个更新动作成功了，那么 *锁对象* Mark Word 的锁标志位就为“00”，表示这个<u>线程拥有了该 *锁对象* 的锁</u>：

	- 设置为“00”是为了「内存对齐」

	- “00”表示此 *锁对象* 处于轻量级锁定状态（Lightweight Lock）

6. 如果 Lightweight Lock 的更新操作失败了，JVM 会检查 *锁对象* 的 Mark Word <u>是否指向当前 Thread 的当前 Stack Frame</u>：

	1. 如果是，说明当前 Thread 已经有了这个 Object 的 Lock：
		1. 也就是<u>该 Thread 重复获取了同一个 Lock</u>
		2. 此时，JVM 会<u>将 Lock Record 清零，表示该 Lock 被重复获取</u>
		3. 然后直接进入同步块继续执行
	2. 如果不是，说明其他 Thread 持有该 Lock：
		1. 若当前<u>只有一个等待线程</u>，则当前 Thread 通过<u>自旋</u>进行等待
		2. 当<u>自旋超过一定的次数</u>，或者<u>一个线程在持有锁、一个在自旋、又有第三个来访</u>时，JVM 会将该 Lock <u>升级为 Heavyweight，并 Block（阻塞）当前 Thread</u>

**Lightweight Lock 解锁操作** ：

1. 当进行解锁操作时，<u>如果当前 Lock Record 的值为 0，则代表重复进入同一把锁，直接返回</u>即可
	- 一个线程的所有 Lock Record（锁记录）类似一个栈结构
	- 每次加锁压入一条 Lock Record，解锁弹出一条 Lock Record
	- 当前 Lock Record 指的便是栈顶的 Lock Record
2. 否则，JVM 会尝试用 CAS 操作，比较 *锁对象* 的 Mark Word 的值是否为当前 Lock Record 的地址：
	1. 如果是，则替换为 Lock Record 中的值，也就是 *锁对象* 原本的 Mark Word。此时，该 Thread 已经成功释放该 Lock
	2. 如果不是，则意味着该 Lock 已经被升级为 Heavyweight Lock。此时，JVM 会进入 Heavyweight Lock 的释放过程，唤醒因竞争该 Lock 而 blocked 的 Thread

## Bias Lock 偏向锁

> Bias Lock 只会在第一次请求时采用 CAS 操作，在锁对象的 Mark Word 中记录下当前 Thread 的地址。
>
> 在之后的运行过程中，持有该 Bias Lock 的 Thread 的加锁操作将直接返回。
>
> Bias Lock 针对的是 Lock 仅会被同一线程持有的情况。

如果说 Lightweight Lock 针对的情况很乐观，那么接下来的 Bias Lock 针对的情况则更加乐观： **从始至终只有一个线程请求某一把锁**

Since most objects are locked by at most one thread during their lifetime, we allow that thread to **bias** an object toward itself.

> 这就好比你在私家庄园里装了个红绿灯，并且庄园里只有你在开车。Bias Lock 的做法便是在红绿灯处识别来车的车牌号。如果匹配到你的车牌号，那么直接亮绿灯。

<u>Once biased, that thread can subsequently lock and unlock the object without resorting to expensive atomic instructions</u>. Obviously, an object can be biased toward at most one thread at any given time. (We refer to that thread as the bias holding thread)

具体来说，在线程进行加锁时，如果该 *锁对象* 支持 Bias Lock，那么 JVM 会通过 CAS 操作，将当前 Thread 的地址记录在锁对象的 Mark Word 中，并且将 Mark Word 的最后三位设置为 `101`

在接下来的运行过程中，每当有 Thread 请求这个 Lock，JVM 只需判断 *锁对象* 的 Mark Word 中：

1. 最后三位是否为 `101`
2. 是否包含当前线程的地址
3. epoch 值是否和锁对象的类的 epoch 值相同

如果都满足，那么当前 Thread 持有该 Bias Lock，可以直接返回。

---

**Bias Lock 的撤销**

If another thread tries to acquire a biased object, however, we need to **revoke** the bias from the original thread. At this juncture we can either <u>rebias the object</u> or simply <u>revert to normal locking</u> for the remainder of the object's lifetime.

The key challenge in revocation is to coordinate the revoker and the revokee (the bias holding thread) -- we must <u>ensure that the revokee doesn't lock or unlock the object during revocation</u>.

「当请求加锁的 Thread」和 「*锁对象*  Mark Word 中保存的 Thread 的地址」不匹配时：

- 如果 epoch 值不相等，当前 Thread 可以将该 Lock 偏向 （bias）自己
- 如果  epoch 值相等，JVM 需要撤销该 Bias Lock

**撤销 Bias Lock 需要持有该 Bias Lock  的 Thread 到达 Safepoint（安全点），再将 Bias Lock  替换为 Lightweight Lock**

Revocation can be implemented in various ways - signals, suspension, and safepoints to name a few.

如果某一<u>类</u> *锁对象* 的总撤销数超过了一个阈值（JVM 参数 `-XX:BiasedLockingBulkRebiasThreshold` ，默认为 20），那么 JVM 会宣布这个<u>类</u>的 **当前 epoch 值的 Bias Lock 失效** （不是“撤销”，而是“失效”）。具体的做法：

- 在每个类中维护一个 epoch 值
	- 可以理解为第几代 Bias Lock
- 当设置 Bias Lock 时，JVM 需要将该 epoch 值拷贝到 *锁对象* 的 Mark Word 中
- 宣布某个<u>类</u>的 Bias Lock 失效，实际上就是 JVM 将该<u>类</u>的 epoch 值 `+1` 
	- epoch 值 `+1` 后，表示 Bias Lock 更新为新的一代，所以之前那一代的 Bias Lock 就会失效
- 设置新的 Bias Lock 时，JVM 就会拷贝该<u>类</u>的新的 epoch 值

当某个<u>类</u>的 epoch 值发生变化时，为了保证当前已经对「该<u>类</u>的 Bias Lock 加锁」的 Thread 不受影响：

- JVM 会遍历所有线程的 Java 栈，找到该<u>类</u>已经加锁的 instance（实例）
- 将这些 instance 的 mark word 中的 epoch 值 `+1`
- 注意：该操作需要所有 Thread 处于 Safepoint 状态

如果总撤销数继续增加，且超过了<u>另一个阈值</u>（不是之前那个阈值，这个的参数为 `-XX:BiasedLockingBulkRevokeThreshold` ，默认值为 40），那么 JVM 会认为这个<u>类</u>已经不再适用于 Bias Lock 的情况

此时，JVM 会撤销该<u>类</u>所有 Instance（实例）的 Bias Lock，并且在之后的加锁过程中，直接为该<u>类</u>的 Instance 设置 Lightweight Lock

参考资料：[Biased Locking in HotSpot](https://blogs.oracle.com/dave/biased-locking-in-hotspot)

---

待补充：

- [ ] JVM 是怎么实现 invokedynamic 的？
- [ ] JVM 是如何实现反射的
- [ ] Java 语法糖与 Java 编译器
- [ ] 即时编译
