#####  webjars

```xml
<dependency>
            <groupId>org.webjars.bower</groupId>
            <artifactId>jquery</artifactId>
            <version>3.5.1</version>
</dependency>

# 访问方式：http://localhost:8080/webjars/jquery/3.5.1/dist/jquery.js
# 静态资源将遍历所有的依赖，查找resources模块是否存在该资源
```

##### 自定义资源

```properties
/** 访问当前项目任何资源
"classpath:/META-INF/resources/""
"classpath:/resources/"
“classspath:/static/"
"classpath:/public"
"/"
```

#####  thymeleaf

​	```

```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

```xml
# 使用thymeleaf3
<properties>
        <java.version>1.8</java.version>
        <thymeleaf.version>3.0.9.RELEASE</thymeleaf.version>
        <thymeleaf-layout-dialect.version>2.1.1</thymeleaf-layout-dialect.version>
</properties>
```

```java
// thymeleaf自动装配属性类
@ConfigurationProperties(
    prefix = "spring.thymeleaf"
)
public class ThymeleafProperties {
    private static final Charset DEFAULT_ENCODING;
    public static final String DEFAULT_PREFIX = "classpath:/templates/";
    public static final String DEFAULT_SUFFIX = ".html";
    private boolean checkTemplate = true;
    private boolean checkTemplateLocation = true;
    private String prefix = "classpath:/templates/";
    private String suffix = ".html";
    private String mode = "HTML";
    // 所以只要放到"classpath:/templates/"就能自动渲染了
```

```
	@RequestMapping("success")
    public Map<String,Object> success(Map<String,Object> map){
        map.put("hello","你好");
        return map;
    }
```

```html
<div id="" th:text="${hello}"></div>
```

