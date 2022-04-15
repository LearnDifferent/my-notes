# 使用 @WebMvcTest 进行单元测试（JUnit）的时候，出现 MyBatis 的 @MapperScan 注解报错，以及阻止拦截器（Interceptor）进行拦截

## 解决MyBatis 的 @MapperScan 注解报错

将 `@SpringBootApplication` 启动类上的 `@MapperScan` 之类的注解，移动到其他地方。比如新建一个 `@Configuration` 配置类，并在该配置类上使用 `@MapperScan("com.xxx.mapper")`。

参考：

- [Don’t component-scan in main Spring application class when using Spring Web MVC Test](https://stevenschwenke.de/DontComponentScanInMainSpringApplicationClassWhenUsingSpringWebMVCTest)
- [@WebMvcTest mapperscan conflict](https://stackoverflow.com/questions/38912710/webmvctest-mapperscan-conflict)

## 阻止自定义的拦截器（Interceptor）进行拦截

假设自定义的 `public class CustomWebConfig implements WebMvcConfigurer` 类里面，注册了拦截器。

那么在测试的时候，只需要加上：

```java
@MockBean
private CustomWebConfig config;
```

创建一个 MockBean 去替代自己的 Spring MVC 配置类，就能使拦截器整体失效。

参考：[How to avoid using an interceptor in Spring boot integration tests](https://stackoverflow.com/questions/63847932/how-to-avoid-using-an-interceptor-in-spring-boot-integration-tests)