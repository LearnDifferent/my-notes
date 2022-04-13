# 如何在 Spring Boot / Spring MVC 中，将传入的字符串（String）类型参数转换为枚举（Enum）类型的值？

## 最简单的方法：implements Converter<S, T>

Converter：

1. 创建一个 Converter 类，并 `implements Converter<String, T>`
2. 重写 `public T convert(String source)` 方法，该方法的返回值就是新生成的枚举

配置类：

1. 创建一个 `@Configuration` 配置类
2. 使用 `implements WebMvcConfigurer` ，并重写（Override）`public void addFormatters(FormatterRegistry registry)` 方法
3. 使用 `registry.addConverter(new 某某Converter())` 注册一个新的 Converter

## 最推荐的方法

接口：

1. 创建一个接口（interface），并在其中，创建一个返回值为 `String[]` 的方法。该方法 return 的，就是可以将 String 映射为 Enum 的参照值
2. 在需要被转换的 Enum 中，实现（implement）该接口，并 Override 那个返回值为 `String[]` 方法，返回 `return new String[]{this.name(), this.属性, ...};` 等可以用作映射的名称

```java
public interface ExampleInterface {
    String[] exampleMethod();
}
```

Converter：

```java
class ExampleConverter<E extends ExampleInterface> implements Converter<String, E> {

    private final Map<String, E> names;

    public ExampleConverter(Class<E> type) {
        names = new HashMap<>();
        E[] es = type.getEnumConstants();
        Arrays.stream(es).forEach(e -> {
            String[] nfc = e.namesForConverter();
            Arrays.stream(nfc).forEach(n -> names.put(n, e));
        });
    }

    @Override
    public E convert(String source) {
        E e1 = names.get(source.toLowerCase());
        E e2 = names.get(source.toUpperCase());
        if (e1 == null && e2 == null) {
            throw new ServiceException("Can't convert " + source);
        }
        return e1 != null ? e1 : e2;
    }
}
```

ConverterFactory：

```java
public class ExampleConverterFactory implements ConverterFactory<String, ExampleConverter> {

    @Override
    public <T extends ExampleConverter> Converter<String, T> getConverter(Class<T> targetType) {
        return new ExampleConverter<>(targetType);
    }
}
```