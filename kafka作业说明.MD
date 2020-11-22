### 安装配置Kafka



**修改配置文件**

```
#kafka连接zookeeper的地址，此处使用本地启动的Zookeeper实例，连接地址是localhost:2181，后面myKafka是kafka在zookeeper中的根节点路径
zookeeper.connect=localhost:2181/myKafka
# 配置kakfa存储持久化数据的目录
log.dirs=/huorq/kafka/kafka_2.12-1.0.2/logs/kafka-logs
```



**启动kafka**

由于kafka依赖于jdk、zookeeper，所以在启动kafka之前需要启动zookeeper。

```
# 进入到kafka的安装路径下
/huorq/kafka/kafka_2.12-1.0.2
# 此处启动kafka引用的配置文件为相对路径
kafka-server-start.sh -daemon config/server.properties
```



**查看kafka的所有主题**

```
kafka-topics.sh --list --zookeeper localhost/myKafka
```



**创建kafka的主题，此处由于单机启动kafka,所以分区以及副本集都选择1**

```
 kafka-topics.sh --create -zookeeper localhost/myKafka --topic my_top_log --partitions 1 --replication-factor 1
```

![image-20201123001522335](C:\Users\myang\AppData\Roaming\Typora\typora-user-images\image-20201123001522335.png)



**安装nginx中的kafka模块**

```
# 下载nginx-kafka的依赖
git clone https://github.com/edenhill/librdkafka
cd librdkafka
# 编译
./configure
# 安装
make
make install

# 下载ngx_kafka_module模块
git clone https://github.com/brg-liuwei/ngx_kafka_module
# 进入之前nginx的源码解压目录
cd /huorq/nginx2/nginx-1.18.0
# 配置kafka_module
./configure --add-module=/path/to/ngx_kafka_module
# 重新编译
make 
make install
```



**启动报错**

> error while loading shared libraries: librdkafka.so.1: cannot open shared object file: No such file or directory 

```
# 加载so库
echo "/usr/local/lib" >> /etc/ld.so.conf
ldconfig
```



**修改nginx配置文件**

```

http {
  
    #配置kafka
    kafka;
    kafka_broker_list localhost:9092;

    server {
        listen       8888;
        server_name  localhost;
        location = /kafka/topic {
            # optional directive: kafka_partition [<partition-num> | auto]
            #
            # kafka_partition auto; # default value
            # kafka_partition 0;
            # kafka_partition 1;
            # 设置发送的top主题
            kafka_topic my_top_log;
        } 
    }
```



**控制台消费消息**

> 说明：页面比较简单，没有做用户登录等session以及记录用户和单位信息等，此处获取的用户id以及单位id都为随机生成的随机数

```
kafka-console-consumer.sh --bootstrap-server localhost:9092  --topic my_top_log --from-beginning
```

![image-20201123002430372](C:\Users\myang\AppData\Roaming\Typora\typora-user-images\image-20201123002430372.png)

![image-20201123002442223](C:\Users\myang\AppData\Roaming\Typora\typora-user-images\image-20201123002442223.png)



**nginx配置说明**

由于kafka安装在虚拟机，本地运气前端项目导致直接发送消息到虚拟机的nginx有问题，所以本地搭建了一个nginx，用于运行前端项目，所有的请求由本地nginx发送到服务器的nginx，去除了跨域问题。

```
# 本地nginx配置
server {
        
	listen 80 ;
	server_name 127.0.0.1; # 配置的https的域名
	location /kafka {
	    proxy_pass        http://192.168.255.50:8888/kafka;
        proxy_set_header  X-Real-IP  $remote_addr;
        proxy_read_timeout 300;
	}
```

