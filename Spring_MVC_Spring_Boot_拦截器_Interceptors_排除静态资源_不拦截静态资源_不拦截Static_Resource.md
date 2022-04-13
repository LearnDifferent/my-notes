# Spring MVC / Spring Boot 拦截器 Interceptors 排除静态资源 / 不拦截静态资源 / 不拦截 Static Resource

因为 Spring Boot 会默认将一些目录注册为 classpath 用于存放资源，所以 `/` 也表示那些目录的根目录。

又因为 Spring Boot 会自动将 `/` 转发到 classpath 目录中的 index.html 中，所以设置不拦截静态资源的时候，一定要记得把 `/` 加入到排除的路径列表中，否则就无法完成访问 `/` 自动转发到 index.html 的操作。

```java
@Configuration
public class CustomWebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new SaRouteInterceptor())
                .addPathPatterns("/**")
                .excludePathPatterns("/", "/css/**", "/js/**", "/img/**", "/favicon.ico", "/index.html");
    }

}
```
