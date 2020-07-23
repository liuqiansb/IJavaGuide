>  谨记：自定义Starter的作用就是给ioc容器加入一个定制化的bean，而在创建这个bean的时候可以进行一些操作，并且定义查询properties的属性前缀，由此可以在主配置文件中定义该bean的属性

##### starter

1. 这个场景需要使用的依赖是什么
2. 如何编写自动配置

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
		ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {
	public static final String DEFAULT_PREFIX = "";
	public static final String DEFAULT_SUFFIX = "";
	private static final String[] SERVLET_LOCATIONS = { "/" };
	@Bean
	@ConditionalOnMissingBean(HiddenHttpMethodFilter.class)
	@ConditionalOnProperty(prefix = "spring.mvc.hiddenmethod.filter", name = "enabled", matchIfMissing = false)
	public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {
		return new OrderedHiddenHttpMethodFilter();
	}
	...
```

```java
@Configuration  //指定这个类是一个配置类
@ConditionalOn //在指定条件成立的情况下自动配置类生效
@AutoConfigureAfter // 指定自动配置类的顺序
@Bean // 向容器中注入bean

@ConfigurationProperties // 编写xxxProperties来绑定相关配置
@EnableConfigurationProperties // 让xxxProperties生效加入到容器中

自动配置类需要能加载，配置在META-INF/spring.factories中
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
```

3. 模式

   启动器:kiy-spring-boot-starter,只用来做依赖导入，依赖自动配置模块

   自动配置模块:kiy-spring-boot-configurer,用来写导入bean,bean配置类，bean自动配置类

   ```xml
   <!-- kiy-spring-boot-starter -->
   <dependencies>
           <dependency>
               <groupId>com.kiy.configurer</groupId>
               <artifactId>kiy-spring-boot-configurer</artifactId>
               <version>0.0.1-SNAPSHOT</version>
           </dependency>
   </dependencies>
   <!--
   	该模块只做引入工作，供用户作为依赖项
   -->
   ```

   ```xml
   <!-- kiy-spring-boot-configurer -->
   <dependencies>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter</artifactId>
           </dependency>
    </dependencies>
    <!--
    	  任何一个starter都必须引入 spring-boot-starter
         因为类似于@ConfigurationProperties都存在于它的依赖中
    -->   
   ```

   ```java
   /**
    * 我们想要放入客户端IOC容器的类
    */
   public class HelloService {
       HelloProperties helloProperties;
       public String sayHelloAtKiy(String name){
           return helloProperties.getPrefix()+"-"+name+"-"+helloProperties.getSuffix();
       }
   
       public HelloProperties getHelloProperties() {
           return helloProperties;
       }
   
       public void setHelloProperties(HelloProperties helloProperties) {
           this.helloProperties = helloProperties;
       }
   }
   ```

   ```java
   /**
    * 配置规则类
    */
   @ConfigurationProperties(prefix = "kiy.hello")
   public class HelloProperties {
       private String prefix;
       private String suffix;
   
       public String getPrefix() {
           return prefix;
       }
   
       public void setPrefix(String prefix) {
           this.prefix = prefix;
       }
   
       public String getSuffix() {
           return suffix;
       }
   
       public void setSuffix(String suffix) {
           this.suffix = suffix;
       }
   }
   
   ```

   ```java
   /**
    *	自动装配类
    */ 
   @Configuration
   @ConditionalOnWebApplication
   @EnableConfigurationProperties(HelloProperties.class) // 作用就是把HelloProperties放入IOC容器
   public class HelloServiceAutoConfiguration {
       @Autowired
       HelloProperties helloProperties;
       @Bean
       public HelloService helloService(){
           HelloService helloService = new HelloService();
           helloService.setHelloProperties(helloProperties);
           return helloService;
       }
   }
   ```

   ```properties
   # MATA-INF spring.factories 客户端在使用该starter时将会扫描此处
   org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
   com.kiy.configurer.kiyspringbootconfigurer.HelloServiceAutoConfiguration
   ```

   





























