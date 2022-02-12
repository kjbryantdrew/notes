版本情况

```
Centos: 7.5 1804
CDH: 6.1.0
```



kafka

```
whbdp01: broker
whbdp02: broker
whbdp03: broker
whbdp04: broker
```



安装

```
wget https://www-us.apache.org/dist/kafka/2.1.0/kafka_2.12-2.1.0.tgz
cd /opt
tar zxvf kafka_2.12-2.1.0.tgz
ln -s kafka_2.12-2.1.0.tgz kafka
```



配置

```
vi /opt/kafka/config/server.properties
# 修改
broker.id=4 # 每台服务器修改为自己的, 每个broker id 不能一样
zookeeper.connect=whbdp01:2181,whbdp02:2181,whbdp03:2181,whbdp04:2181
log.dirs=/data/kafka/kafka-logs
num.partitions=2


# 添加
default.replication.factor = 3
log.cleaner.enable=true
replica.fetch.max.bytes=5242880
message.max.byte=5242880
```



所有服务器启动服务

```
cd /opt/kafka
/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/server.properties
```



测试

```
# 1) : 创建Topic来验证是否创建成功
./kafka-topics.sh --create --zookeeper whbdp01:2181 --topic testtopic --partitions 2 --replication-factor 2 
./kafka-topics.sh --create --zookeeper localhost:2181 --topic testtopic --partitions 1 --replication-factor 1
# 2): 查看topic 
./kafka-topics.sh --list --zookeeper whbdp01:2181
./kafka-topics.sh --list --zookeeper localhost:2181
# 3): 查看topic状态
./kafka-topics.sh --describe --zookeeper whbdp01:2181 --topic testtopic
./kafka-topics.sh --describe --zookeeper localhost:2181 --topic testtopic
# 4): 创建发布者
./kafka-console-producer.sh --broker-list whbdp01:9092 --topic testtopic
./kafka-console-producer.sh --broker-list localhost:9092 --topic testtopic
# 5): 创建订阅者
./kafka-console-consumer.sh --bootstrap-server whbdp01:9092 --topic testtopic --from-beginning
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic testtopic --from-beginning
```



```
[Confluent.dist]
name=Confluent repository (dist)
baseurl=http://packages.confluent.io/rpm/3.3/6
gpgcheck=0
gpgkey=http://packages.confluent.io/rpm/3.3/archive.key
enabled=1


[Confluent]
name=Confluent repository
baseurl=http://packages.confluent.io/rpm/3.3
gpgcheck=0
gpgkey=http://packages.confluent.io/rpm/3.3/archive.key
enabled=
```