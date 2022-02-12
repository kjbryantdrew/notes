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



whbdp01:/home/dw/whbdp/software/confluent-kafka 有下载好的包

```bash
[root@whbdp01 confluent-kafka]# ls
archive.key     confluent.repo.origin       packages
confluent.repo  kafka-manager-1.3.3.15.zip  repodata
[root@whbdp01 confluent-kafka]#
```



配置 yum repo

```
# 路径
[root@whbdp02 yum.repos.d]# ls
cloudera-cdh6.repo      confluent.repo  epel-testing.repo
cloudera-manager6.repo  epel.repo       tmp
[root@whbdp02 yum.repos.d]# pwd
/etc/yum.repos.d


# epel.repo
[root@whbdp02 yum.repos.d]# cat epel.repo
[epel]
name=Extra Packages for Enterprise Linux 7
baseurl=http://yum-server:8000/epel7/
failovermethod=priority
enabled=1
gpgcheck=0
gpgkey=http://yum-server:8000/epel7/RPM-GPG-KEY-EPEL-7


# confluent-kafka.repo
[root@whbdp02 yum.repos.d]# cat confluent.repo
[Confluent.dist]
name=Confluent repository (dist)
baseurl=http://yum-server:8000/confluent-kafka
gpgcheck=0
gpgkey=http://yum-server:8000/confluent-kafka/archive.key
enabled=1
[Confluent]
name=Confluent repository
baseurl=http://yum-server:8000/confluent-kafka
gpgcheck=0
gpgkey=http://yum-server:8000/confluent-kafka/archive.key
enabled=
```



安装

```
yum clean all && yum install confluent-platform-oss-2.11
```



confluent-kafka 有三个版本，oss 开源社区版，enterprise 企业版，cloud 企业云版。我们这里安装 oss 开源社区版。

修改 kafka 配置

```
[root@whbdp01 kafka]# vim /etc/kafka/server.properties
# 修改
broker.id=4
zookeeper.connect=whbdp01:2181,whbdp02:2181,whbdp03:2181,whbdp04:2181
log.dirs=/data/kafka/kafka-logs
num.partitions=2
# 添加
default.replication.factor = 3
log.cleaner.enable=true
replica.fetch.max.bytes=5242880
message.max.byte=5242880
```



确保 zookeeper 已启动

```
zookeeper-server status
JMX enabled by default
Using config: /etc/zookeeper/conf/zoo.cfg
Mode: observer
```



启动

```
kafka-server-start  -daemon /etc/kafka/server.properties
```