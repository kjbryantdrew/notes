版本情况



```
Centos: 7.5 1804
CDH: 6.1.0
```



Zookeeper



```
whbdp01: observer
whbdp02: zookeeper server
whbdp03: zookeeper server
whbdp04: zookeeper server
```



所有服务器上安装



```
yum install -y zookeeper
```



每台服务器配置



```
cat /etc/zookeeper/conf/zoo.cfg
maxClientCnxns=50
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
dataDir=/ssd/var/lib/zookeeper/data
# the port at which the clients will connect
clientPort=2181
# the directory where the transaction logs are stored.
dataLogDir=/ssd/var/lib/zookeeper/log
server.0=whbdp02:2888:3888
server.1=whbdp03:2888:3888
server.2=whbdp04:2888:3888
server.3=whbdp01:2888:3888:observer


mkdir -p /ssd/var/lib/zookeeper/data
mkdir -p /ssd/var/lib/zookeeper/log
chown -R zookeeper.zookeeper /ssd/var/lib/zookeeper
```



whbdp01 中添加观察者角色



```
# /etc/zookeeper/conf/zoo.cfg
peerType=observer
```



根据上面 server.id 后面 id 值分别在对应服务器上执行 init, 值为 id



```
zookeeper-server-initialize --myid=0
zookeeper-server-initialize --myid=1
zookeeper-server-initialize --myid=2
zookeeper-server-initialize --myid=3
```



每台服务器上启动



```
zookeeper-server start
```



查看状态



```
zookeeper-server status
```