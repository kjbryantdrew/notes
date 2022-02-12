版本情况

```
Centos: 7.5 1804
CDH: 6.1.0
```

HDFS 角色部署方案

```
whbdp01: DataNode, NameNode, DFSZKFailoverController(zkfc)
Whbdp02: DataNode, SecondaryNameNode, DFSZKFailoverController(zkfc)
Whbdp03: DataNode
Whbdp04: DataNode
```

每台服务器均安装 Java1.8

```
rpm -ivh jdk-8u181-linux-x64.rpm
vi /etc/bashrc
export JAVA_HOME=/usr/java/jdk1.8.0_181-amd64
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$JAVA_HOME/bin:$PATH
```

每台服务器上创建 hadoop/hdfs 用户，增加管理员权限

```bash
useradd hadoop
passwd hadoop
usermod -G root hadoop
usermod -G hadoop hdfs

useradd hdfs
passwd hdfs
usermod -G root hdfs

visudo
root ALL=(ALL) ALL
hadoop ALL=(ALL) ALL
hdfs ALL=(ALL) AL
```

whbdp01 安装依赖

```bash
yum install -y hadoop hadoop-hdfs-datanode hadoop-hdfs-namenode [hadoop-hdfs-zkfc hadoop-hdfs-nfs3 hadoop-httpfs]
yum install -y hadoop-yarn hadoop-yarn-nodemanager
```


配置文件

```bash
cat /etc/hadoop/conf/core-site.xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://whbdp01:8020</value>
    </property>
    <!-- HTTPFS proxy user setting -->
  <property>
    <name>hadoop.proxyuser.httpfs.hosts</name>
    <value>*</value>
  </property>
  <property>
    <name>hadoop.proxyuser.httpfs.groups</name>
    <value>*</value>
  </property>
    <!-- end -->
<!-- for impala -->
  <property>
    <name>dfs.datanode.hdfs-blocks-metadata.enabled</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.client.read.shortcircuit</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.domain.socket.path</name>
    <value>/var/run/hdfs-sockets/dn._PORT</value>
  </property>
  <property>
    <name>dfs.client.file-block-storage-locations.timeout.millis</name>
    <value>10000</value>
  </property>
</configuration>
```



```
cat /etc/hadoop/conf/hdfs-site.xml
<property>
        <name>dfs.namenode.name.dir</name>
    <value>file:///data/hadoop-hdfs/cache/hdfs/dfs/name</value>
</property>
<property>
        <name>dfs.namenode.checkpoint.dir</name>
        <value>file:///data/hadoop-hdfs/cache/hdfs/dfs/namesecondary</value>
</property>
<property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///data/hadoop-hdfs/cache/hdfs/dfs/data</value>
</property>
<property>
    <name>hadoop.tmp.dir</name>
    <value>/data/hadoop-hdfs/cache/hdfs</value>
</property>
<!-- CLOUDERA-BUILD: CDH-64745. -->
<property>
    <name>cloudera.erasure_coding.enabled</name>
    <value>true</value>
</property>
<property>
    <name>dfs.client.read.shortcircuit</name>
    <value>true</value>
 </property>
 <property>
    <name>dfs.client.file-block-storage-locations.timeout.millis</name>
    <value>10000</value>
 </property>
 <property>
    <name>dfs.domain.socket.path</name>
    <value>/var/run/hadoop-hdfs/dn._PORT</value>
 </property>
 <property>
    <name>dfs.datanode.hdfs-blocks-metadata.enabled</name>
    <value>true</value>
 </property>
```



单机版，复制因子设置为 1

```
<configuration>
        <property>
            <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```



创建文件

```
mkdir -p /data/hadoop-hdfs
chown -R hdfs.hadoop /data/hadoop-hdfs
chmod a+w /data/hadoop-hdfs


cat /etc/hadoop/conf/hadoop-env.sh
export JAVA_HOME=/usr/java/jdk1.8.0_181-amd64


cat /etc/hadoop/conf/yarn-env.sh
JAVA_HOME=/usr/java/jdk1.8.0_181-amd64
JAVA_HEAP_MAX=-Xmx1000m
export JAVA_HOME JAVA_HEAP_MAX


cat /etc/hadoop/conf/yarn-site.xml
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>whbdp01</value>
</propert
```



初始化 HDFS

```
su - hadoop
hdfs namenode -format
```



启动 namenode

```
systemctl start  hadoop-hdfs-namenode
```



启动 datanode

```
systemctl start  hadoop-hdfs-datanode
```



whbdp02 安装依赖

```
yum install -y hadoop hadoop-hdfs-datanode hadoop-hdfs-secondarynamenode [hadoop-hdfs-namenode ，hadoop-hdfs-zkfc]
yum install -y hadoop-yarn hadoop-yarn-nodemanager hadoop-yarn-resourcemanager


# 复制 whbdp01 /etc/hadoop/conf 下的配置
```



初始化 HDFS

```
hdfs namenode -format
systemctl start hadoop-hdfs-secondarynamenode
```



启动 secondary namenode

```
systemctl start hadoop-hdfs-secondarynamenode
```



启动 datanode



```
systemctl start  hadoop-hdfs-datanode
```



whbdp03 安装依赖



```
yum install -y hadoop hadoop-hdfs-datanode
yum install -y hadoop-yarn hadoop-yarn-nodemanager hadoop-yarn-resourcemanager


# 复制 whbdp01 /etc/hadoop/conf 下的配置
```



启动 datanode



```
systemctl start  hadoop-hdfs-datanode
```



whbdp04 安装依赖



```
yum install -y hadoop hadoop-hdfs-datanode
yum install -y hadoop-yarn hadoop-yarn-nodemanager


# 复制 whbdp01 /etc/hadoop/conf 下的配置
```



启动 datanode



```
systemctl start  hadoop-hdfs-datanode
```



访问 web: `http://whbdp01:9870/`
