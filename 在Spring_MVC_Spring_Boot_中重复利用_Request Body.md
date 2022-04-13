# 在 Spring MVC / Spring Boot 中，重复利用 Request Body

## 过滤器配置

```java
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<RequestBodyCacheFilter> registerRequestBodyCacheFilter() {

        // 配置自定义的 Request Body 的过滤器
        FilterRegistrationBean<RequestBodyCacheFilter> bean = new FilterRegistrationBean<>();
        bean.setFilter(new RequestBodyCacheFilter());
        bean.setOrder(1);
      	// 使用该过滤器的路径
        bean.addUrlPatterns("/log/in");

        return bean;
    }
}

class RequestBodyCacheFilter extends GenericFilterBean {

    @Override
    public void doFilter(ServletRequest servletRequest,
                         ServletResponse servletResponse,
                         FilterChain filterChain)
            throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) servletRequest;

        ContentCachingRequestWrapper wrapperRequest = new ContentCachingRequestWrapper(request);

        filterChain.doFilter(wrapperRequest, servletResponse);
    }
}
```

## 使用

```java
/**
 * 获取可以重复使用的 Request Wrapper
 *
 * @return {@code ContentCachingRequestWrapper}
 */
private ContentCachingRequestWrapper getRequestWrapper() {
    ServletRequestAttributes attributes =
            (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
    assert attributes != null;
    return (ContentCachingRequestWrapper) attributes.getRequest();
}
```

```java
/**
 * 获取 Request Body
 *
 * @param requestWrapper Request Wrapper
 * @return {@code Map<String, String>} key 为参数名，value 为参数值
 */
private Map<String, String> getRequestBodyFromRequest(ContentCachingRequestWrapper requestWrapper) {

    // 将 request wrapper 中的参数转化为 byte 数组
    byte[] contentAsByteArray = requestWrapper.getContentAsByteArray();

    // 将该数组转化为字符串
    String json = new String(contentAsByteArray);

    // 获取 Map<String, String> 的 TypeReference，用于下一步的转换操作
    TypeReference<Map<String, String>> typeRef = new TypeReference<Map<String, String>>() {};

    // 使用自己习惯的工具，将 json 字符串转换为 Map<String, String> 并返回，即可重复利用
    return JsonUtils.toObject(json, typeRef);
}
```