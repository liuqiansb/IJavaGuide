##### java原生的RMI

1.服务端代码

```java
public class ServerMain {
    public static void main(String[] args) throws RemoteException, AlreadyBoundException, MalformedURLException {
        // 1.启动RMI注册服务，指定端口号
        LocateRegistry.createRegistry(8888);
        // 2.创建要被访问的远程对象实例
        UserService userService = new UserServiceImpl();
        // 3.把远程对象实例注册到RMI注册服务器上
        Naming.bind("rmi://localhost:8888/UserService",userService);
        System.out.println("服务端启动中！");
    }
}
```

```java
public interface UserService extends Remote {
    String sayHello(String name) throws RemoteException;
}
```

```java
public class UserServiceImpl extends UnicastRemoteObject implements UserService{
    public UserServiceImpl() throws RemoteException {
    }
    public String sayHello(String name) throws RemoteException{
        return name+"传过来了!";
    }
}
```

2.客户端代码

```java
public class ClientMain {
    public static void main(String[] args) throws RemoteException, NotBoundException, MalformedURLException {
       UserService userService = (UserService) Naming.lookup("rmi://localhost:8888/UserService");
        String s = userService.sayHello("this");
        System.out.println(s); 
    }
}
```

##### hessian

```java
	// 使用tomcat插件启动服务
	<packaging>war</packaging>
    <dependencies>
        <dependency>
            <groupId>com.caucho</groupId>
            <artifactId>hessian</artifactId>
            <version>4.0.7</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.tomcat.maven</groupId>
                <artifactId>tomcat7-maven-plugin</artifactId>
                <version>2.2</version>
                <configuration>
                    <port>8888</port>
                    <path>/</path>
                    <uriEncoding>UTF-8</uriEncoding>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

```java
// 配置一个servlet服务
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <servlet>
        <servlet-name>HessianServlet</servlet-name>
        <servlet-class>com.caucho.hessian.server.HessianServlet</servlet-class>
        <init-param>
            <param-name>service-class</param-name>
            <param-value>com.itheima.service.impl.UserServiceImpl</param-value>
        </init-param>
    </servlet>
    <servlet-mapping>
        <servlet-name>HessianServlet</servlet-name>
        <url-pattern>/hessianServlet</url-pattern>
    </servlet-mapping>
</web-app>
```

```java
// 调用处
public class ClientMain {
    public static void main(String[] args) throws MalformedURLException {
        String url = "http://localhost:8888/hessianServlet";
        HessianProxyFactory hessianProxyFactory = new HessianProxyFactory();
        UserService api = (UserService) hessianProxyFactory.create(UserService.class, url);
        System.out.println(api.sayHello("你好"));
    }
}
```

##### Dubbo

1. 集群容错

   1. 服务路由

      服务路由包含一条路由规则，路由规则决定服务消费者的调用目标，规定服务消费者可以调用哪些服务提供者，dubbo提供三种服务路由实现，分别为条件路由，脚本路由，标签路由

      - 条件路由规则格式

        [服务消费者匹配条件]=>[服务提供者匹配条件]

        host=10.29.13.143=>host=10.29.13.142   表示143这台记起只能调用142这台机器

      - 常见路由配置

        1. 白名单：除了10，11的消费者都不处理

           host!=10.20.153.10，10.20.153.11=>

        2. 黑名单：10，11不处理

           host = 10.20.153.10，10.20.153.11=>

        3. 读写分离(常用，比如读机器多)

           method=find,list,get,is=>host:10.20.153.10

           method!=find,list,get,is=>host:10.20.153.11

        4. 前后台分离

           application=front=>host:10.20.153.10

           application=!front=>host:10.20.153.10

   2. 集群容错

      - 重试：<dubbo:service retries="2"> 失败重试(重试其他机器，带来更长的延迟) 
      - 快速失败：只发起一次操作，失败立即报错
      - 失败安全：出现异常忽略，常用写入日志
      - 失败自动恢复：记录失败，定时重发，用于消息通知，不用马上有返回值的那种
      - 并行调用：并行调用多个服务器，用于实时性较高的读，一定不能用于写

   3. 负载均衡

      - 按照权重设置随机概率
      - 轮询，有状态
      - 一致性hash
      - 最少活跃随机数，方法纬度统计服务调用数

2. 服务治理

   1. 服务降级

      使用dubbo在进行服务调用时，可能各种原因（宕机，网络超时等）,调用出现RpxException

      此时可以使用默认值

      ```java
       // 对服务的所有的方法进行同样的降级操作 return null
       @Reference(mock = "return null",check = false,timeout = 2000)
       private HelloService helloService;
      ```
   
      ```java
      // 定制化服务降级
       @Reference(mock = "true",check = false,timeout = 2000)
       private HelloService helloService;
      
      // 客户端替代方案的接口实现,当使用mock="true"时，dubbo会自动实例化一个HelloServiceMock的类
      public class HelloServiceMock implements HelloService{
          public String sayHello() {
              return "我将默认执行";
          }
      }
      
      
      ```
   
      
   
   2. 整合hystrix
   
      Hystrix旨在通过控制那些远程访问系统，服务和第三方库的几点，从而对延迟和故障提供更加强大的容错能力，Hystrix具备拥有回退机制和断路器功能的线程和信号隔离，请求缓存和请求打包，以及监控和配置等功能，SpringBoot提供了Hystrix的继承
   
      ```
      自己写Spring测试
      ```
   
      



