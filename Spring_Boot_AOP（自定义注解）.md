[toc]

# AOP 概念

## 基础概念

对比：

* AOP（Aspect Oriented Programming）：以切面（切入点+额外功能）为基本单位的程序开发，通过对象间的彼此协同、互相调用，完成程序的构建
* POP（Producer Oriented Programming）：以过程（方法、函数）为基本单位的程序开发，通过对象间的彼此协同、互相调用，完成程序的构建
* OOP（Object Oriented Programming）：以对象为基本单位的程序开发，通过对象间的彼此协同、互相调用，完成程序的构建

本质：**动态代理开发，通过代理类为原始类增加额外功能**

好处：利于原始类的维护

## 切面的名词解释

几何学中，`面 = 点 + 相同点性质` ，比如：桌面由有木头属性的点组成。

在编程中，假设有 UserServiceImpl、OrderServiceImpl 和  ProductServiceImpl，这三个 S ervice 实现类中，都有需要加上某个功能的方法。

所以，这三个类中的每个方法，都类似于 *点* ，都具有相同的性质（这个性质，指的是需要添加的功能）。

所以，**切面 = 切入点 + 额外功能** 。

## 创建代理的三要素

1. 原始对象
2. 额外功能
3. 让代理对象和原始对象实现相同的接口

「让代理对象和原始对象实现相同的接口」的原因：

1. 保证 *代理类* 和 *原始类* 方法一致，迷惑调用者
2. 代理类可以在执行原始方法的前后（包括抛出异常），添加额外的功能

# AOP 的底层实现原理

## JDK 的动态代理

先看一下 org.aopalliance.intercept 包下的 MethodInterceptor 接口：

```java
// 这是 org.aopalliance.intercept 包下的接口
public interface MethodInterceptor extends Interceptor {
  	// 返回值是 Object，表示“原始方法的返回值”
    Object invoke(MethodInvocation invocation) throws Throwable;
}
```

这里写一个类，继承 MethodInterceptor 接口，来演示这个接口的使用方法：

```java
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;

public class AOPTest implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {

        // 可以在这里添加额外的功能
      	System.out.println("------在方法之前执行------");

        // invocation.proceed() 方法可以获取原始方法的返回值
        Object result = invocation.proceed();

        // 也可以在这里添加额外的功能
      	System.out.println("------在方法之后执行------");

        // 最后，返回原始方法的返回值
        return result;
    }
}
```

有了这个基础认知之后，看一下 JDK的 java.lang.reflect 包下，实现动态代理的主要方法：

```java
// 返回值是 Object，返回值表示创建好的动态代理对象
Object Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler handler)
```

参数 `InvocationHandler handler` 表示“添加额外功能”。我们可以在原始方法执行之前、执行之后以及抛出异常的时候，为原始方法添加“额外功能”，这个 `InvocationHandler` 接口就和之前提到的 `MethodInterceptor` 接口类似：

```java
public interface InvocationHandler {
  // 返回值 Object 表示“原始方法的返回值”
  public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```

`InvocationHandler` 有三个参数：

* `Object proxy` 可以忽略，代表“代理对象”，没有什么用
* `Method method`  代表“需要添加额外功能的原始方法”
* `Object[] args` 代表“原始方法的参数”

一般来说，添加额外功能，需要“对象/实例”、“对象的方法”和“参数”，比如说 `userService.login("john");` 中的 `userService` 对应“对象/实例”，`.login()` 表示“对象的方法”，而 `"john"` 表示参数。

也就是说，`InvocationHandler` 接口中的 `Method method` 对应 `.login()`，`Object[] args` 就对应 `"john"`  ，但是  `Object proxy` 是“代理对象”，而不是“原始对象”，所以不能对应 `userService` ，这个时候，就需要使用反射：

```java
// userService 是从 UserService userService = new UserServiceImpl(); 中来的
// args 是 Object[] args，也就是传入的“原始方法的参数”
method.invoke(userService, args);
// 返回值是 Object，表示执行该方法的结果
```

`Method method` 中的 `invoke` 方法，可以传入 `userService` ，来表示“原始对象”。这样，就凑齐了 “对象/实例”、“对象的方法”和“参数” 三个要素，这个 `method.invoke()` 方法的返回值，就是原始对象的该方法的返回值。

完整的代码如下：

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class AOPTest implements InvocationHandler {

    UserService userService = new UserServiceImpl();

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        System.out.println("------在方法之前执行------");
        Object result = method.invoke(userService, args);
        System.out.println("------在方法之后执行------");

        return result;
    }
}
```

***

除了 `InvocationHandler handler` 参数，JDK 中的 `Proxy.newProxyInstance()` 还有 `Class<?>[] interfaces` 参数：

```java
// interfaces 表示：原始对象所实现的接口
Object Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler handler)
```

这个参数可以通过反射来获取，这里假设要获取 UserService 的原始接口，可以这样：

```java
Class<?>[] interfaces = UserService.class.getInterfaces();
```

***

最后还有 `ClassLoader loader` 参数：

```java
Object Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler handler)
```

ClassLoader 是类加载器，主要作用有：

1. 通过 ClassLoader 将 Class 中的字节码文件加载到 JVM
2. 通过 ClassLoader 创建 Class 对象（某个类的 Class 对象，有这个类所有的信息），根据 Class 对象获取到的类信息，来创建类的实例对象：
	* 比如：`User user = new User();`
	* `User.java` 这个类，编译为 `User.class` 
		* `User.class` 包含了 `User.java` 对应的字节码
	* ClassLoader 将 `User.class` 加载到 JVM 中，然后生成 *User.java 类的 Class 实例对象* 
		* 类似于：`Class<User> userClass = User.class;` ，这个 `userClass` 实例对象就是 ClassLoader 生成的 Class 实例对象
	* 然后根据生成的 `User.java` 的 Class 实例对象，来 `new User()` ，从而创建 `user` 实例对象

**JVM 正常情况下会给每个 `.class` 文件自动分配对应的 ClassLoader，但是在动态代理 `Proxy.newProxyInstance()` 方法中，是通过动态字节码技术创建字节码，然后直接载入 JVM 中，而没有编译为 `.class` 文件的过程，所以 JVM 在动态代理的过程中，不会提供 ClassLoader** 

这种情况下，需要**借用一个 ClassLoader，来创建代理类的 Class 对象** （这个借用的 ClassLoader，随便一个类的 ClassLoader 都可以）。

***

JDK 实现动态代理的完整代码：

```java
@Test
void test() {
    // 创建原始对象（JDK 8 之前要加上 final）
    UserService userService = new UserServiceImpl();
  
    // 创建内部类 InvocationHandler（用于创建动态代理类）
    InvocationHandler handler = new InvocationHandler() {
      
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
          
            System.out.println("------在方法之前运行------");
          
            // 通过反射，获取原方法的结果
            Object res = method.invoke(userService, args);
          
            System.out.println("------在方法之后运行------");
          
            // 返回原方法的结果
            return res;
        }
    };
  
    // 根据 JDK 的 java.lang.reflect.Proxy 创建动态代理类
    // 这里是创建 UserService 的动态代理类，该实例对象命名为 userServiceProxy
    UserService userServiceProxy = (UserService) Proxy.newProxyInstance(
            UserService.class.getClassLoader() // 随便借用一个 ClassLoader（哪个类的都行）
            , userService.getClass().getInterfaces() // 获取原始类的接口，用于实现代理
            , handler // 方法增强
    );
  
    // 然后，可以通过动态代理类的实例，来完成原方法的增强
    userServiceProxy.getUserByName("john");
}
```

## Cglib 的动态代理

和 JDK 实现代理的不同：

1. JDK 是通过实现原始类的接口，来完成动态代理
2. Cglib 是通过继承原始类，然后在  `super.方法` 的前后，来添加额外功能，以此来完成动态代理

实现代码：

```java
@Test
void test() {
    // 创建原始对象
    UserService userService = new UserServiceImpl();
  
    // 通过 Cglib 的 Enhancer 实例来实现动态代理
    Enhancer enhancer = new Enhancer();
  
    // 随便借用一个 ClassLoader
    enhancer.setClassLoader(UserService.class.getClassLoader());
  
    // 设置父类为原始类
    enhancer.setSuperclass(userService.getClass());
  
    // 调用原方法，并设置额外功能/增强功能
    enhancer.setCallback(new MethodInterceptor() {
        // 通过 MethodInterceptor 的匿名类来实现额外功能
        @Override
        public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
            System.out.println("------在方法之前运行------");
          
            // 调用原始方法，并传入原始类的实例对象，以及原始方法的参数，获得原始方法的结果
            Object res = method.invoke(userService, args);
          
            System.out.println("------在方法之后运行------");
          
          	// 返回原方法的结果
            return res;
        }
    });
  
  	// 通过 Enhancer 来创建动态代理类
		UserService userServiceProxy = (UserService) enhancer.create();
  
		// 调用动态代理类的方法
		userServiceProxy.getUserByName("sally");
}
```

# AOP 编程实战

## AOP 编程的开发步骤

1. 原始对象
2. 额外功能（MethodInterceptor）
3. 切入点
4. 组装切面：额外功能 + 切入点

## 引入 AOP 依赖

引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

## Aspect 类 + 切入点表达式（execution）

Aspect 类：

```java
@Aspect
public class UserServiceAspect {

    /**
     * 相当于 MethodInterceptor 接口的 Object invoke(MethodInvocation invocation) 方法
     * ，表示原方法及其增强（额外功能）
     *
     * @param joinPoint 相当于 Object intercept() 中的 invocation.proceed();
     *                  用于调用原方法，并获取原方法的返回值
     * @return 返回值 Object 表示原方法的返回值
     * @throws Throwable 抛出异常
     */
    @Around("execution(* service.impl.*.*(..))")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("-----方法之前------");
        Object res = joinPoint.proceed();
        System.out.println("-----方法之后------");
        return res;
    }

    ///////////////////////////////////////////////////////////

    /**
     * 切入点的复用：
     * 这里抽取出来切入点表达式 execution
     */
    @Pointcut("execution(* service.impl.*.*(..))")
    public void customPointCut() {}

    /**
     * 这个方法的效果和上面的 around 方法一致
     * ，只不过使用了抽取出来的 execution
     * ，这就叫切入点的复用   
     */
    @Around(value = "customPointCut()")
    public Object around1(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("-----方法之前------");
        Object res = joinPoint.proceed();
        System.out.println("-----方法之后------");
        return res;
    }
}
```

## Aspect 类 + 注解类

如果不想写切入点表达式（execution），还可以通过注解类来指定作用范围。只用了该注解的地方才会使用切面编程。

注解类：

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface UserServiceLog {
    String value() default "";
}
```

Aspect 类：

```java
@Aspect
@Component
public class UserServiceLogAspect {

    @Pointcut(value = "@annotation(UserServiceLog)")
    public void pointcut() {
    }

    @Around(value = "pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("========在方法开始前输出========");
        return joinPoint.proceed();
    }
}
```

> 想在哪里使用，就将注解放在哪里

## 调用代理方法的正确写法

需要注意的是，如果业务比较复杂，需要调用代理类的方法（而不是原方法）的时候，要先获取 `ApplicationContext` ：

```java
public class UserServiceImpl implements UserService, ApplicationContextAware {

    private ApplicationContext applicationContext;

    /**
     * 继承了 ApplicationContextAware 接口后，需要实现这个方法
     * ，用于获取 ApplicationContext（相当于依赖注入）
     */
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    /**
     * 在业务比较复杂的情况下，可能会出现需要调用代理类的方法的情况
     * ，这里不能使用 this.getUserNameById(1)，因为这样只会调用当前类的方法
     * ，如果需要使用代理类的该方法，必须这样做：
     */
    public void printUserId1() {
        // 获取 Spring 容器内的 UserService（和当前的 UserService 不同，这个是代理类）
        UserService userServiceProxy = (UserService) applicationContext.getBean("userService");
        // 调用代理类的方法
        String userId1 = userServiceProxy.getUserNameById(1);
        System.out.println(userId1);
    }

    // 这是模拟根据 ID 获取用户的情况，也是需要被调用的原方法 
    @Override
    public String getUserNameById(int id) {
        if (id == 1) {
            return "john";
        } else {
            return "sally";
        }
    }
}
```

