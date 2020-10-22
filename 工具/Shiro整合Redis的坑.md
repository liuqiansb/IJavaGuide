##### Shiro整合Redis的坑

- doGetAuthenticationInfo只有在登录那一次调用

- doGetAuthorizationInfo方法每一次请求都会调用,除非设置缓存
- 默认情况下Session中并没有存放用户的权限信息，而只有用户名和ip，所以doGetAuthorizationInfo每一次请求都会调用重新进行授权
- 无论是使用拦截器还是注解进行的权限控制，都会调用doGetAuthorizationInfo方法，初步判断应该是第一次使用doGetAuthenticationInfo并存入session,后面每一次都使用session中的用户信息重新拉取权限信息



账号：3530578

密码：251303

##### shiro简介

Apache Shiro 是 Java 的一个安全框架，提供以下功能

![img](C:\Users\皿煮国的潜逃败类\Desktop\1.png)

- **Authentication**:身份认证 / 登录，验证用户是不是拥有相应的身份；
- **Authorization**：授权，即权限验证，验证某个已认证的用户是否拥有某个权限；即判断用户是否能做事情
- **Session** **Management**：会话管理
- **Cryptography**：加密
- **Web Support**：Web 支持,可以非常容易的集成到 Web 环境；
- **Caching**：缓存，比如用户登录后，其用户信息、拥有的角色 / 权限不必每次去查，这样可以提高效率；
- **Concurrency**：shiro 支持多线程应用的并发验证，即如在一个线程中开启另一个线程，能把权限自动传播过去；
- **Testing**：提供测试支持
- **Remember Me**：记住我，这个是非常常见的功能，即一次登录后，下次再来的话不用登录了。
- **Run As**：允许一个用户假装为另一个用户（如果他们允许）的身份进行访问；

##### shiro的架构

![img](C:\Users\皿煮国的潜逃败类\Desktop\3.png)

- **Subject**：主体，代表了当前 “用户”,所有 Subject 都绑定到 SecurityManager，与 Subject 的所有交互都会委托给 SecurityManager
- **SecurityManager**：安全管理器；即所有与安全有关的操作都会与 SecurityManager 交互；且它管理着所有 Subject,你可以把它看成 DispatcherServlet 前端控制器；
- **Realm**:那么它需要从 Realm 获取相应的用户进行比较以确定用户身份是否合法，可以把 Realm 看成 DataSource，即安全数据源。
- **Authenticator**：认证器，负责主体认证的
- **Authrizer**：授权器，或者访问控制器，用来决定主体是否有权限进行相应的操作
- **SessionDAO**：DAO 大家都用过，数据访问对象，用于会话的 CRUD
- **CacheManager**：缓存控制器，来管理如用户、角色、权限等的缓存的；因为这些数据基本上很少去改变，放到缓存中后可以提高访问的性能
- **Cryptography**：密码模块，Shiro 提高了一些常见的加密组件用于如密码加密 / 解密的。



##### 简单使用

在 shiro 中，用户需要提供 principals （身份）和 credentials（证明）给 shiro，从而应用能验证用户身份：

**principals**：身份，即主体的标识属性，可以是任何东西，如用户名、邮箱等，唯一即可。一个主体可以有多个 principals，但只有一个 Primary principals，一般是用户名 / 密码 / 手机号。

**credentials**：证明 / 凭证，即只有主体知道的安全值，如密码 / 数字证书等。

另外两个相关的概念是之前提到的 **Subject** 及 **Realm**，分别是主体及验证主体的数据源。

```ini
# shiro.ini
[users]
zhang=123
wang=123
```

```java
public class testShiro {
    @Test
    public void login(){
        // 1.获取SecurityManger工厂,此处使用Ini配置文件初始化SecurityManger,ini中配置了用户规则或者realm
        Factory<SecurityManager> factory = new IniSecurityManagerFactory("classpath:shiro.ini");
        
        // 2.得到SecurityManger实例，并绑定给SecurityUtils,SecurityManger中有个realms属性，里面保存了所有的用户信息
        SecurityManager securityManager = factory.getInstance();
        SecurityUtils.setSecurityManager(securityManager);

        // 3.得到Subject验证主体,该对象中有个securityManager
        Subject subject = SecurityUtils.getSubject();

        UsernamePasswordToken token = new UsernamePasswordToken("zhang", "123");
        try {
            // 使用验证主体验证对象合法性
            subject.login(token);
        }catch (AuthenticationException e){
            // 身份验证失败
        }
        // 断言用户已经登录
        Assert.assertTrue(subject.isAuthenticated());
    }
}
```

#####  验证流程

1. 调用Subject.login(token)进行登录，其会自动委托给SecurityManger,调用前必须使用SecurityUtils.setSecurityManger()设置
2. **SecurityManger**负责验证逻辑，它会委托给Authenticator进行验证
3. **Authenticator**才是真正的验证这，ShiroApi核心的身份认证入口
4. **Authenticator** 可能会委托给相应的 AuthenticationStrategy 进行多 Realm 身份验证，默认 ModularRealmAuthenticator 会调用 **AuthenticationStrategy** 进行多 Realm 身份验证
5. Authenticator 会把相应的 token 传入 **Realm**，从 Realm 获取身份验证信息，如果没有返回 / 抛出异常表示身份验证失败了。此处可以配置多个 Realm，将按照相应的顺序及策略进行访问。
6. shiro身份验证的本质是将token传入Realm中,Realm中的用户，角色，权限，如丰驰后台管理系统

##### 自定义Realm的实现

```ini
# shiro.ini 声明一个realm
myRealm1=com.kiy.shiro.realm.MyRealm
#指定securityManager的realms实现
securityManager.realms=$myRealm1
```

```java
public class MyRealm implements Realm {
    @Override
    public String getName() {
        return "myrealm1";
    }
    @Override
    public boolean supports(AuthenticationToken token) {
        //仅支持UsernamePasswordToken类型的Token
        return token instanceof UsernamePasswordToken;
    }
    // 用户验证
    @Override
    public AuthenticationInfo getAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        String username = (String)token.getPrincipal();
        String password = new String((char[])token.getCredentials());
        if(!"zhang".equals(username)) {
            throw new UnknownAccountException();
        }
        if(!"123".equals(password)) {
            throw new IncorrectCredentialsException();
        }
        //如果身份认证验证成功，返回一个AuthenticationInfo实现；
        return new SimpleAuthenticationInfo(username, password, getName());
    }
}
```

shiro中默认提供的Realm一般继承自AuthorizingRealm

- **IniRealm** 读取ini文件获取用户组信息
- **PropertiesRealm**  读取properties文件获取用户组信息
- **JdbcRealm** 通过 sql 查询相应的信息,诸如 select password from users where username = ?

##### 多Realm验证策略

SecurityManger有一个默认实现ModularRealmAuthenticator ，可以多realm指定验证规则

**FirstSuccessfulStrategy**  只要有一个 Realm 验证成功即可，只返回第一个 Realm 身份验证成功的认证信息，其他的忽略；

**AtLeastOneSuccessfulStrategy** 只要有一个 Realm 验证成功即可，和 FirstSuccessfulStrategy 不同，返回所有 Realm 身份验证成功的认证信息

**AllSuccessfulStrategy**：所有 Realm 验证成功才算成功，且返回所有 Realm 身份验证成功的认证信息，如果有一个失败就失败了。

丰驰后台管理系统使用的是AtLeastOneSuccessfulStrategy策略

```xml
<!--配置权限核心管理器 -->
<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
    <!-- 缓存管理器 -->
    <property name="cacheManager" ref="cacheManager" />
    <!-- 验证 -->
    <property name="authenticator" ref="authenticator"/>
    <!-- 多个验证策略 realmes -->
    <property name="realms">
        <list>
            <!-- 这个认证，有一个先后的顺序 -->
            <ref bean="myCasRealm"/>
        </list>
    </property>
    <property name="subjectFactory" ref="casSubjectFactory"/>
</bean>

<bean id="authenticator" class="org.apache.shiro.authc.pam.ModularRealmAuthenticator">
    <property name="authenticationStrategy" >
        <!-- 所有Reaml都全部匹配的策略 -->
        <!-- <bean class="org.apache.shiro.authc.pam.AllSuccessfulStrategy"/> -->
        <bean class="org.apache.shiro.authc.pam.AtLeastOneSuccessfulStrategy"/>
    </property>
</bean>
```

##### 授权

```ini
[users]
zhang= 123,role1,role2
wang = 123,role1
[roles]
role1=user:create,user:update
role2=user:create,user:delete
```

```java
login("classpath:shiro2.ini","zhang","123");
Assert.assertTrue(SecurityUtils.getSubject().hasRole("role1"));
Assert.assertTrue(SecurityUtils.getSubject().isPermitted("user:create"))
    
// shiro提供了hasRole,hasRoles,checkRole,checkRoles方法用来判断角色
// shiro提供了isPermitted，isPermittedAll,checkPermissions来鉴权
```

- 调用Subject.isPermitted/hasRole接口，其会委托给SecurityManger，而SecurityManger会委托给Authorizer
- Authorzer是真正的授权者，**此处需要理解的是SecurityManager既继承了Authorzer，又继承了Authenticator**
- 再进行验权前，会调用相应的Realm获取subject相应的角色权限用于匹配传入的角色权限

##### 使用自定义Realm进行授权

```ini
#声明一个realm
myRealm1=com.kiy.shiro.realm.MyRealm
#指定securityManager的realms实现
securityManager.realms=$myRealm1
```

```java
public class MyRealm extends AuthorizingRealm {
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        SimpleAuthorizationInfo authorizationInfo = new SimpleAuthorizationInfo();
        authorizationInfo.addRole("role2");
        authorizationInfo.addStringPermission("update");
        return authorizationInfo;
    }
    ...
}
```

##### 登录授权总结

1. 根据Realm生成一个SecurityManager实例，并将这个SecurityManager实例放到SecurityUtils的subject属性中
2. 调用subject属性的login方法，将会执行realm的doGetAuthenticationInfo用户验证和doGetAuthorizationInfo用户授权，其中login的参数AuthenticationToken token将会传给doGetAuthenticationInfo
3. 此时在拦截器处可以通过SecurityUtils.getSubject获取到当前用户，并且根据Subject.hasRole/isPermitted判断是否属于拥有某个角色或者权限，以此来达到鉴权的目的



##### 基于spring，拦截器，注解的WEB集成

```xml
<!-- web.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    <display-name>Archetype Created Web Application</display-name>
    <!--把applicationContext.xml加入到配置文件中-->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath*:spring.xml</param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!-- 代理拦截器，它将拦截转发给了ioc容器中的shiroFilter-->
    <filter>
        <filter-name>shiroFilter</filter-name>
        <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
        <init-param>
            <param-name>targetFilterLifecycle</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>shiroFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    <!--配置springmvc-->
    <servlet>
        <servlet-name>springMVC</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <!--配置spring.xml作为mvc的配置文件-->
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath*:spring.xml</param-value>
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

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:util="http://www.springframework.org/schema/util"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context-3.0.xsd http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd">
    <context:component-scan base-package="com.kiy" />


    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/pages/"/>
        <property name="suffix" value=".jsp"/>
    </bean>

    <mvc:annotation-driven />

    <mvc:resources mapping="/javascript/**" location="/assets/js/"/>
    <mvc:resources mapping="/styles/**" location="/assets/css/"/>
    <mvc:resources mapping="/html/**" location="/assets/html/"/>


    <bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
        <property name="realm" ref="userRealm"/>
    </bean>

    <bean class="org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor">
        <property name="securityManager" ref="securityManager"/>
    </bean>
    <bean id="lifecycleBeanPostProcessor" class="org.apache.shiro.spring.LifecycleBeanPostProcessor" />
    <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"
          depends-on="lifecycleBeanPostProcessor">
        <property name="proxyTargetClass" value="true" />
    </bean>


    <bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
        <property name="securityManager" ref="securityManager"/>
        <property name="loginUrl" value="/login"/>
        <property name="unauthorizedUrl" value="/refuse.html"/>
        <!-- 
        	使用配置形式的路由拦截规则，可对批量或对一些无法添加注解的url进行权限拦截
		 -->
        <property name="filters">
            <util:map>
                <entry key="permission" value-ref="permission"/>
                <entry key="logout" value-ref="logout"/>                
            </util:map>
        </property>
        <property name="filterChainDefinitions">
            <value>
                <!-- 登录拦截器 -->
                /logout = logout
                <!-- 所有url都需要登录 -->
                /** = authc
            </value>
        </property>
        <!-- 
			请注意，shiro的默认验权跳转位置为上一次访问位置，所以大多数的时候successUrl是不起作用的，
			它只有在session缓存中中没有用户请求地址时后才会生效
		-->
        <property name="successUrl" value="/test"/>
    </bean>
	<!-- 自定义验证和授权Realm -->
    <bean id="userRealm" class="com.kiy.realm.UserRealm"/>
    <!-- 自定义拦截器 -->
    <bean id="permission" class="com.kiy.filter.PermissionFilter"/>
    <bean id="logout" class="com.kiy.filter.CustomerLogoutFilter"/>
</beans>
```

```java
package com.kiy.realm;

import org.apache.shiro.authc.*;
import org.apache.shiro.authz.AuthorizationInfo;
import org.apache.shiro.authz.SimpleAuthorizationInfo;
import org.apache.shiro.realm.AuthorizingRealm;
import org.apache.shiro.subject.PrincipalCollection;

import java.util.HashSet;
import java.util.Set;

/**
 * @title:
 * @author: 80002834
 * @create: 2020-08-14 16:05:58
 **/
public class UserRealm  extends AuthorizingRealm {
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();
        Set<String> strings = new HashSet<>();
        strings.add("/test");
        simpleAuthorizationInfo.setStringPermissions(strings);
        return simpleAuthorizationInfo;
    }

    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
        String username = (String)authenticationToken.getPrincipal();
        String password = new String((char[])authenticationToken.getCredentials());
        if(!"lisi".equals(username)) {
            throw new UnknownAccountException();
        }
        if(!"123".equals(password)) {
            throw new IncorrectCredentialsException();
        }
        //如果身份认证验证成功，返回一个AuthenticationInfo实现；
        return new SimpleAuthenticationInfo(username, password, getName());
    }
}
```

```java
package com.kiy.filter;

import org.apache.shiro.subject.Subject;
import org.apache.shiro.util.StringUtils;
import org.apache.shiro.web.filter.AccessControlFilter;
import org.apache.shiro.web.util.WebUtils;

import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.HashMap;
import java.util.Map;

/**
 * @title:
 * @author: 80002834
 * @create: 2020-08-14 17:49:44
 **/
public class PermissionFilter extends AccessControlFilter {
    @Override
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
        //先判断带参数的权限判断
        Subject subject = getSubject(request, response);
        if(null != mappedValue){
            String[] arra = (String[])mappedValue;
            for (String permission : arra) {
                if(subject.isPermitted(permission)){
                    return Boolean.TRUE;
                }
            }
        }
        HttpServletRequest httpRequest = ((HttpServletRequest)request);

        String uri = httpRequest.getRequestURI();//获取URI
        String basePath = httpRequest.getContextPath();//获取basePath
        if(null != uri && uri.startsWith(basePath)){
            uri = uri.replaceFirst(basePath, "");
        }
        if(subject.isPermitted(uri)){
            return Boolean.TRUE;
        }
        if(ShiroFilterUtils.isAjax(request)){
            Map<String,String> resultMap = new HashMap<String, String>();
            resultMap.put("login_status", "401");
            resultMap.put("message", "权限不足");//！
            ShiroFilterUtils.out(response, resultMap);
        }
        return Boolean.FALSE;
    }

    @Override
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
        Subject subject = getSubject(request, response);
        if (null == subject.getPrincipal()) {//表示没有登录，重定向到登录页面

            saveRequest(request);
            WebUtils.issueRedirect(request, response, ShiroFilterUtils.LOGIN_URL);
        } else {
            if (StringUtils.hasText(ShiroFilterUtils.UNAUTHORIZED)) {//如果有未授权页面跳转过去
                WebUtils.issueRedirect(request, response, ShiroFilterUtils.UNAUTHORIZED);
            } else {//否则返回401未授权状态码
                WebUtils.toHttp(response).sendError(HttpServletResponse.SC_UNAUTHORIZED);
            }
        }
        return Boolean.FALSE;
    }
}
```

```java
package com.kiy.filter;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;

import org.apache.shiro.session.SessionException;
import org.apache.shiro.subject.Subject;
import org.apache.shiro.web.filter.authc.LogoutFilter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 *
 * @author 01369611
 * @author 80002826
 *
 */
public class CustomerLogoutFilter extends LogoutFilter {
    private static final Logger logger = LoggerFactory.getLogger(CustomerLogoutFilter.class);

    public CustomerLogoutFilter(){super();}

    @Override
    protected boolean preHandle(ServletRequest request,ServletResponse response) throws Exception{
        Subject subject = getSubject(request, response);
        String redirectUrl = getRedirectUrl(request, response, subject);
        try{
            subject.logout();
        }catch(SessionException ise){
            logger.debug("登出时session处理出错",ise);
        }
        issueRedirect(request, response, redirectUrl);
        return false;
    }
}

```

```java
package com.kiy.filter;

import net.sf.json.JSONObject;
import org.springframework.beans.factory.annotation.Value;

import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.io.PrintWriter;
import java.util.Map;

/**
 *
 */

public class ShiroFilterUtils {

    public static String LOGIN_URL = "login";

    public static String UNAUTHORIZED = "/html/refuse.html";

    /**
     * 是否是Ajax请求
     * @param request
     * @return
     */
    public static boolean isAjax(ServletRequest request){
        return "XMLHttpRequest".equalsIgnoreCase(((HttpServletRequest) request).getHeader("X-Requested-With"));
    }

    /**
     * response 输出JSON
     * @param response
     * @param resultMap
     * @throws IOException
     */
    public static void out(ServletResponse response, Map<String, String> resultMap){

        PrintWriter out = null;
        try {
            response.setCharacterEncoding("UTF-8");
            out = response.getWriter();
            out.println(JSONObject.fromObject(resultMap).toString());
        } catch (Exception e) {
            System.out.println(e);
        }finally{
            if(null != out){
                out.flush();
                out.close();
            }
        }
    }
}
```

```java
/**
 *  测试方式
 */
 package com.kiy.controller;

import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authz.annotation.Logical;
import org.apache.shiro.authz.annotation.RequiresPermissions;
import org.apache.shiro.authz.annotation.RequiresRoles;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;

/**
 * @title:
 * @author: 80002834
 * @create: 2020-08-12 16:46:42
 **/
@Controller
public class TestController {
    @RequestMapping(value = "test",method = RequestMethod.GET)
    @ResponseBody
    public String test(){
        System.out.println(SecurityUtils.getSubject().getSession());
        return "this";
    }

    @RequestMapping("login")
    public String login(HttpServletRequest request, Model mv) {
        String e = (String) request.getAttribute("shiroLoginFailure");
        if (e != null) {
            if (e.contains("org.apache.shiro.authc.UnknownAccountException")) {
                mv.addAttribute("msg", "账号不存在");
            } else if (e.contains("org.apache.shiro.authc.IncorrectCredentialsException")) {
                mv.addAttribute("msg", "密码错误");
            }
        }
        return "login";
    }
}
```

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>
<html>
<head>
    <title>Title</title>

    <style type="text/css">
        .p
        {
            color:red;
            font-size:16px;
        }
    </style>
</head>
<body>
<c:if test="${msg != null}">
    <p class="p">${msg}</p >
</c:if>
<form action="${pageContext.request.contextPath}/login" method="post">
    <table>
        <tr>
            <td>用户名：</td>
            <td><input type="text" name="username"></td>
        </tr>
        <tr>
            <td>密码：</td>
            <td><input type="password" name="password"></td>
        </tr>
    </table>
    <input type="submit" value="登录">
</form>
</body>
</html>
```

##### 会话

```java
login("classpath:shiro.ini","zhang","123");
Subject subject = SecurityUtils.getSubject();
Session session = subject.getSession();
session.getId();	// 获取当前会话的唯一标识
session.getHost(); // 获取当前会话的主机地址,该地址通过HostAuthenticationToken.getHost()提供
session.getTimeout(); // 获取当前会话的的过期时间
session.setTimeout();	// 设置当前会话的过期时间
session.getStartTimestamp(); // 设置当前会话启动时间
session.getLastAccessTime(); // 获取当前会话最后访问时间
session.touch(); // 更新最后访问时间，如果是web应用，shiroFilter会自动更新
session.stop(); // 销毁会话,Subject.logout会自动调用
session.setAttribute("key","value"); // 设置session字段
session.getAttribute("key"); // 获取session字段
session.removeAttribute("key"); // 删除字段
```

##### 会话管理器

```bash
# SecurityManager 集成了SessionManger，提供了如下接口
Session start(SessionContext context);
Session getSession(SessionKey key);

# shiro默认提供了三种session管理器
DefaultSessionManager:DefaultSecurityManager 使用的默认实现，用于 JavaSE 环境
ServletContainerSessionManager:DefaultWebSecurityManager 使用的默认实现，用于 Web 环境，其直接使用 Servlet 容器的会话，Session 的超时依赖于底层 Servlet 容器的超时时间，可以在 web.xml 中配置其会话的超时时间（分钟为单位）：
DefaultWebSessionManager:用于 Web 环境的实现，可以替代 ServletContainerSessionManager，自己维护着会话，直接废弃了 Servlet 容器的会话管理。

# 默认情况下session中只存在了三条数据
登录状态：org.apache.shiro.subject.support.DefaultSubjectContext_AUTHENTICATED_SESSION_KEY:true
用户ip地址：org.apache.shiro.web.session.HttpServletSession.HOST_SESSION_KEY:0:0:0:0:0:0:0:1
用户标识(名称):org.apache.shiro.subject.support.DefaultSubjectContext_PRINCIPALS_SESSION_KEY:lisi
```

```ini
# ini文件中控制Cookie的生成
#在 Servlet 容器中，默认使用 JSESSIONID Cookie 维护会话，且会话默认是跟容器绑定的；在某些情况下可能需要使用自己的#会话机制，此时我们可以使用 DefaultWebSessionManager 来维护会话：
sessionIdCookie 是 sessionManager 创建会话 Cookie 的模板：
sessionIdCookie.name：设置 Cookie 名字，默认为 JSESSIONID；
sessionIdCookie.domain：设置 Cookie 的域名，默认空，即当前访问的域名；
sessionIdCookie.path：设置 Cookie 的路径，默认空，即存储在域名根下；
sessionIdCookie.maxAge：设置 Cookie 的过期时间，秒为单位，默认 - 1 表示关闭浏览器时过期 Cookie；
sessionIdCookie.httpOnly：如果设置为 true，则客户端不会暴露给客户端脚本代码，使用 HttpOnly cookie 有助于减少某些类型的跨站点脚本攻击；此特性需要实现了 Servlet 2.5 MR6 及以上版本的规范的 Servlet 容器支持；
sessionManager.sessionIdCookieEnabled：是否启用 / 禁用 Session Id Cookie，默认是启用的；如果禁用后将不会设置 Session Id Cookie，即默认使用了 Servlet 容器的 JSESSIONID，且通过 URL 重写（URL 中的 “;JSESSIONID=id” 部分）保存 Session Id。
```

##### 会话监听

```
sessionListener1=com.github.zhangkaitao.shiro.chapter10.web.listener.MySessionListener1
sessionListener2=com.github.zhangkaitao.shiro.chapter10.web.listener.MySessionListener2
sessionManager.sessionListeners=$sessionListener1,$sessionListener2
```

```java
public class MySessionListener1 implements SessionListener {
    @Override
    public void onStart(Session session) {//会话创建时触发
        System.out.println("会话创建：" + session.getId());
    }
    @Override
    public void onExpiration(Session session) {//会话过期时触发
        System.out.println("会话过期：" + session.getId());
    }
    @Override
    public void onStop(Session session) {//退出/会话过期时触发
        System.out.println("会话停止：" + session.getId());
    }  
}
```



##### 缓存

**权限信息存放在数据库中时，对于每次前端的访问请求都需要进行一次数据库查询**。特别是在大量使用shiro的jsp标签的场景下，对应前端的一个页面访问请求会同时出现很多的权限查询操作，这对于权限信息变化不是很频繁的场景，每次前端页面访问都进行大量的权限数据库查询是非常不经济的。因此，非常有必要对权限数据使用缓存方案。

```
缓存解决的问题就是doGetAuthorizationInfo多次调用的问题
```

















