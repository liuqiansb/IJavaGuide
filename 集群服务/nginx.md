#### docker 安装nginx

```

docker pull nginx
docker run --name runoob-nginx-test -p 8081:80 -d nginx
mkdir -p /home/dconfs/nginx/12801/www  /home/dconfs/nginx/12801/conf /home/dconfs/nginx/12801/logs
# 复制基础配置文件到宿主机，为建立挂载做准备
docker cp 6dd4380ba708:/etc/nginx/nginx.conf /home/dconfs/nginx/12801/conf

#  删除原来的容器重新配置
docker run -d -p 8081:80 --name nginx01 -v /home/dconfs/nginx/12901/www:/usr/share/nginx/html -v /home/dconfs/nginx/12901/conf/nginx.conf:/etc/nginx/nginx.conf -v /home/dconfs/nginx/12901/logs:/var/log/nginx nginx
```

##### docker安装tomcat

```
docker pull tomcat
docker run --name tomcat -p 8080:8080 -d tomcat
docker cp tomcat:/usr/local/tomcat/webapps.dist /home/dconfs/tomcat/12801/
docker cp tomcat:/usr/local/tomcat/conf /home/dconfs/tomcat/12801/
docker cp tomcat:/usr/local/tomcat/logs /home/dconfs/tomcat/12801/
mkdir /home/dconfs/tomcat/12801/
docker run --name tomcat02 -p 8882:8080 -v $PWD/webapps:/usr/local/tomcat/webapps -v $PWD/logs:/usr/local/tomcat/logs -v $PWD/conf:/usr/local/tomcat/conf -d tomcat
```

##### 当前系统服务器集群

```
容器机nginx:192.168.43.128:8081
容器机nginx:192.168.43.128:8082
容器机nginx:192.168.43.129:8081



容器机tomcat:192.168.43.130：8082
容器机tomcat：192.168.43.130：8882
容器机tomcat：192.168.43.130：8883




```

##### 反向代理

```bash
#nginx.conf
user  nginx;
worker_processes  1;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;
events {
    worker_connections  1024;
}
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    #tcp_nopush     on;
    keepalive_timeout  65;
    #gzip  on;
    #include /etc/nginx/conf.d/*.conf;
	server {
		# 注意，此处的端口配置应该是容器内的nginx端口
		listen 80;
		# 这个server_name其实可以乱填
		server_name 192.168.43.128:8082;
		# 根据不同的路径进行转发 ~ 表示后面是正则表达式形式
		location ~ /tdop-air/ {
			root html;
			index index.html index.htm;
			# 转发地址
			proxy_pass http://192.168.43.130:8082
		}
		location ~ /tdop-core/ {
			root html;
			index index.html index.htm;
			# 转发地址
			proxy_pass http://192.168.43.130:8081
		}
		
	}
}


1.=：用于不含正则表达式的uri，要求请求字符串和uri严格匹配
2.~：用于表示uri包含正则表达式，且区分大小写
3.~*：用于表示uri包含正则表达式，且不区分大小写
4.^~：用于不含正则表达式的uri,要求nginx找到标识uri和请求字符串匹配度最高的location后，立刻使用此location进行处理，而不再使用location块中的正则uri和请求字符串做匹配
```



##### 负载均衡

```
# 依赖stream_upstream_module模块
#http://nginx.org/en/docs/stream/ngx_stream_upstream_module.html
user  nginx;
worker_processes  1;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;
events {
    worker_connections  1024;
}
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    #tcp_nopush     on;
    keepalive_timeout  65;
    #gzip  on;
    #include /etc/nginx/conf.d/*.conf;
	# 负载均衡池
	upstream webpools {
		server 192.168.43.130:8082 weight=5;
		server 192.168.43.130:8882 weight=5;
		# 备用只有在前面某个server挂掉之后，才会转发到备用机
		server 192.168.43.130:8883 weight=5 backup;
	}
	# 设置服务名称和代理地址
	server {
		listen 80;
		server_name 192.168.43.128:8082;
		location / {
			root html;
			index index.html index.htm;
			proxy_pass http://webpools;
		}
		
	}
}
```

##### 动静分离

```
1.根据不同的url后缀转发到不同的服务器实现动静分离
2.通过expires参数设置，使用浏览器缓存过期时间减少服务器之前的请求和流量
3.expires的原理是请求该静态文件url判断文件更新时间，如果未更新，返回302，使用浏览器缓存的静态文件，该方式一般不用于ajax
```



##### keepalive的使用

fsaf

- 拟的IP的概念

  ```
  虚拟ip实际上就是一个虚拟网卡，对外访问的地址将是这个并不会实际存在的ip
  
  keepalive的作用其实就是将这个虚拟ip根据是否存活绑定到不同的服务器，
  
  
  1.主nginx服务器中需要一台keepalive
  2.从服务器中也需要一台keepalive
  3.需要一个虚拟ip
  ```

- 安装keepalived

  ```
  yum install keepalived -y 
  rpm -q -a keepalived
  ```

- 主从配置

  ```bash
  # 主机配置
  global_defs {
  	notification_email{
  		2436931151@qq.com
  		1937252523@qq.com
  	}
  	notification_email_from 192@firewall.com
  	smtp_server 192.168.17.129
  	smtp_connect_timeout 30
  	router_id LVS_DEVEL # 唯一值，也可以写ip
  }
  # 心跳检测脚本
  vrrp_script chk_http_port {
  	script "/usr/local/src/nignx_check.sh"
  	interval 2 # 心跳检测时间间隔
  	werght 2	# 权重
  }
  vrrp_instance VI_1 {
  	state MASTER # 备份服务器上将此改为BACKUP
  	interface ens33 # 网卡
  	virtual_router_id 51 # 主，备机的virtual_router_id 需要相同
  	priority 100 # 主，备机取不同的优先级，主机值较大，备机值较小
  	advert_int 1
  	authentication {
  		auth_type PASS
  		auth_pass 1111
  	}
  	virtual_ipaddress {
  		192.168.17.50 # 虚拟ip地址
  	}
  }
  ```

  ```
  # 主服务器检测脚本
  #!/bin/bash
  A=`ps -C nginx -no-header | wc -l`
  if [$A -eq 0];then
  	/usr/local/nginx/sbin/nginx	# 判断当前nginx是否启动，如果未启动，干掉keepalive进程
  	sleep 2
  	if [`ps -C nginx --no-header | wc -l` eq 0];then
  		killall keepalived	# 干掉keepalive进程后，那么只有一台keepalive服务器，他将把虚拟ip地址绑定到自身主机上
  	fi
  fi
  ```

  ```
  # 主机配置
  global_defs {
  	notification_email{
  		2436931151@qq.com
  		1937252523@qq.com
  	}
  	notification_email_from 192@firewall.com
  	smtp_server 192.168.17.129
  	smtp_connect_timeout 30
  	router_id LVS_DEVEL # 唯一值，也可以写ip
  }
  # 心跳检测脚本
  vrrp_script chk_http_port {
  	script "/usr/local/src/nignx_check.sh"
  	interval 2 # 心跳检测时间间隔
  	werght 2	# 权重
  }
  vrrp_instance VI_1 {
  	state SLAVE # 备份服务器上将此改为BACKUP
  	interface ens33 # 网卡
  	virtual_router_id 50 # 主，备机的virtual_router_id 需要相同
  	priority 100 # 主，备机取不同的优先级，主机值较大，备机值较小
  	advert_int 1
  	authentication {
  		auth_type PASS
  		auth_pass 1111
  	}
  	virtual_ipaddress {
  		192.168.17.50 # 虚拟ip地址
  	}
  }
  ```

  ```
  
  ```

##### 虚拟ip的概念

众所皆知的，IP 位址仅为 xxx.xxx.xxx.xxx 的资料型态，其中， xxx 为 1-255 间的整数，由于近来计算机的成长速度太快，实体的 IP 已经有点不足了，好在早在规划 IP 时就已经预留了三个网段的 IP 做为内部网域的虚拟 IP 之用。这三个预留的 IP 分别为：

 A级：10.0.0.0 - 10.255.255.255

 B级：172.16.0.0 - 172.31.255.255

 C级：192.168.0.0 - 192.168.255.255

私有位址的路由信息不能对外散播,使用私有位址作为来源或目的地址的封包﹐不能透过Internet来转送

##### 实现原理

 我们知道**一般的IP地址是和物理网卡绑定的，而VIP相反，是不与实际网卡绑定的的IP地址**。当外网的上的一个机器，通过域名访问某公司内网资源时，**内网的DNS服务器会把域名解析到一个VIP上**。当外网主机经过域名解析得到这个VIP后，就将数据包发往这个VIP。但是在内网中，这个VIP是不与具体的设备相连接的，**所以外网发过来的目的地址是VIP的IP数据包，究竟会到哪台机器呢？**

内网的的网络传输过程通过ARP协议来完成，将VIP映射到的MAC地址可以控制

主机地址：192.168.43.128	物理地址：00:21:5A:DB:7F:C2

从机地址：192.168.43.129    物理地址：00:21:5A:DB:68:E8

虚拟ip地址：192.168.43.133 

keepalive启动时，主keepalive服务会发送ARP数据包告知所有的内网主机，192.168.43.133将链接到物理机00:21:5A:DB:7F:C2，而从会先判断主keepalive是否启动，如果已经启动，将不发送ARP数据包，当主挂掉的时候，从keepalive将会发送ARP数据包，告诉所有的主机192.168.43.133切换到00:21:5A:DB:68:E8

keepalive虚拟ip技术主要的原理在于ARP ip和物理地址高速缓存的实现，每台主机中都有一个ARP高速缓存，存储同一个网络内的IP地址与MAC地址的对应关系，keepalive通过发送ARP数据包的形式，更改了所有内网机器的ARP缓存，让他们的请求到指定的物理机

而内网则是基于这个





































