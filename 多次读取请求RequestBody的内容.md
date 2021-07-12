# 使用Spring MVC多次读取请求Request Body的内容

### HttpServletRequest and Request Body

Spring MVC是建立在Servlet API之上的，其中Spring MVC的入口是一个Servlet，即Dispatcher Servlet。 因此，当我们处理HTTP请求时，HttpServletRequest为我们提供了两种读取请求主体的方法：`getInputStream`和`getReader`方法。 这些方法中的每一个都依赖于相同的InputStream。

因此，当我们一次读取`InputStream`时，就无法再次读取它。

现在，我们将研究如何访问原始请求内容，即使我们无法两次读取相同的InputStream。

### 使用ContentCachingRequestWrapper

Spring MVC提供了`ContentCachingRequestWrapper`类。它是原始`HttpServletRequest`对象的包装。 当我们读取请求正文时，`ContentCachingRequestWrapper`会缓存内容供以后使用。

创建一个web filter就可以使用它了

```java
import java.io.IOException;
import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.GenericFilterBean;
import org.springframework.web.util.ContentCachingRequestWrapper;

@Component
public class CachingRequestBodyFilter extends GenericFilterBean {

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest currentRequest = (HttpServletRequest) servletRequest;
        ContentCachingRequestWrapper wrappedRequest = new ContentCachingRequestWrapper(currentRequest);

        chain.doFilter(wrappedRequest, servletResponse);
    }
}
```

配置完上面的类后，就可以多次取出request body的内容了

### 使用示例



```java
@RestController
public class GreetController {

    @PostMapping("/greet")
    public String greet(@RequestBody String name, HttpServletRequest request) {
        ContentCachingRequestWrapper requestWrapper = (ContentCachingRequestWrapper) request;
        //name 和 此值是一样的
        String requestBody = new String(requestWrapper.getContentAsByteArray());
        return "Greetings " + requestBody;
    }
}
```

