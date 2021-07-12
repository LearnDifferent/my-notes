[toc]

# [SpringBoot系列——@Async优雅的异步调用](https://www.cnblogs.com/huanzi-qch/p/11231041.html)

## 前言

　　众所周知，java的代码是同步顺序执行，当我们需要执行异步操作时我们需要创建一个新线程去执行，以往我们是这样操作的：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
    /**
     * 任务类
     */
    class Task implements Runnable {

        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + "：异步任务");
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
        //新建线程并执行任务类
        new Thread(new Task()).start();
```

 　jdk1.8之后可以使用Lambda 表达式

```java
        //新建线程并执行任务类
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "：异步任务");
        }).start();
```

　　当然，除了显式的new Thread，我们一般通过线程池获取线程，这里就不再展开

 

　　Spring 3.0之后提供了一个**@Async**注解，使用@Async注解进行优雅的异步调用，我们先看一下API对这个注解的定义：https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/scheduling/annotation/Async.html

![img](https://img2018.cnblogs.com/blog/1353055/201907/1353055-20190723114404617-1578165620.png)

 

　　本文记录在SpringBoot项目中使用@Async注解，实现优雅的异步调用

 

##  　代码与测试

　　项目工程结构

![img](https://img2018.cnblogs.com/blog/1353055/201907/1353055-20190723120022180-2018930228.png)

　　因为要测试事务，所以需要引入

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```xml
        <!--添加springdata-jpa依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!--添加MySQL驱动依赖 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

　　在启动类开启启用异步调用，同时注入ApplicationRunner对象在启动类进行调用测试

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
package cn.huanzi.qch.springbootasync;

import cn.huanzi.qch.springbootasync.service.TestService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.stereotype.Component;

@Component
@EnableAsync//开启异步调用
@SpringBootApplication
public class SpringbootAsyncApplication {

    @Autowired
    private TestService testService;

    public static void main(String[] args) {
        SpringApplication.run(SpringbootAsyncApplication.class, args);
    }

    /**
     * 启动成功
     */
    @Bean
    public ApplicationRunner applicationRunner() {
        return applicationArguments -> {
            long startTime = System.currentTimeMillis();
            System.out.println(Thread.currentThread().getName() + "：开始调用异步业务");
            //无返回值
//            testService.asyncTask();

            //有返回值，但主线程不需要用到返回值
//            Future<String> future = testService.asyncTask("huanzi-qch");
            //有返回值，且主线程需要用到返回值
//            System.out.println(Thread.currentThread().getName() + "：返回值：" + testService.asyncTask("huanzi-qch").get());

            //事务测试，事务正常提交
//            testService.asyncTaskForTransaction(false);
            //事务测试，模拟异常事务回滚
//            testService.asyncTaskForTransaction(true);

            long endTime = System.currentTimeMillis();
            System.out.println(Thread.currentThread().getName() + "：调用异步业务结束，耗时：" + (endTime - startTime));
        };
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　看一下我们的测试业务类TestService

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
package cn.huanzi.qch.springbootasync.service;

import java.util.concurrent.Future;

public interface TestService {
    /**
     * 异步调用，无返回值
     */
    void asyncTask();

    /**
     * 异步调用，有返回值
     */
    Future<String> asyncTask(String s);

    /**
     * 异步调用，无返回值，事务测试
     */
    void asyncTaskForTransaction(Boolean exFlag);
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
package cn.huanzi.qch.springbootasync.service;

import cn.huanzi.qch.springbootasync.pojo.TbUser;
import cn.huanzi.qch.springbootasync.repository.TbUserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.AsyncResult;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.concurrent.Future;

@Service
public class TestServiceImpl implements TestService {

    @Autowired
    private TbUserRepository tbUserRepository;

    @Async
    @Override
    public void asyncTask() {
        long startTime = System.currentTimeMillis();
        try {
            //模拟耗时
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        long endTime = System.currentTimeMillis();
        System.out.println(Thread.currentThread().getName() + "：void asyncTask()，耗时：" + (endTime - startTime));
    }

    @Async("asyncTaskExecutor")
    @Override
    public Future<String> asyncTask(String s) {
        long startTime = System.currentTimeMillis();
        try {
            //模拟耗时
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        long endTime = System.currentTimeMillis();
        System.out.println(Thread.currentThread().getName() + "：Future<String> asyncTask(String s)，耗时：" + (endTime - startTime));
        return AsyncResult.forValue(s);
    }

    @Async("asyncTaskExecutor")
    @Transactional
    @Override
    public void asyncTaskForTransaction(Boolean exFlag) {
        //新增一个用户
        TbUser tbUser = new TbUser();
        tbUser.setUsername("huanzi-qch");
        tbUser.setPassword("123456");
        tbUserRepository.save(tbUser);

        if(exFlag){
            //模拟异常
            throw new RuntimeException("模拟异常");
        }
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

### 　　配置线程池

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
package cn.huanzi.qch.springbootasync.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.AsyncTaskExecutor;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

/**
 * 线程池的配置
 */
@Configuration
public class AsyncConfig {

    private static final int MAX_POOL_SIZE = 50;

    private static final int CORE_POOL_SIZE = 20;

    @Bean("asyncTaskExecutor")
    public AsyncTaskExecutor asyncTaskExecutor() {
        ThreadPoolTaskExecutor asyncTaskExecutor = new ThreadPoolTaskExecutor();
        asyncTaskExecutor.setMaxPoolSize(MAX_POOL_SIZE);
        asyncTaskExecutor.setCorePoolSize(CORE_POOL_SIZE);
        asyncTaskExecutor.setThreadNamePrefix("async-task-thread-pool-");
        asyncTaskExecutor.initialize();
        return asyncTaskExecutor;
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　配置好后，@Async会默认从线程池获取线程，当然也可以显式的指定@Async("asyncTaskExecutor")

 

### 　　无返回值

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
    /**
     * 启动成功
     */
    @Bean
    public ApplicationRunner applicationRunner() {
        return applicationArguments -> {
            long startTime = System.currentTimeMillis();
            System.out.println(Thread.currentThread().getName() + "：开始调用异步业务");

            //无返回值
            testService.asyncTask();

            long endTime = System.currentTimeMillis();
            System.out.println(Thread.currentThread().getName() + "：调用异步业务结束，耗时：" + (endTime - startTime));
        };
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 ![img](https://img2018.cnblogs.com/blog/1353055/201907/1353055-20190723115827369-137054437.png)

 

### 　　有返回值

　　有返回值，但主线程不需要用到返回值

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
    /**
     * 启动成功
     */
    @Bean
    public ApplicationRunner applicationRunner() {
        return applicationArguments -> {
            long startTime = System.currentTimeMillis();
            System.out.println(Thread.currentThread().getName() + "：开始调用异步业务");//有返回值，但主线程不需要用到返回值
            Future<String> future = testService.asyncTask("huanzi-qch");

            long endTime = System.currentTimeMillis();
            System.out.println(Thread.currentThread().getName() + "：调用异步业务结束，耗时：" + (endTime - startTime));
        };
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 ![img](https://img2018.cnblogs.com/blog/1353055/201907/1353055-20190723120750063-1637394085.png)

　　有返回值，且主线程需要用到返回值

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
    /**
     * 启动成功
     */
    @Bean
    public ApplicationRunner applicationRunner() {
        return applicationArguments -> {
            long startTime = System.currentTimeMillis();
            System.out.println(Thread.currentThread().getName() + "：开始调用异步业务");
//有返回值，且主线程需要用到返回值
            System.out.println(Thread.currentThread().getName() + "：返回值：" + testService.asyncTask("huanzi-qch").get());

            long endTime = System.currentTimeMillis();
            System.out.println(Thread.currentThread().getName() + "：调用异步业务结束，耗时：" + (endTime - startTime));
        };
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

![img](https://img2018.cnblogs.com/blog/1353055/201907/1353055-20190723121014479-355020614.png)

　　可以发现，有返回值的情况下，虽然异步业务逻辑是由新线程执行，但如果在主线程操作返回值对象，主线程会等待，还是顺序执行 

 

### 　　事务测试

　　为了方便观察、测试，我们在配置文件中将日志级别设置成debug

```yaml
#修改日志登记，方便调试
logging.level.root=debug
```

 

　　事务提交

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
    /**
     * 启动成功
     */
    @Bean
    public ApplicationRunner applicationRunner() {
        return applicationArguments -> {
            long startTime = System.currentTimeMillis();
            System.out.println(Thread.currentThread().getName() + "：开始调用异步业务");//事务测试，事务正常提交
            testService.asyncTaskForTransaction(false);

            long endTime = System.currentTimeMillis();
            System.out.println(Thread.currentThread().getName() + "：调用异步业务结束，耗时：" + (endTime - startTime));
        };
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

![img](https://img2018.cnblogs.com/blog/1353055/201907/1353055-20190723121259049-1268764937.png)

![img](https://img2018.cnblogs.com/blog/1353055/201907/1353055-20190723115133055-806425816.png)

![img](https://img2018.cnblogs.com/blog/1353055/201907/1353055-20190723115333527-573926793.png)

 

　　模拟异常，事务回滚

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```java
    /**
     * 启动成功
     */
    @Bean
    public ApplicationRunner applicationRunner() {
        return applicationArguments -> {
            long startTime = System.currentTimeMillis();
            System.out.println(Thread.currentThread().getName() + "：开始调用异步业务");
//事务测试，模拟异常事务回滚
            testService.asyncTaskForTransaction(true);

            long endTime = System.currentTimeMillis();
            System.out.println(Thread.currentThread().getName() + "：调用异步业务结束，耗时：" + (endTime - startTime));
        };
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

![img](https://img2018.cnblogs.com/blog/1353055/201907/1353055-20190723121249738-1640041332.png)

![img](https://img2018.cnblogs.com/blog/1353055/201907/1353055-20190723113653670-484594019.png)

 ![img](https://img2018.cnblogs.com/blog/1353055/201907/1353055-20190723115412812-758023998.png)

 

## 　　后记

　　SpringBoot使用@Async优雅的异步调用就暂时记录到这里，以后再进行补充

 

## 　　代码开源

　　代码已经开源、托管到我的GitHub、码云：

　　GitHub：https://github.com/huanzi-qch/springBoot

　　码云：https://gitee.com/huanzi-qch/springBoot