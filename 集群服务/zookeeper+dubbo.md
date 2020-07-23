##### zookeeper数据一致性

强一致性：如果没有完成数据同步，不对外提供服务，借助锁机制

最终一致性：不考虑即时性

**CAP原则：**

1. 一致性(Consystency)

   强一致性

2. 可用性(Availability)

   提供的服务一直处于可用状态

3. 分区容错性(Partition tolerance)

   在分布式系统中任何网络分区故障的时候，仍需要能够对外提供一致性和可用性服务

注意：对于分布式系统中不可能同时满足一致性，可用性，分区容错性，因为一致性和可用性本就是互相矛盾的，所以要么满足AP模型，要么满足CP模型

**ACID特性:一致性协议**

1. **2pc二阶段提交**

   ![未命名文件](D:\下载\未命名文件.png)

   - **阶段一：提交事务请求**

   1. 协调者向所有的参与者发送事务内容，询问是否可以执行事务操作，并等待反馈

   2. 各个参与者节点执行事务结果

   3. 各参与者反馈给协调者，事务是否可以执行

   - **阶段二：事务提交**

   1. 协调者向各参与者发送commit请求
   2. 参与者提交事务
   3. 参与者向协调者提交commit提交成功确认消息
   4. 协调者接受各个参与者节点的ack后，完成事务commit

   - **中断事务**

   1. 发送回滚请求
   2. 各个参与者节点回滚事务
   3. 反馈协调者事务回滚结果
   4. 协调者接受各个参与者节点ack后回滚事务

   - **二阶段提交存在的问题**

   1. 同步阻塞：协调者在发送请求给各个参与者之后，该协调者处于阻塞状态
   2. 单点问题：协调者挂掉，整个都会挂掉
   3. 脑裂导致数据不一致：即某个参与者因为网络连接不到协调者则会自动进行选举，此时协调者再发送请求该参与者将接收不到导致数据混乱

2. **三阶段提交**

   1. 解决了同步阻塞，引入了超时时间
   2. 长时间接收不到协调者的消息，参与者将会把事务提交，解决了单点问题
   3. 无法解决网络问题

3. paxos算法

   1. 目前解决分布式一致性问题最有效算法

   2. 在分布式系统中如果产生了宕机或者网络异常情况，快速正确的在集群内部对于某个数据达成一致性，并且不管发生任何异常都不会破坏整体一致性

   3. 重要概念：少数服从多数

   4. 算法解析

      一脸懵逼，下回再看。。。

4. ZAB协议

   1. zookeeper使用的是zap协议，因为paxos太难了
   2. 先提案，半数通过，leader执行，然后通知所有follower同步数据

5. 恢复模式

   1. 当服务启动或者leader奔溃之后，将选举leader，完成新leader和其他机器和数据同步

6. 广播模式

   1. 一旦leader和多数follower进行状态同步之后，进入广播模式，如果有新的机器加入，将从leader中进行数据同步

##### zookeeper环境搭建



##### 基本使用



##### 源码解析



##### 应用场景

1. 分布式配置中心

   将配置信息存放在zk中的一个节点中，同时给该节点注册一个数据节点变更的watcher监听，一旦节点发生改变，所有订阅该节点的客户端都可以获取到数据变更的通知

2. 负载均衡

   问题：如果服务器列表产生了更新，如何让反向代理服务器知道，从而更新负载均衡算法

   解决：建立监听servers子节点的状态，在每个服务器启动时，在servicer节点下建立具体服务器的子节点

3. 统一命名服务

4. DNS服务

5. 集群管理

6. 分布式锁

7. 分布式队列

##### dubbo

1. RPC核心
   1. 常见的rpc框架
   
      RMI:
   
      1. javay原生支持的远程调用，使用JRMP作为通信协议，纯java版本的分布式远程调用方案![RMI](C:\Users\皿煮国的潜逃败类\Desktop\RMI.png)
   
      2. RMI步骤
         - 创建远程接口，并且继承java.rmi.Remote接口
         - 实现远程接口，并继承UnlcasRemoteObject
         - 创建服务器程序:createRegistry()方法注册远程对象
         - 创建客户端程序获取注册信息，调用接口方法
   
      Hessian:
   
      Dubbo:
   
   2. 协议
   
   3. 序列化
      - JDK Serializable
      - JSON序列化
      - FST
      - Thrift
      - Hessian
      - Protocal Buffers
   
   4. IO模型
      - BIO
      
        每个连接都需要一个线程来维护,同步阻塞式IO，每个线程上都有一个while，不断监测是否有数据可以读
      
     缺点：每个连接大部分时间都是没有数据可以读的，造成了线程的浪费
      
      - NIO
      
        dubbo选择的IO，非阻塞式IO，每个连接都让同一个线程去处理，该线程上有一个while监听是否有数据，然后依次处理所有连接
      
        负载问题：轮询方式使用多个连接
      
      - IO多路复用
      
      - AIO异步IO
      
   5. 手写简单的rpc框架
2. dubbo高可用
   1. dubbo容错机制
   2. dubbo负载均衡策略与自定义实现
   3. dubbo服务降级
   4. 服务暴露
   5. 多版本控制
   6. 多注册中心
3. dubbo原理
   1. 协议
   2. 序列化
   3. Netty
4. dubbo源码解析
   1. dubbo的SPI

      SPI全程Service Provider Interface 服务发现机制，SPI的本质将接口实现类的全限定名配置在文件中，并通过服务加载器读取配置文件，加载实现类，这样可以在运行时动态为接口替换实现类

      ```java
      // java origin spi
      public interface Student{
      	void service();
      }
      public class StudentImplA implements Student{
          public void service(){
              System.out.println("StudentImplA");
          }
      }
      public class StudentImplB implement Student{
          public void service(){
              System.out.println("StudentImplB");
          }
      }
      
      // create file and input the following content in resources/META-INF/services/com.itheima.Student
      // notice:the file path&name is certain,because of SericeLoader has a prop which value is META-INF/services
      // notice:if you just write one , the iterator create by ServiceLoader has one element    
      
      com.itheima.impl.StudentImplA
      com.itheima.impl.StudentImplB  
      
      @Test
      public void test(){
          ServiceLoader<Student> load = ServiceLoader.load(Student.class);
          Iterator<Student> iterator = load.iterator();
          while (iterator.hasNext()){
               Student studentImpl = iterator.next();
               studentImpl.service();
           }
      }
      // StudentImplA
      // StudentImplB
      ```

      java SPI的问题：

      	1. 所有配置的实现类都将被实例化
       	2. 无法根据参数来获取对应的实现类

   2. dubbo配置文件加载解析

      dubbo使用的是键值对的形式来配置，所以可以根据参数来获取对应的实现类实例

      ```java
      注意：Dubbo spi配置文件地址迁移到META-INF/dubbo下
      StudentImplA=com.itheima.impl.StudentImplA
      StudentImplB=com.itheima.impl.StudentImplB
      ```

      ```java
      ExtensionLoader<Student> extensionLoader = ExtensionLoader.getExtensionLoader(Student.class);
      Student studentImpl = extension.Loader.getExtension("StudentImplA")
      studentImpl.service()
      // StudentImplA
      ```

   3. dubbo服务暴露的刨析

      1. 加载注册中心的链接
      2. 导出服务，注册到注册中心（生成一个链接，包括主机名，时间戳，版本，方法名等等）

   4. 服务引入的源码

      生成代理类，使用NettyClient进行通信
