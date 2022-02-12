版本情况



```bash
Centos: 7.5 1804
CDH: 6.1.0
```



Impala



```bash
whbdp01: impalad
whbdp02: impalad
whbdp03: impalad, catalog server, statestore
whbdp04: impalad
```



所有服务器



```bash
yum install -y impala*
```



每台服务器上添加hdfs配置，如果是按照前述方式部署hadoop的话，配置中已经有了如下配置



```bash
cat /etc/hadoop/conf/hdfs-site.xml
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
```



每台服务器上创建相关文件夹



```bash
mkdir -p /var/run/hdfs-sockets/ && chown -R hdfs.hdfs /var/run/hdfs-sockets/


usermod -a -G hadoop impala
usermod -a -G hdfs impala


mkdir -p /data/impala/impalad
mkdir -p /data/impala/log/impalad/lineage
mkdir -p /data/impala/impala-minidumps
chown -R impala.impala /data/impal
```



每台服务器上配置 impala 参数



```bash
cat /etc/default/impala


IMPALA_CATALOG_SERVICE_HOST=whbdp03
IMPALA_STATE_STORE_HOST=whbdp03
IMPALA_STATE_STORE_PORT=24000
IMPALA_BACKEND_PORT=22000
IMPALA_LOG_DIR=/var/log/impala
IMPALA_CATALOG_ARGS=" -log_dir=${IMPALA_LOG_DIR} "
IMPALA_STATE_STORE_ARGS=" -log_dir=${IMPALA_LOG_DIR} -state_store_port=${IMPALA_STATE_STORE_PORT}"
IMPALA_SERVER_ARGS=" \
    -log_dir=${IMPALA_LOG_DIR} \
    -catalog_service_host=${IMPALA_CATALOG_SERVICE_HOST} \
    -state_store_port=${IMPALA_STATE_STORE_PORT} \
    -state_store_host=${IMPALA_STATE_STORE_HOST} \
    -enable_webserver=true -webserver_port=25000  \
    -mem_limit=219902325555 -max_log_files=10 -max_result_cache_size=100000 -max_cached_file_handles=20000 \
    -unused_file_handle_timeout_sec=21600 \
    -state_store_subscriber_port=23000 \
    -statestore_subscriber_timeout_seconds=30 \
    -scratch_dirs=/data/impala/impalad \
    -log_filename=impalad \
    -minidump_path=/data/impala/impala-minidumps \
    -max_minidumps=9 \
    -lineage_event_log_dir=/data/impala/log/impalad/lineage \
    -max_lineage_log_file_size=5000 \
    -queue_wait_timeout_ms=60000 \
    -disk_spill_encryption=false \
    -abort_on_config_error=true \
    -kudu_master_hosts=whbdp04 \
    -fe_service_threads=256 \
    -be_port=${IMPALA_BACKEND_PORT}"
ENABLE_CORE_DUMPS=false
# LIBHDFS_OPTS=-Djava.library.path=/usr/lib/impala/lib
# MYSQL_CONNECTOR_JAR=/usr/share/java/mysql-connector-java.jar
IMPALA_BIN=/usr/lib/impala/sbin
IMPALA_HOME=/usr/lib/impala
HIVE_HOME=/usr/lib/hive
#HBASE_HOME=/usr/lib/hbase
IMPALA_CONF_DIR=/etc/impala/conf
HADOOP_CONF_DIR=/etc/impala/conf
HIVE_CONF_DIR=/etc/impala/conf
# HBASE_CONF_DIR=/etc/impala/conf
```



每台服务器上复制配置文件



```bash
cp /etc/hadoop/conf/core-site.xml /etc/impala/conf/
cp /etc/hadoop/conf/hdfs-site.xml /etc/impala/conf/
```



所有服务器启动 impalad



```bash
service impala-server start
```



whbdp03 上启动 catalog server, statestore



```bash
service impala-state-store start
service impala-catalog start
```



测试



```bash
impala-shell
Starting Impala Shell without Kerberos authentication
Opened TCP connection to whbdp01:21000
Connected to whbdp01:21000
Server version: impalad version 3.1.0-cdh6.1.0 RELEASE (build 5efe077603091e405fbae2e5d68b851d6e790f59)
***********************************************************************************
Welcome to the Impala shell.
(Impala Shell v3.1.0-cdh6.1.0 (5efe077) built on Thu Dec  6 17:40:23 PST 2018)
When pretty-printing is disabled, you can use the '--output_delimiter' flag to set
the delimiter for fields in the same row. The default is '\t'.
***********************************************************************************
[whbdp01:21000] default> show databases;
Query: show databases
+------------------+----------------------------------------------+
| name             | comment                                      |
+------------------+----------------------------------------------+
| _impala_builtins | System database for Impala builtin functions |
| default          | Default Hive database                        |
+------------------+----------------------------------------------+
Fetched 2 row(s) in 0.17s
[whbdp01:21000] default>
```



- 查看 impalad: `http://whbdp03:25000/`
- 查看 StateStore: `http://whbdp03:25010/`
- 查看 Catalog: `http://whbdp03:25020/`