版本情况

```
Centos: 7.5 1804
CDH: 6.1.0
```



服务器

```
whbdp01: none
whbdp02: redis
whbdp03: pika
whbdp04: none
```



切换至 pika 用户

```
cd /home/pika/pika-v3.0.6


cat /home/pika/pika-v3.0.6/conf/pika.conf
dump-path : /ssd/var/lib/pika/dump/
pidfile : /ssd/var/lib/pika/pika.pid
db-sync-path : /ssd/var/lib/pika/dbsync/
```



启动

```
/home/pika/pika-v3.0.6/bin/pika -c /home/pika/pika-v3.0.6/conf/pika.conf
```



配置 redis

```
# 修改redis.conf
database 256
```



启动

```
redis-server /etc/redis.conf
```