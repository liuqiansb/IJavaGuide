##### Spring通知类型

| 名称                                                        | 说明                                                         |
| ----------------------------------------------------------- | ------------------------------------------------------------ |
| org.springframework.aop.MethodBeforeAdvice（前置通知）      | 在方法之前自动执行的通知称为前置通知，可以应用于权限管理等功能。 |
| org.springframework.aop.AfterReturningAdvice（后置通知）    | 在方法之后自动执行的通知称为后置通知，可以应用于关闭流、上传文件、删除临时文件等功能。 |
| org.aopalliance.intercept.MethodInterceptor（环绕通知）     | 在方法前后自动执行的通知称为环绕通知，可以应用于日志、事务管理等功能 |
| org.springframework.aop.ThrowsAdvice（异常通知）            | 在方法抛出异常时自动执行的通知称为异常通知，可以应用于处理异常记录日志等功能。 |
| org.springframework.aop.IntroductionInterceptor（引介通知） | 在目标类中添加一些新的方法和属性，可以应用于修改旧版本程序（增强类）。 |

##### 声明式Spring AOP配置属性

| 属性名称         | 描述                                                         |
| ---------------- | ------------------------------------------------------------ |
| target           | 代理的目标对象                                               |
| proxyInterfaces  | 代理要实现的接口，如果有多个接口，则可以使用以下格式赋值： <list>   <value ></value>   ... </list> |
| proxyTargetClass | 是否对类代理而不是接口，设置为 true 时，使用 CGLIB 代理      |
| interceptorNames | 需要植入目标的 Advice                                        |
| singleton        | 返回的代理是否为单例，默认为 true（返回单实例）              |
| optimize         | 当设置为 true 时，强制使用 CGLIB                             |

##### AOP添加通知

```java
/**
 * 环绕通知，会拦截执行方法的为interceptor
 */
public class IMethodInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("前置通知");
        Object proceed = invocation.proceed();
        System.out.println("后置通知");
        return proceed;
    }
}
/**
 * return后通知,不会拦截执行方法的为advice
 */
public class IAfterReturnAdvice implements AfterReturningAdvice {
    @Override
    public void afterReturning(Object returnValue, Method method, Object[] args, Object target) throws Throwable {
        System.out.println("this is after return advice");
    }
}
```



```xml
<bean id="userService" class="com.demo.proxy.service.impl.UserServiceImpl"/>
<bean id="iIMethodInterceptor" class="com.demo.proxy.aspect.IMethodInterceptor"/>
<bean id="iAfterReturnAdvice" class="com.demo.proxy.aspect.IAfterReturnAdvice"/>

<bean id="userServiceMethodInterceptorProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
       <property name="proxyInterfaces" value="com.demo.proxy.service.UserService"/>
       <property name="target" ref="userService"/>
       <property name="interceptorNames" value="iIMethodInterceptor"/>
       <property name="proxyTargetClass" value="true"/>
</bean>
<bean id="userServiceAfterReturnAdvice" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="proxyInterfaces" value="com.demo.proxy.service.UserService"/>
        <property name="target" ref="userService"/>
        <property name="interceptorNames" value="iAfterReturnAdvice"/>
        <property name="proxyTargetClass" value="true"/>
</bean>
<bean id="userServiceAdvices" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="proxyInterfaces" value="com.demo.proxy.service.UserService"/>
    <property name="target" ref="userService"/>
    <property name="interceptorNames">
        <list>
            <value>iAfterReturnAdvice</value>
            <value>iIMethodInterceptor</value>
        </list>
    </property>
    <property name="proxyTargetClass" value="true"/>
</bean>
```

##### 使用AspectJ来配置通知

- spring-aop：AOP核心功能，例如代理工厂等等

- aspectjweaver：简单理解，支持切入点表达式等等

- aspectjrt：简单理解，支持aop相关注解等等

```txt
xml形式的使用aspectJ和spring-aop实现正则校验拦截，即可以实现批量拦截和通知
```

```java
public class LogAspect {
    // 前置通知
    public void before(JoinPoint joinPoint) {
        System.out.print("前置通知，目标：");
        System.out.print(joinPoint.getTarget() + "方法名称:");
        System.out.println(joinPoint.getSignature().getName());
    }
    // 后置通知
    public void afterReturning(JoinPoint joinPoint,Object returnVal) {
        System.out.print("后置通知，方法名称：" + joinPoint.getSignature().getName());
    }
    // 环绕通知
    public Object around(ProceedingJoinPoint proceedingJoinPoint)
            throws Throwable {
        System.out.println("环绕开始"); // 开始
        Object obj = proceedingJoinPoint.proceed(); // 执行当前目标方法
        System.out.println("环绕结束"); // 结束
        return obj;
    }
    // 异常通知
    public void afterThrowing(JoinPoint joinPoint, Throwable e) {
        System.out.println("异常通知" + "出错了" + e.getMessage());
    }
    // 最终通知
    public void after() {
        System.out.println("最终通知");
    }
}
```

```xml
<bean id="logAspect" class="com.demo.proxy.aspect.LogAspect"/>
<aop:config>
    <aop:aspect ref="logAspect">
        <!-- 配置切入点，通知最后增强哪些方法 -->
        <aop:pointcut expression="execution ( * com.demo.proxy.service.*.* (..))"
                      id="pointCut" />
        <!--前置通知，关联通知 Advice和切入点PointCut -->
        <aop:before method="before" pointcut-ref="pointCut"/>
        <!--后置通知，在方法返回之后执行，就可以获得返回值returning 属性 -->
        <!--* returning属性用于设置通知第二个参数的名称，即返回值，类型为Object-->
        <aop:after-returning method="afterReturning"
                             pointcut-ref="pointCut" returning="returnVal" />
        <!--环绕通知 -->
        <aop:around method="around" pointcut-ref="pointCut" />
        <!--抛出通知：用于处理程序发生异常，可以接收当前方法产生的异常 -->
        <!-- *注意：如果程序没有异常，则不会执行增强 -->
        <!-- * throwing属性：用于设置通知第二个参数的名称，类型Throwable -->
        <aop:after-throwing method="afterThrowing"
                            pointcut-ref="pointCut" throwing="e" />
        <!--最终通知：无论程序发生任何事情，都将执行 -->
        <aop:after method="after" pointcut-ref="pointCut" />
    </aop:aspect>
</aop:config>
```

##### aop的annotation实现

> 使用前需要开启注解扫描

```xml
<context:component-scan base-package="com.demo.proxy"/>
<!-- 使切面开启自动代理 -->
<aop:aspectj-autoproxy></aop:aspectj-autoproxy>
```



| 名称            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| @Aspect         | 用于定义一个切面。                                           |
| @Before         | 用于定义前置通知，相当于 BeforeAdvice。                      |
| @AfterReturning | 用于定义后置通知，相当于 AfterReturningAdvice。              |
| @Around         | 用于定义环绕通知，相当于MethodInterceptor。                  |
| @AfterThrowing  | 用于定义抛出通知，相当于ThrowAdvice。                        |
| @After          | 用于定义最终final通知，不管是否异常，该通知都会执行。        |
| @DeclareParents | 用于定义引介通知，相当于IntroductionInterceptor（不要求掌握）。 |

> 相当于将aop:config 和 切面类一起写了

```java
@Aspect
@Component
public class LogAnnotationAop {
    @Pointcut("execution(* com.demo.proxy.service.*.* (..))")
    public void pointcut(){}

    @Before(value = "pointcut()")
    public void before(JoinPoint joinPoint) {
        System.out.print("前置通知，目标：");
        System.out.print(joinPoint.getTarget() + "方法名称:");
        System.out.println(joinPoint.getSignature().getName());
    }
    @AfterReturning(value = "pointcut()",returning = "result")
    public void afterReturning(JoinPoint joinPoint,Object result) {
        System.out.print("后置通知，方法名称：" + joinPoint.getSignature().getName());
    }
    @Around(value = "pointcut()")
    public Object around(ProceedingJoinPoint proceedingJoinPoint)
            throws Throwable {
        System.out.println("环绕开始"); // 开始
        Object obj = proceedingJoinPoint.proceed(); // 执行当前目标方法
        System.out.println("环绕结束"); // 结束
        return obj;
    }
    @After(value = "pointcut()")
    public void after() {
        System.out.println("最终通知");
    }
    @AfterThrowing(value = "pointcut()",throwing = "e")
    public void afterThrowing(JoinPoint joinPoint, Throwable e) {
        System.out.println("异常通知" + "出错了" + e.getMessage());
    }
}
```

##### execution表达式解析

| 符号                        | 含义                                                     |
| --------------------------- | -------------------------------------------------------- |
| **execution（） **          | **表达式的主体；**                                       |
| **第一个”\*“符号 **         | **表示返回值的类型任意；**                               |
| **com.sample.service.impl** | **AOP所切的服务的包名，即，我们的业务部分**              |
| **包名后面的”..“**          | **表示当前包及子包**                                     |
| **第二个”\*“**              | **表示类名，\*即所有类。此处可以自定义，下文有举例**     |
| **.\*(..)**                 | **表示任何方法名，括号表示参数，两个点表示任何参数类型** |

##### spring的JDBCTemplate类

```

```

##### spring实现事务的核心接口

> 总结来说就是事务操作，事务定义，事务管理

1. PlatformTransactionManager接口是 Spring 提供的平台事务管理器，用于管理事务。该接口中提供了三个事务操作方法，具体如下。

- TransactionStatus getTransaction（TransactionDefinition definition）：用于获取事务状态信息。
- void commit（TransactionStatus status）：用于提交事务。
- void rollback（TransactionStatus status）：用于回滚事务。



2. TransactionDefinition接口是事务定义（描述）的对象，它提供了事务相关信息获取的方法，其中包括五个操作，具体如下。

- String getName()：获取事务对象名称。
- int getIsolationLevel()：获取事务的隔离级别。
- int getPropagationBehavior()：获取事务的传播行为。
- int getTimeout()：获取事务的超时时间。
- boolean isReadOnly()：获取事务是否只读。

| 属性名称                  | 值            | 描 述                                                        |
| ------------------------- | ------------- | ------------------------------------------------------------ |
| PROPAGATION_REQUIRED      | required      | 支持当前事务。如果 A 方法已经在事务中，则 B 事务将直接使用。否则将创建新事务 |
| PROPAGATION_SUPPORTS      | supports      | 支持当前事务。如果 A 方法已经在事务中，则 B 事务将直接使用。否则将以非事务状态执行 |
| PROPAGATION_MANDATORY     | mandatory     | 支持当前事务。如果 A 方法没有事务，则抛出异常                |
| PROPAGATION_REQUIRES_NEW  | requires_new  | 将创建新的事务，如果 A 方法已经在事务中，则将 A 事务挂起     |
| PROPAGATION_NOT_SUPPORTED | not_supported | 不支持当前事务，总是以非事务状态执行。如果 A 方法已经在事务中，则将其挂起 |
| PROPAGATION_NEVER         | never         | 不支持当前事务，如果 A 方法在事务中，则抛出异常              |
| PROPAGATION.NESTED        | nested        | 嵌套事务，底层将使用 Savepoint 形成嵌套事务                  |



​	3.TransactionStatus接口是事务的状态，它描述了某一时间点上事务的状态信息。

| 名称                       | 说明               |
| -------------------------- | ------------------ |
| void flush()               | 刷新事务           |
| boolean hasSavepoint()     | 获取是否存在保存点 |
| boolean isCompleted()      | 获取事务是否完成   |
| boolean isNewTransaction() | 获取是否是新事务   |
| boolean isRollbackOnly()   | 获取是否回滚       |
| void setRollbackOnly()     | 设置事务回滚       |

##### 基于xml的事务管理配置



