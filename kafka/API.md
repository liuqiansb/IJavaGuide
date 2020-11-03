##### 生产者API

- 采用异步发送的方式，启动Sender线程，有一个和main线程共享的RecordAccumulator的变量

  Sender不断从RecordAccumulator拉取数据发送到broker

  ![无标题.1png](C:\Users\皿煮国的潜逃败类\Desktop\无标题.1png.png)

##### 生产者

```xml
<dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>0.11.0.0</version>
 </dependency>
```

```java
/**
 * 基本使用
 */
public class MyProducer {
    public static void main(String[] args) {
        Properties properties = new Properties();
        // 指定连接的集群
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,"192.168.43.130:9092");
        // ack应答级别
        properties.put("acks","all");
        // 重试的次数
        properties.put("retries",1);
        // 批次大小
        properties.put("batch.size",16384);
        // 等待时间，如果没有达到批次大小，到了等待时间也会把数据发出去
        properties.put("linger.ms",1);
        // RecordAccumulator缓冲区大小
        properties.put("buffer.memory",33554432);
        // key,value的序列化类
        properties.put("key.serializer","org.apache.kafka.common.serialization.StringSerializer");
        properties.put("value.serializer","org.apache.kafka.common.serialization.StringSerializer");
        // 创建生产者对象
        KafkaProducer<String, String> producer = new KafkaProducer<String,String>(properties);
        for (int i = 0; i < 10; i++) {
            producer.send(new ProducerRecord<String, String>("test1","test--"+i));
        }
        producer.close();
    }
}

// 注意：ProducerConfig中含有所有的属性的key，和它的描述
```

```java
/**
 * 带有回调的producer
 */
public class CallBackProducer {
    public static void main(String[] args) throws InterruptedException {
        Properties properties = new Properties();
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,"192.168.43.130:9092");
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,"org.apache.kafka.common.serialization.StringSerializer");
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,"org.apache.kafka.common.serialization.StringSerializer");
        KafkaProducer<String, String> producer = new KafkaProducer<String,String>(properties);
        final CountDownLatch lock = new CountDownLatch(10);
        for (int i = 0; i < 10; i++) {
            producer.send(new ProducerRecord<String, String>("test1","callback"+i), new Callback() {
                public void onCompletion(RecordMetadata recordMetadata, Exception e) {
                    if (e==null){
                        System.out.println(recordMetadata.offset()+"--"+recordMetadata.partition());
                    }
                    lock.countDown();
                }
            });
        }
        lock.await();
        producer.close();
    }
}
```

```java
/**
 * 带有分区策略的producer,如果指定了key，不指定partition将会对key取hash存到不同分区
 * 如果指定了partition,那么直接放到指定分区
 */
producer.send(new ProducerRecord<String, String>(topic_name,partition_num,key,value), new Callback() {})
```

```java
/**
 *	自定义分区器，读DefaultPartitioner学学怎么写
 */  
public class MyPartitioner implements Partitioner {
    public int partition(String topic, Object key, byte[] bytes, Object value, byte[] bytes1, Cluster cluster) {
        Integer integer = cluster.partitionCountForTopic(topic);
        return key.toString().hashCode() % integer;
    }

    public void close() {

    }

    public void configure(Map<String, ?> map) {

    }
}

// 使用自定义分区策略
properties.put(ProducerConfig.PARTITIONER_CLASS_CONFIG,"partitioner.MyPartitioner")
```

```
/**
 * 阻塞,send方法返回的Future对象,但是生产环境基本不写同步发送
 */
Future future = producer.send(new ProducerRecord<String, String>(topic_name,partition_num,key,value), new Callback() {})
future.get();
```

##### 消费者

``` java
public class MyConsumer {
    public static void main(String[] args) {
        Properties properties = new Properties();
        // 连接集群
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,"192.168.43.130:9092");
        // 自动提交的延迟
        properties.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG,"1000");
        // 开启自动提交
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG,true);

        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,"org.apache.kafka.common.serialization.StringDeserializer");
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,"org.apache.kafka.common.serialization.StringDeserializer");

        // 消费者组
        properties.put(ConsumerConfig.GROUP_ID_CONFIG,"kiy_demo");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(properties);
        // 可以订阅多个主题
        consumer.subscribe(Collections.singletonList("test1"));

        while (true){
            // 获取数据,kafka是主动拉取的形式，所以维持了一个长轮询，这个是轮询的延迟
            ConsumerRecords<String, String> poll = consumer.poll(100);
            // 解析并且打印
            for (ConsumerRecord<String,String> record:poll){
                System.out.println(record.key()+"--"+record.value());
            }
        }
    }
}
```

```
// 该属性生效条件
// 1.该消费组第一次消费
// 2.如果当前消费组的offset在任何机器上都不存在了，数据已经被删除了,假设消费者挂掉了，且七天后才重启
properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG,“earlist”);
```

##### 手动提交offset

> 开发人员难以把握提交的时机，比如消费者还没有处理完，这是已经达到了提交时间间隙，但是如果还没有处理完，挂掉了，那么下一次重启的时候的offset就不对，所以提供了手动提交offset的api，**可能重复数据，但是不会丢数据**

```java
// 开启自动提交,此时将不再回写offset，那么每一次都将拉取重复数据
properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG,false);
// 配合着AUTO_OFFSET_RESET_CONFIG使用，将每一次都拉取全量的数据
properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG,“earlist”);


enable.auto.commit:是否自动提交
auto.commit.interval.ms : 自动提交offset的时间间隔    

// 注意：不回写数据，那么每一次拉取的offset都不会变，也就是每一次重启consumer的时候拉取的都是全量的
// 为什么是重启？
// 因为在消费者启动后，只访问一次文件系统中的offset，其余都是访问内存，所以不重写数据的影响应该是对下一次重启的时候
```

1. 同步提交

   将本次poll的一批数据的最高批次提交，consumer.commitSync(),一直提交直到提交成功才会拉取下一次的数据

2. 异步提交

   将本次poll的一批数据的最高批次提交,consumer.commitAsync(),继续拉数据，另开线程去提交，常用

##### 自定义存储offset

```java
1.先提交，后搞业务逻辑，业务逻辑挂掉了，丢数据
2.先搞业务逻辑，后提交，提交逻辑挂掉了，重复数据

此影响只在于重启后，因为consumer在运行过程中只会读取内存中的consumer，绝对正确，而在重启后才会去获取kafka中的初始值,所以可以将这个初始值存到数据库中


所以无论如何，使用kafka内的offset都是不可信的，所以引入了自定义存储，使用内存或者数据库，自己维护offset，这样可以保证offset绝对正确

 while (true){
            // 获取数据,kafka是主动拉取的形式，所以维持了一个长轮询，这个是轮询的延迟
            ConsumerRecords<String, String> poll = consumer.poll(100);
            // 解析并且打印
            for (ConsumerRecord<String,String> record:poll){
                System.out.println(record.key()+"--"+record.value());
            }
     		commit();      
}
void commit(){
      // 事务保持
      if(consumer.commitSync()){
          // 写入库
      } 
}
    
// 设置重启时自定义设置offset，即重置kafka中的offset到数据库中的正确值
consumer.subscribe(Collections.singletonList("test1"),new ConsumerRebalanceListener(){
      public void onPartitionsRevoked(Collection<TopicPartition> collection) {
            // 将数据库中的offset提交到kafka    
         	commitOffset(currentOffset);
      }
      public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
                // 调整kafka中的offset，使用数据库中的offset ： group-partition-offset 三列数据维护
          		currentOffset.clear();
                for(TopicPartition partition:partitions){
                    consumer.seek(partition,getOffset(partition));
            	}
      }
});
```

##### 拦截器

对于producer而言。拦截器在用户发送消息前和producer回调前有机会对于消息做一些定制化的操作，比如加上过滤，更改属性等

可以指定多个拦截器实现拦截链条,拦截器的实现接口时ProducerInterceptor

```java
public class MyProducerInterceptor implements ProducerInterceptor<String,String> {
    public ProducerRecord<String,String> onSend(ProducerRecord<String,String> producerRecord) {
        // 发送数据时调用，此处可以对数据进行更改
        String value = (String) producerRecord.value();
        ProducerRecord<String, String> recode = new ProducerRecord<String, String>(producerRecord.topic(), producerRecord.partition(), producerRecord.key(), "我设置一个默认值");
        return recode;
    }
    // 每次发送都会回调此方法，且它运行在io线程中，所以不要写太多东西，影响速度
    public void onAcknowledgement(RecordMetadata recordMetadata, Exception e) {
        // 发送成功之后的回调信息
    }

    public void close() {
        // 关闭，主要用于资源的清理工作
    }

    public void configure(Map<String, ?> map) {
        // 获取配置信息时使用，可以临时更改配置信息
    }
}
```

```java
// 添加拦截器的方法
ArrayList<String> interceptors = new ArrayList<String>();
interceptors.add("interceptor.MyProducerInterceptor");
properties.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG,interceptors);
```

##### Eagle监控器





































