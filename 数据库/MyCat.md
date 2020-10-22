##### 安装

http://dl.mycat.org.cn/

##### 配置文件

```
schema.xml:定义逻辑库，表，分片节点等内容
rule.xml:定义分片规则
server.xml:定义用户以及系统相关变量
```

```xml
<!-- server.xml -->
<user name="mycat" defaultAccount="true">
		<property name="password">123456</property>
		<property name="schemas">kiy</property>
		<property name="defaultSchema">kiy</property>
		<!--No MyCAT Database selected 错误前会尝试使用该schema作为schema，不设置则为null,报错 -->
</user>
<!-- 配置的虚拟库需要和实际库的名称相同，否则使用navicate等工具时会找不到表 -->
```

```xml
<!-- schemal-->
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
	<!-- randomDataNode is default data node -->
	<schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" randomDataNode="dn1">
	</schema>
	<dataNode name="dn1" dataHost="host128" database="kiy" />
	
	<dataHost name="host128" maxCon="1000" minCon="10" balance="0"
			  writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">	  
		<heartbeat>select user()</heartbeat>
		<!-- can have multi write hosts -->
		<writeHost host="hostM128" url="192.168.43.128:3306" user="root"
				   password="lele1937252523">				  
				    <!-- can have multi read hosts -->
					<readHost host = "hostS128" url="192.168.43.129:3306" user = "root" password = "lele1937252523">				   
		</writeHost>
	</dataHost>	
</mycat:schema>   	
```

>  读写分离，负载均衡配置，dataHost的balance属性

 1. **balance = "0" 不开启读写分离机制，所有读操作都发送到当前可用的writeHost上**
  2. **banlace = "1" 全部的readHost 与 stand by writeHost参与select的负载均衡，简单的说，当双主双从模式(M1->S1),**

    **(M2->S2)，并且M1和M2互为主备，正常情况下，M2，S1,S2都参与select语句的负载均衡**
  3. **banlance ="2" 所有的读操作都随机在writeHost,readhost上分发**   
  4. **banlace = "3" 所有的读请求随机的分发到readhost,writerHost不负担读压力**   

**注意：企业里面一般设置为1或者3，也就是双主双从或者单主单从的情况**

##### 启动

```java
./mycat console
管理端口 9066 
数据端口 8066
```

##### 主从复制

```bash
# 注意：mysql和redis的主从复制的差异
# redis接入从机的时候是会先复制所有数据
# 但是mysql接入从机的时候，只从接入点开始复制

# mysql的binary.log会记录mysql所有的写操作，此时从机会读取该文件，然后写入自己的relay.log日志，最后使用# sql线程执行操作，主从复制存在延时的问题
```

##### Mysql一主一丛

```bash
# master my.cnf
# 注意：建立数据库的sql也将传输到从机，且主从复制的时机是在接入点，接入点前的操作，从库不会执行，所以在搭建主从复制的时候设置需要复制的主库名字，和从库是否需要提前建立好库和表需要仔细考虑

# 主服务器唯一id
server-id = 1
# 启用二进制文件
log-bin = mysql-bin
# 设置不要复制的数据库(可设置多个,如果设置了binlog-do-db那这一块就不需要设置了)
binlog-ignore-db = mysql
binlog-ignore-db = information_schema
# 设置需要复制的数据库名
binlog-do-db = dbname
# 设置logbin格式
binlog_format = STATEMENT

#  logbin的三种格式
#  STATEMENT  直接记录操作sql，但是会导致数据不一致，如now(),在两台机器上的时间计算不一样
#  ROW 记录更改的数据行，即sql更改了多少行，就把这些行发送到从库，数据一致性很强，但是效率低
#  MIXED 将now之类会影响主从不一致的sql,切换到ROW模式，其余的直接发送sql，但是无法解决如@@host数据库系统变量之类不一致的情况

# 所以开发过程中尽量使用代码来约束需要主从强一致性的字段，而不是使用sql函数或者系统变量

```

```bash
# slave my.cnf
server-id = 2
# 启用中继日志
relay-log = mysql-relay
```

```bash
# 主机类创建slave用户给从机使用
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%' IDENTIFIED BY 'lele1937252523'
# 注意：如果出现密码等级认证问题，可以降低密码强度
set global validate_password_policy=LOW
# 查看主库的配置状态
show master status
+------------------+----------+--------------+--------------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB         | Executed_Gtid_Set |
+------------------+----------+--------------+--------------------------+-------------------+
| mysql-bin.000001 |      430 | kiy          | mysql,information_schema |                   |
+------------------+----------+--------------+--------------------------+-------------------+
```

```bash
# 从机操作
# 关闭当前原来的slave操作
stop slave;

CHANGE MASTER TO MASTER_HOST = '192.168.43.128',
MASTER_USER = 'slave',
MASTER_PASSWORD = 'lele1937252523',
MASTER_LOG_FILE = 'mysql-bin.000001',
MASTER_LOG_POS = 154;
# 启动复制关系
start slave


# 从节点彻底取消主从关系reset slave all

# 主节点清除 binglog日志 reset master; 如果有slave运行，master将不能执行该命令

# 主节点查看当前连接的从节点 show slave hosts



# 此处有几个坑需要避免
Slave_IO_Running: No  可能是两主机的uuid或者server_id相同（原因可能是容器复制）
# 解决：find / -name 'auto.cnf' 修改auto.cnf文件中的uuid 或者my.cnf中的server_id

Slave_SQL_Running: No 可能是MASTER_LOG_POS填写错误，由于错误操作导致指针前移
# 解决：查看最新的MASTER_LOG_POS

```

##### 读写分离

```
# 注意：使用navicate等工具连接数据库时，创建的虚拟库名要和实际库名相同，否则将获取不到数据库
```



##### docker容器下的多机群搭建

```bash
[mysqld]
# 这个是MySQL容器默认的数据库安装路径
basedir         = /usr
socket          = /mysqldata/3306/data/discard/other/mysql.sock
pid-file        = /mysqldata/3306/data/discard/other/mysqld.pid
# 数据
datadir         = /mysqldata/3306/data
# 错误日志
log-error       = /mysqldata/3306/data/discard/errlog/mysqld.err
# 这个缺省为datadir，可不写
innodb_data_home_dir = /mysqldata/3306/data 
# binlog 从库无需开启
log-bin=/mysqldata/3306/data/discard/logdir/binlog
# redolog
innodb_log_group_home_dir = /mysqldata/3306/data/discard/logdir/redolog
# 临时文件
tmpdir=/mysqldata/3306/data/discard/tmpdir
#慢查询日志
slow_query_log_file=/mysqldata/3306/data/discard/slowlog/slow.log

server-id = 12801
# 启用中继日志
relay-log = mysql-relay
```

```
mkdir -p /home/dconfs/mysql/12802/data
mkdir -p /home/dconfs/mysql/12802/logdir/binlog
mkdir -p /home/dconfs/mysql/12802/logdir/redolog
mkdir -p /home/dconfs/mysql/12802/errlog
mkdir -p /home/dconfs/mysql/12802/tmpdir
mkdir -p /home/dconfs/mysql/12802/slowlog
mkdir -p /home/dconfs/mysql/12802/conf
mkdir -p /home/dconfs/mysql/12802/other
chmod -R 755 /home/dconfs/mysql/12802/conf
```

```
docker run -d --name slave13001 -e MYSQL_ROOT_PASSWORD=123456 -p 3301:3306 \
-v /home/dconfs/mysql/13001/data:/mysqldata/3306/data \
-v /home/dconfs/mysql/13001/errlog:/mysqldata/3306/data/discard/errlog \
-v /home/dconfs/mysql/13001/logdir/binlog:/mysqldata/3306/data/discard/logdir/binlog \
-v /home/dconfs/mysql/13001/logdir/redolog:/mysqldata/3306/data/discard/logdir/redolog \
-v /home/dconfs/mysql/13001/tmpdir:/mysqldata/3306/data/discard/tmpdir \
-v /home/dconfs/mysql/13001/slowlog:/mysqldata/3306/data/discard/slowlog \
-v /home/dconfs/mysql/13001/other:/mysqldata/3306/data/discard/other \
-v /home/dconfs/mysql/13001/conf/my.cnf:/etc/mysql/my.cnf \
--privileged=true mariadb:latest
```

```
# 修改容器为自启动
docker container update --restart=always mysql3306 
```

##### 多主多从

```
# 在作为从数据库的时候，有写入操作也要更新二进制文件
log-slave-updates
# 表示自增长字段每次递增的量，默认是1
auto-increment-increment =2 
# 表示自增长字段从哪个数开始，指定字段一次递增多少
auto-increment-offset = 1



auto-increment-increment
auto-increment-offset为什么必须设置为不一样

在双主机的情况下，如果A,B
A写入id=1，B备机
A宕机，但是在数据还没有更新到B,此时使用B作为主力写，那么B的自增id还是从1开始
当A再开启的时候，AB有重复id数据

所以在多台备机的情况下，自增id的规律应该是  即几台主机auto-increment-increment就等于多少
# 两台
1,3,5,7
2,4,6,8

# 三台
1,4,7
2,5,8
3,6,9

# 四台
1,5,9
2,6,10
3,7,11
4,8,12

```

> 多主机的精髓:A为B的从，B为C的从，C为A的从
>
> 128->129->130->128

##### 多主机备份下的容灾思考

```bash
mycat的dataHost的属性
writeType = "0" 所有的写操作都将发送到第一台主机，如果挂了切换到生存的第二个
switchType="1" 默认值，自动切换，-1不自动切换，2基于同步状态决定是否切换
```



##### 基于mycat的分库分表

> 原则：单表的瓶颈是1000万条，两个数据库在一台机器上，能做join，但是两个数据库在两台机器上，不能join

- 竖直拆分库----将强关联的表放到一个物理集群库里面

  ```xml
  <!-- 默认的表都到dn1配置节点数据库集群中去查找 -->
  <schema name="kiy" checkSQLschema="false" sqlMaxLimit="100" dataNode = "dn1">	
     
      <!-- 配置customer表到dn2集群中去查找，也就是dn1中可以没有这张表,达到竖直分库作用 -->
      <table name="customer" dataNode = "dn2"/>
     
      
      <!-- 
  		配置order表到dn1,dn2两个集群中查找，也就是dn1,dn2中都要有这张表，并使用mod_rule规则进行区分
   		达到水平分库的作用
  	-->
      <!-- 盲猜丰驰就设置了水平分表，每个表都写到了128个库里，效率根本不高 -->
      <table name="order" dataNode = "dn1,dn2" rule="mod_rule"/>
  </schema>
  
  <dataNode name ="dn1" dataHost="host1" database = "orders">    
  <dataNode name ="dn2" dataHost="host2" database = "orders">
  
      
  // 配置节点数据库集群
  <dataHost name="host1" maxCon="1000" minCon="10" balance="3"
  			  writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">	  
  		<heartbeat>select user()</heartbeat>
  		<!-- can have multi write hosts -->
  		<writeHost host="hostM128" url="192.168.43.128:3306" user="root"
  ....             
  ```

- 水平拆分表

  ```
  将表分到多个物理机库中
  规则：单一库可满足原则，比如订单，使用用户id划分库，这样大多数场景下一个库能满足数据查询
  
  在水平分表的情况下，如果不指定orderby,那么id将可能不会按照正常顺序来排
  ```

- rule.xml

  ```xml
  <tableRule name = "mod_rule">
      <rule>
          <!-- 分片字段 -->
          <columns>slice_id</columns>
          <!-- 分片算法，如丰驰将对128取余 -->
          <algorithm>mod-long</algorithm>
      </rule>
  </tableRule>
  ```

- 水平分片情况下的join考虑

  1. **字段冗余**，在业务上对于数据进行划分，比如丰驰同一个中转场的数据都有一个中转场编码，但是这样很傻
  
  2. **设置子表**，即子表数据中有字段和主表关联，比如订单表和订单详情表,子表数据将按照父表数据进行分片规则
  
   ```xml
     <table name= "orders" dataNode="dn1,dn2" rule="mod_rule">
     	<childTable name = "oders_detail" primaryKey="id" joinKey="order+id" parentKey="id"/>
     </table>
   ```
  
     
  
  3. **全局表**,业务表分片之后，相关的其他无需分片但是又需要进行关联查询的表咋办，也就是关于把丰驰业务库和基础库合并的思考?
  
     ```xml
     <table name="transit_dept_conf" dataNode = "dn1,dn2" type="global"></table>
     
     // 注意：全局表的思想是在多个数据节点上拥有全量的数据，例如按照如上配置，dn1和dn2都将有中转场的全量数据，这样会造成大量冗余数据，这种情况只有在数据量并不够大的配置表且经常需要关联查询的配置表才使用
     ```

##### 常用的分片规则

1. 取模，常用 mod_long

2. 分片枚举 

3. 范围约定 rang_long

4. 按照日期分片 

   ```xml
   <!--自定义分片规则-->
   <function name ="shardingByDate" class="io.mycat.route.function.PartitionByDate">
       <property name="dateFormat">yyyy-MM-dd</property>
       <property name ="sBeginDate">2019-01-01</property> <!-- 开始日期 -->
       <property name ="sEndDate">2019-01-04</property> <!-- 结束日期,必须设置，到结束日期时会重新从第一个节点插入 --> 
       <property name ="sPartitionDay">2</property> <!-- 两天分一片 -->
   </function>
   ```

##### 全局序列

> 全局ID的唯一性，使用数据库自增无法满足唯一性 (如果是使用步长自增，无法扩容，所以自增一般不用)

1. 时间戳

2. 数据库方式，使用数据库存放ID，mycat先去某个库中拿一批ID， 每一次进入插入的时候使用，不够了再去拿，类似于oms的请求容器号

   ```xml
   <!-- vim sequence_db_conf.properties 先修改配置文件确定全局序列配置位于哪个数据库节点 -->
   
   <!-- vim server.xml 确认使用哪一个全局序列方式 -->
   <property name="sequnceHandlerType">1</property>
   
   <!-- 调用全局序列 -->
   <!-- insert into orders(id,amount,cunstomer) values(next value from MYCATSEQ_ORDERS,1999,1999)-->
   
   ```

##### 高可用方案

**HAProxy+Keepalived** 配合两台Mycat大器Mycat集群实现高可用性，HAProxy实现了mycat多节点的集群高可用性和负载均衡，而HAProxy自身的高可用可以用Keepalived来实现

![无标题](C:\Users\皿煮国的潜逃败类\Desktop\无标题.png)

```xml
#### 目前虚拟机的数据库的集群方案是

# 使用keepalived保证了haproxy的高可用


# 使用haproxy保证了mycat的负载均衡
haproxy
192.168.43.129:48066

mycat_1
192.168.43.128:9092

mycat_2
192.168.43.129:9092


数据节点DBnode1 三主三从 
master:宿主机192.168.43.129:3306   容器从机192.168.43.129:3301
master:宿主机192.168.43.130:3306   容器从机192.168.43.130:3301
master:宿主机192.168.43.128:3306   容器从机192.168.43.128:3301

数据节点DBnode2 单主机
master:宿主机192.168.43.129:3302

数据节点DBnode3 单主机
master:宿主机192.168.43.130:3302

数据节点DBnode4 单主机
master:宿主机192.168.43.128:3302

#### 数据节点DBnode2,DBnode3,DBnode4未做容灾处理
```

##### HAProxy

> 反向代理服务器，将数据请求转发到其他服务器的特定端口

```
# 安装haproxy
make TARGET=linux310 PREFIX=/usr/local/haproxy ARCH=x86_64
make install PREFIX=/usr/local/haproxy
```

```
# 配置文件
global
	log 127.0.0.1 local0
	maxconn 4096
	chroot /usr/local/haproxy
	pidfile /usr/local/haproxy/data/haproxy.pid
	uid 99
	gid 99
	daemon
	

defaults
	log global
	mode tcp
	option abortonclose
	option redispatch
	retries 3
	maxconn 2000
	timeout connect 5000
	timeout client 50000
	timeout server 50000


listen proxy_status 
	bind :48066
	mode tcp
	balance roundrobin
	server mycat_1	192.168.43.128:8066 check inter 10s
		

frontend admin_stats 
	bind :7777
	mode http
	stats enable
	option httplog
	maxconn 10
	stats refresh 30s
	stats uri /admin
	stats auth admin:123456
	stats hide-version
	stats admin if TRUE
	
# 启动haproxy	
./haproxy -f /usr/local/haproxy/conf/haproxy.conf 
# 访问haproxy控制台
192.168.43.129:7777/admin


# 负载均衡访问mycat
mysql -umycat -p123456 -h 192.168.43.129 -p 48066
```

##### keepalived

```
# 编译安装
# 安装依赖
yum install -y gcc openssl-devel popt-devel
# 配置安装目录，每一个源码包都有一个configure的shell文件
./configure --prefix=/usr/local/keepalived
# 编译安装
make&&make install

# 拷贝源码提供的开机启动文件
cp /usr/local/keepalived-1.4.2/keepalived/etc/init.d/keepalived /etc/init.d/

# 拷贝源码提供系统配置文件
cp /usr/local//keepalived-1.4.2/keepalived/etc/sysconfig/keepalived /etc/sysconfig

# 创建keepalived配置目录
mkdir /etc/keepalived

# 拷贝安装后的配置文件
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived

# 拷贝执行文件，使之成为用户命令
cp /usr/local/keepalived/sbin/keepalived /usr/sbin/

# vi /etc/keepalived/keepalived.conf

# 走到了这一步,学完nginx再来












```

















































