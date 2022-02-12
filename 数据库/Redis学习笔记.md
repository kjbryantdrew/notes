# Redis学习笔记

[TOC]

Redis 是完全开源免费的，遵守BSD协议，是一个高性能的key-value数据库

### 初识Redis
相比其他同类的key-value数据库，Redis具有如下优势：
- 持久化。可将数据保存在磁盘中，就是机器重启数据也不会丢失；
- 多样性。除了可以保存基础的key-value以外，还支持list、set、zset、hash等数据结构存储；
- 容灾性。支持数据备份，master-slaver模式的数据备份

Redis优势：
- 性能极高。110000次/s的写和81000次/s的读
- 丰富的数据类型。支持二进制案例的Strings、Lists、Hashes、Sets、Ordered Sets等数据类型
- 原子。Redis所有的操作都是原子性的，即要么执行成功，要么完全不执行。单个操作是原子性的，多个操作也支持事物，即原子性，通过MULTI和EXEC指令包起来
- 丰富的特征。Redis还支持publish/subscribe、通知、key过期等等特征

Redis与其他key-value存储有什么不同：
- Redis有着更为复杂的数据结构并且提供对他们的原子性操作，这是一个不同于其他数据库的进化路径。Redis的数据类型都是基于基本数据结构的同时对程序员透明，无需进行额外的抽象。
- Redis运行在内存中但是可以持久化到磁盘，所以在对不同数据集进行高速读写时需要权衡内存，因为数据量不能大于硬件内存。在内存数据库方面的另一个优点是，相比在磁盘上相同的复杂的数据结构，在内存中操作起来非常简单，这样Redis可以做很多内部复杂性很强的事情。同时，在磁盘格式方面他们是紧凑的以追加的方式产生的，因为他们并不需要进行随机访问。

> from [菜鸟教程](https://www.runoob.com/redis/redis-intro.html)

### Redis数据类型
Redis支持如下几种数据类型：
- string字符串
    1. key-value
    2. 二进制数据，意味着可保存jpg图片或者序列化对象等等
    3. 最大存储512M
    
- hash哈希

    1. 键值对集合
    2. string 类型的 field 和 value 的映射表
    3. 适合存储对象
    
    ```bash
    127.0.0.1:6379[2]> HMSET runoob field1 "Hello" field2 "World"
    OK
    127.0.0.1:6379[2]> HMGET runoob field1
    1) "Hello"
    127.0.0.1:6379[2]> HMGET runoob field2
    1) "World"
    127.0.0.1:6379[2]> HMGET runoob
    (error) ERR wrong number of arguments for 'hmget' command
    ```
    
    > 每个 hash 可以存储 2^32 -1 键值对（40多亿）
    
- list列表
    列表是简单的字符串列表，按照插入顺序排序
    ```bash
    127.0.0.1:6379[2]> LPUSH test redis
    (integer) 1
    127.0.0.1:6379[2]> LPUSH test mongodb
    (integer) 2
    127.0.0.1:6379[2]> RPUSH test rabbitmq
    (integer) 3
    127.0.0.1:6379[2]> RPUSH test db2
    (integer) 4
    127.0.0.1:6379[2]> LRANGE test 0 10
    1) "mongodb"
    2) "redis"
    3) "rabbitmq"
    4) "db2"
    ```
    
- set集合
    1. string 类型的无序集合
    2. 集合内元素唯一性
    3. 通过哈希表实现的，所以添加、删除、查找的复杂度都是`O(1)`
    
    ```bash
    127.0.0.1:6379[2]> SADD test redis
    (integer) 1
    127.0.0.1:6379[2]> SADD test rabbit
    (integer) 1
    127.0.0.1:6379[2]> SADD test mongodb
    (integer) 1
    127.0.0.1:6379[2]> SADD test mongodb
    (integer) 0
    127.0.0.1:6379[2]> SMEMBERS test
    1) "mongodb"
    2) "rabbit"
    3) "redis"
    ```

- zset(sort set)有序集合

    1. zset和set一样，是string类型元素的集合，且不允许重复
    2. 不同的是每个元素都会关联一个double类型的分数，Redis正是通过分数来为集合中的成员进行从小到大的排序
    3. zset的成员是唯一的,但分数(score)却可以重复
    ```bash
    127.0.0.1:6379[2]> zadd runoob 0 redis
    (integer) 1
    127.0.0.1:6379[2]> zadd runoob 0 mongodb
    (integer) 1
    127.0.0.1:6379[2]> zadd runoob 0 rabitmq
    (integer) 1
    127.0.0.1:6379[2]> zadd runoob 0 rabitmq
    (integer) 0
    127.0.0.1:6379[2]> ZRANGEBYSCORE runoob 0 1000
    1) "mongodb"
    2) "rabitmq"
    3) "redis"
    ```

### Redis订阅
Redis 发布订阅(pub/sub)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。
Redis 客户端可以订阅任意数量的频道。
```bash
127.0.0.1:6379> SUBSCRIBE channel1
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "channel1"
3) (integer) 1
1) "message"
2) "channel1"
3) "Redis is a great caching technique"
1) "message"
2) "channel1"
3) "Learn redis by runoob.com"
```
```bash
(integer) 1
127.0.0.1:6379> PUBLISH channel1 "Learn redis by runoob.com"
(integer) 1
127.0.0.1:6379>
```

### Redis事务

Redis事务一次执行多个命令，并且具有如下特征：

- 批量操作在发送`EXEC`命令前都会被放入缓存队列
- 收到`EXEC`命令后进入事务执行，并且值得注意的是，队列中任一任务失败不会影响到其他任务的运行
- 在事务执行过程中，其他客户端提交的命令请求不会被添加到队列中

一个事务从开始到结束经历如下三个阶段：

1. 开始事务
2. 命令入队
3. 执行事务

事务以`MULTI`标记开始，以`EXEC`标记结束以及触发事务运行

```bash
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> SET book-name "Mastering C++ in 21 days"
QUEUED
127.0.0.1:6379> GET book-name
QUEUED
127.0.0.1:6379> SADD tag "C++" "Programming" "Mastering Series"
QUEUED
127.0.0.1:6379> SMEMBERS tag
QUEUED
127.0.0.1:6379> EXEC
1) OK
2) "Mastering C++ in 21 days"
3) (integer) 3
4) 1) "Mastering Series"
   2) "Programming"
   3) "C++"
```

> 单个的Redis命令的执行是具有原子性的，但是Redis并没有在事务上增加任何维持原子性的机制，所以Redis的事务不具有原子性

### Redis GEO

Redis GEO 主要用于存储地理位置信息，并对存储的信息进行操作，该功能在 Redis 3.2 版本新增。
Redis GEO 操作方法有：

- geoadd：添加地理位置的坐标
- geopos：获取地理位置的坐标
- geodist：计算两个位置之间的距离
- georadius：根据用户给定的经纬度坐标来获取指定范围内的地理位置集合
- georadiusbymember：根据储存在位置集合里面的某个地点获取指定范围内的地理位置集合
- geohash：返回一个或多个位置对象的 geohash 值

### Redis数据备份与恢复

用`Save`命令用于创建当前数据库备份

> 该命令将在redis安装目录下创建`dump.rbd`

如果需要恢复数据，只需要将`dump.rbd`移动到安装路径而后启动服务即可，查看Redis安装路径

```bash
127.0.0.1:6379> CONFIG GET dir
1) "dir"
2) "/var/lib/redis"
```

`BGSAVE`命令将在后台执行备份任务

### Redis安全

通过配置文件设置密码，使客户端连接到Redis服务时需要密码验证，提升安全性

```bash
# 查看是否设置密码验证
127.0.0.1:6379> CONFIG get requirepass
1) "requirepass"
2) ""
# 修改密码参数
127.0.0.1:6379> CONFIG set requirepass "runoob"
OK
127.0.0.1:6379> CONFIG get requirepass
1) "requirepass"
2) "runoob"
# 密码验证
127.0.0.1:6379> AUTH "runoob"
OK
127.0.0.1:6379> SET mykey "Test value"
OK
127.0.0.1:6379> GET mykey
"Test value"
```
