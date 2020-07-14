##### 连接

```bash
# 开机
redis-server ../etc/redis.conf
# 连接
redis-cli -h localhost -p 6379
# 使用密码连接
redis-cli -h localhost -p 6379 -a 123456
# 关闭
redis-cli -h localhost -p 6379 -a 123456 shutdown
# 关闭
kill -9 pid
```





##### 配置文件

> 你以后工作中玩就是玩这个配置文件

1. INCLUDE

   ```bash
   # 配置文件包含
   include /path/to/local.conf 
   ```

2. GENERAL

   ```
   daemonize yes 
   pidfile /var/run/profile.log   # 进程管理文件
   port						   # 端口
   tcp-backlog					   # 解决客户端慢连接问题，tcp连接队列总和设置
   timeout						   # 是否自动中断连接，如果在一定时间内无传输信息
   bind						   # 绑定主机地址
   tcp-keepalive                  # 集群心跳连接测试
   loglevel					   # 日志级别
   logfile						   # 日志文件
   syslog-enabled				   # 系统启动日志相关	
   syslog-ident
   syslog-facility
   databases					   # 数据库数量
   ```

3. SECURITY

   ```
   安全访问限制相关，密码
   ```

4. LIMITS

   ```
   连接数，内存空间
   ```

##### 简单测试

```bash
redis-benchmark  redis性能测试
```

##### 基础命令

```bash
select n  选择库
DBSIZE    获取数据量
keys * 	  获取所有key
FLUSHDB   清除当前库
FLUSHALL  清除所有库
```

##### 数据类型

| 类型   | 描述                  |
| ------ | --------------------- |
| String | 简单key-value         |
| Hash   | HashMap               |
| List   | 链表                  |
| Set    | String无序集合        |
| Zset   | String有序集合,带权重 |

##### 常用命令

http://doc.redisfans.com/

1. **KEY**

| 命令             | 描述                                                     |
| ---------------- | -------------------------------------------------------- |
| keys *           | 查看所有key                                              |
| exists key       | 判断key是否存在                                          |
| move key db1 db2 | 数据移库                                                 |
| expire key       | 设置过期时间                                             |
| ttl key          | time to leave 查看过期时间,-1表示永不过期,-2表示已经过期 |
| type key         | 查看key是什么类型                                        |

2. **String**

| 命令                    | 描述                                  |
| ----------------------- | ------------------------------------- |
| set/get/del             | 增删查操作                            |
| append/strlen           | 字符串操作                            |
| Incr/decr/incrby/decrby | 数字操作,递增，递减                   |
| getrange/setrange       | 字符串裁剪和覆盖子字符串              |
| setex/setnx             | set with exipre ,set if not exist     |
| mset/mget/msetnx        | 批量操作,msetnx要么全成功，要么全失败 |
| getset                  |                                       |

3. **List**

| 命令                           | 描述lpush                                 |
| ------------------------------ | ----------------------------------------- |
| lpush/rpush/lrange             | 根据不同方向插入,lrange查询list的指定区间 |
| lpop/rpop                      | 根据不同方向获取第一个                    |
| lindex                         | 按照索引下标获取元素                      |
| llen                           | 链表总长度                                |
| lrem key n value               | 删除n个value                              |
| ltrim key  开始index 结束index | 截取子链表赋值给当前链表                  |
| rpoplpush 源链表 目的链表      | 取值放到另外一个链表                      |

如果list中的值全部移除，那么key将会消失，也就是redis链表没有空数组的说法

4. **Set**

| 命令                            | 描述lpush        |
| ------------------------------- | ---------------- |
| sadd/smembers/sysmember         | 添加元素         |
| scard                           | 获取元素中的个数 |
| srem key value                  | 删除集合中元素   |
| srandmember key  n              | 随机出n个数      |
| spop key                        | 随机出栈一个     |
| smove key 1 key2 在key1中某个值 | 值迁移           |
| sdiff                           | 集合差集         |
| sinter                          | 集合交集         |
| sunion                          | 集合并集         |

4. **Hash**

| 命令                                | 描述lpush                               |
| ----------------------------------- | --------------------------------------- |
| hset/hget/hmeset/hmget/hgetall/hdel | hash表操作**（重要）**                  |
| hlen                                | 多少个key                               |
| hexist key k                        | hash表中某个key是否存在                 |
| hkeys/hvals                         | 获取hash表中所有的key或value **(重要)** |
| hincrby/hincrybyfloat               | hash表中某个值递增，递减                |
| hsetnx                              | hash表中某个key不存在则增加             |

5. **Zset**

| 命令                                  | 描述lpush          |
| ------------------------------------- | ------------------ |
| zadd/zrange                           | 添加，根据权重获取 |
| zrangebyscore                         | 根据权重域获取     |
| zrem                                  | 删除集合元素       |
| zcard/zcountbyscore                   | 获取集合长度       |
| zrevrank key values                   | 逆序获得下标值     |
| zrevrangebyscore   结束权重到开始权重 | 根据权重域获取     |



##### RDB

1. Snapshot快照,redis会单独的创建一个子线程来进行持久化，会将数写入到一个临时文件，待持久化结束后，再用这个临时文件替换上一次持久化好的文件，整个过程中，主进程是不进行IO操作，如果需要进行大规模的数据的恢复，且对于数据恢复的完整性不是很敏感，那RDB方式要比AOF方式更加的高效，**RDB的缺点是最后一次持久化的数据可能丢失**

2. Fork的作用是赋值一个与当前进程一样的进程，新进程所有的数据和原来进程所有的数据都是一直，但是是一个全新的进程，并作为原进程的子进程

3. RDB保存的是dump.rdp文件

   ```bash 
   # 配置规则 面试就说默认规则答这个就行
   1.save 900 1     900s中有一个key改变
   2.save 300 10	 300s中有10个key改变
   3.save 60 10000	 60s中有10000个key改变
   4.save ""		 不进行RDB操作
   
   # 记住使用shutdown关闭的时候，会迅速生成一个关机前的RDB
   # RDB只能恢复断电，关机等操作的数据
   # 如果使用flushall 再关机，那关机前的备份数据就是空，咋也恢复不了
   
   #save或者bgsave即时备份操作，客户端执行save命令，马上备份
   save 只管save,全部阻塞，半夜干活
   bgsave 备份同时也能写入
   
   
   #conf
   # 如果有一次备份失败，马上停止写入，并且报错
   stop-writes-on-bgsave-error yes 
   # 是否使用压缩算法来进行备份数据压缩，这个会影响cpu性能，也就是磁盘换cpu,如果cpu够的话最好关了，看老板
   rdbcompression yes
   # 使用CRC64算法来进行数据校验，但是这样做会增加10%性能消耗
   rdbchecksum yes
   
   # RDB文件名
   dbfilename dump.rdb
   
   # dir 备份文件存放路径
   ./
   
   注意:一般十五分钟或者半小时恢复一次，RDB会丢失最后一次的快照数据，也就是最后一次修改的数据将会丢失，所以需要引入AOF
   ```

##### AOF

> 为了解决RDB导致的最后一次数据丢失的问题，但是其实这种数据修复的东西，运维完全可以搞定，据说两年的数据都能修复过来

```
# AOF类似于mysql的log日志，记录写操作
# 问题：写入大量的log
# 开启AOF
appendonly yes
# 请他妈的记住 flushall之类的操作 必须找运维，aof也会记录flushall操作

# 如果appendonly损坏，咋办？
# 如果appendonly损坏，那么redis将启动不了
# 此时需要修复文件，redis-check-aof --fix appendonly aof 命令，将会删除aof文件中所有不符合规范的字符


# 问题：RDB和AOF同时开启，数据恢复的逻辑是什么
# 如果共存，先读取AOF，这时候如果aof文件损坏，那么redis就会起不来
# 官方文档中的意思是如果存在aof将会读取aof，不存在则读取rdb,因为aof更加准确


#conf
# 该配置最好使用shell脚本来动态更改，在高峰期绝对要关闭掉aways
Appendfsync
	1.Aways 每次数据发生变更的时候都会被立即记录到磁盘上，性能差，但是数据绝对完整
	2.Everysec 每秒记录，会有一秒钟的数据丢失,(推荐)
	3.No	

# 重写机制
AOF采用文件追加的形式，文件会越来越大，避免这种情况新增了重写的机制，当AOF的文件大小超过了所设定的阈值，redis启动AOF文件内容压缩，只保留可以恢复数据的最小指令集，使用bgrewriteaof重写

# 重写数据的百分比
auto-aof-rewrite-percentage 100
# 重写数据的阈值，看一下这个重写大小就知道公司啥水平了
# 市场行情：3GB
# 重写压缩数据的大小，每一次新增超过64mb的时候，将这些数据压缩重写一次
auto-aof-rewrite-min-size 64mb
# 重写数据的基准值
auto-aof-rewrite-percentage 
原理：
```

##### 事务

**redis是单进程并不是单线程所以多个客户端的数据还是可能修改同一份数据,导致事务问题****

DISCARD 取消事务

EXEC 执行所有事物块内的命令

MULTI 标记一个事务的开始

UNWATCH 取消watch对于所有key的监视

WATCH 监视一个或者多个key

监视：如果事务执行前被监视的key被其他命令修改，那么事务将被打断

```
# 事务的基本执行
localhost:6379> clear
localhost:6379> keys *
(empty array)
localhost:6379> MULTI 
OK
localhost:6379> set k1 v1
QUEUED
localhost:6379> set k2 v2
QUEUED
localhost:6379> EXEC
```

##### 事务的两样性

```
# 事务的回滚
MULTI
set k1 v1
getset k2 v2
EXEC
# 如果getset报错，k1也将不执行,这种类似于编译时错误，将会直接挂掉回滚
```

```
MULTI
set k1 1
incr k1
EXEC
# 但是像incr这样的命令，类似于运行时错误，前面的k1还是能插入进去，不会全部回滚
```

> 这就是为什么在回答redis是否支持事务的时候，只能答部分支持



##### 乐观锁 (类似行锁，并发性)

> 乐观锁和悲观锁就是并发性和一致性的对立

乐观锁的实现：在每一条记录的后面加一个版本号

张三：获取版本号V1

李四：获取版本号V1

李四：提交，并将版本号改为V2

张三：提交，发现版本号和自己获取的不一样，将会重新获取这条数据，然后第二次进行他的操作，再提交

`乐观锁的本质是每一次提交的时候判断自己修改的那条数据的版本号是否和自己拿到的版本号一致，不一致则重新获取该数据，工作中用的多是乐观锁`

##### 悲观锁 (类似表锁，一致性)

锁住整张表，非常安全，但是不用，在考虑并发的情况下，锁表是不可能锁表的，这辈子都不可能锁表

##### CAS (Check and Set)

##### WATCH

```bash
WATCH lock
MULTI
# 如果在在这个过程中有另外一个客户端修改了lock，事务将执行失败
incr lock	
EXEC
UNWATCH lock
# WATCH的本质意思就是在使用监听某一个key的时候
# 后续执行的事务中将会对这个key进行版本号判断，即乐观锁判断，如果被其他客户端修改，则报错

所以我们使用乐观锁执行事务的逻辑应该是

WATCH lock
process()
UNWATCH lock
# 一旦执行EXEC操作，前面加的锁都将被清除掉
function process(){
	MULTI
	操作
	if(!EXEC) process()
}
```

##### 消息发布和订阅机制

> 进程中的一种消息通信模式 一般没人用，使用kafka,rocketMQ 等中间件更多



##### 主从复制

1. 配从不配主

2. 从库配置

   ```bash
   slaveof 主库IP 主库端口
   # 注意：每一次和master断开之后，都需要重新连接，除非配置进redis.conf中
   
   ```

   

3. 修改文件细节操作









