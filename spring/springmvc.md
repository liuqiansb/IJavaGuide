##### WEB.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath*:spring/spring-context.xml</param-value>
    </context-param>
    <servlet>
        <servlet-name>springMVC</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
       	<!-- 
			servlet的配置参数如下方式获取
			ServletConfig config = getServletConfig();
			config.getInitParameter("contextConfigLocation");
		-->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath*:spring/spring-context.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
        <async-supported>true</async-supported>
    </servlet>
    <servlet-mapping>
        <servlet-name>springMVC</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```

##### spring-mvc.xml

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context-3.0.xsd http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd">
    <!-- 开启注解扫描 -->
    <context:component-scan base-package="com.kiy.shiro.controller" />
     <!--
		配置静态资源文件映射后必须开启注解驱动
		否则DispatcherServlet无法区分请求是资源文件还是mvc的注解
	-->
    <mvc:annotation-driven />

	<!-- 配置视图解析器 -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/pages/" />
        <property name="suffix" value=".jsp" />
    </bean>
    
    <!--静态页面，如html,css,js,images可以访问-->
	<mvc:default-servlet-handler />
   	<!-- 配置静态文件映射 -->
    <mvc:resources mapping="/javascript/**" location="/assets/js/"/>
    <mvc:resources mapping="/styles/**" location="/assets/css/"/>
    <mvc:resources mapping="/html/**" location="/assets/html/"/>

    
</beans>
```

```java
@Controller
public class TestController {
    // rest风格的ajax请求接口
    @RequestMapping(value = "test",method = RequestMethod.GET)
    @ResponseBody
    public String test(){
        return "this";
    }
    
    // 配合jstl使用的视图
    @RequestMapping(value = "index",method = RequestMethod.GET)
    public ModelAndView index(){
        ModelAndView modelAndView = new ModelAndView();
        modelAndView.addObject("Title","测试");
        modelAndView.setViewName("index");
        return modelAndView;
    }
} 
```

