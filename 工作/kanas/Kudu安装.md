版本情况



```
Centos: 7.5 1804
CDH: 6.1.0
```



Kudu



```
whbdp01: kudu tablet server
whbdp02: kudu tablet server
whbdp03: kudu tablet server
whbdp04: kudu tablet server, kudu master
```



所有服务器安装



```
yum install -y kudu*
```



Kudu 使用 ntp 同步



```
systemctl disable chronyd
systemctl stop chronyd
```



所有服务器配置 tserver



```
mkdir -p /data/kudu/tserver
mkdir -p /data/kudu/log/tserver
chown -R kudu.kudu /data/kudu


cat /etc/kudu/conf/tserver.gflagfile
# Do not modify these two lines. If you wish to change these variables,
# modify them in /etc/default/kudu-tserver.
--fromenv=rpc_bind_addresses
--fromenv=log_dir
--fs_wal_dir=/data/kudu/tserver
--fs_data_dirs=/data/kudu/tserver
--tserver_master_addrs=whbdp04:7051
--default_num_replicas=3
 # 180GB
--memory_limit_hard_bytes=193273528320
 # default is 80
--memory_limit_soft_percentage=80
--block_cache_capacity_mb=55296
--memory_pressure_percentage=60


# 说明：
# memory_pressure_percentage default is 60% 
# block_cache_capacity_mb = (50% * --memory_pressure_percentage) * —memory_limit_hard_byte
```



whbdp04 配置 master



```
mkdir -p /data/kudu/master
mkdir -p /data/kudu/log/master
chown -R kudu.kudu /data/kudu


cat /etc/kudu/conf/master.gflagfile
# Do not modify these two lines. If you wish to change these variables,
# modify them in /etc/default/kudu-master.
--fromenv=rpc_bind_addresses
--fromenv=log_dir
--fs_wal_dir=/data/kudu/master
--fs_data_dirs=/data/kudu/maste
```



启动



```
systemctl start kudu-tserver
systemctl start kudu-master
```



测试



```
http://whbdp04:8051
```



数据存储在 kudu



```
CREATE TABLE testdb.my_table_stored_as_kudu
(
  id BIGINT,
  name STRING,
  PRIMARY KEY(id) 
)PARTITION BY HASH PARTITIONS 16 STORED AS KUDU;


CREATE TABLE testdb.my_table_stored_as_kudu2
(
  id BIGINT,
  method STRING,
  dv DOUBLE,
  fv FLOAT,
  PRIMARY KEY(id) 
)PARTITION BY HASH PARTITIONS 16 STORED AS KUDU;


insert into dbtest.my_table_stored_as_kudu(id, name) values(1, 'mike');
select * from dbtest.my_table_stored_as_kudu;


insert into testdb.my_table_stored_as_kudu(id, name) values(1, 'mike'
```



数据存储在 hive parquet



```
CREATE TABLE dbtest.my_table_stored_as_parquet
(
  id BIGINT,
  name STRING
) STORED AS PARQUET;


insert into dbtest.my_table_stored_as_parquet(id, name) values(1, 'mike');
select * from dbtest.my_table_stored_as_parquet;
```



解除 kudu 用户的线程限制



```
vi /etc/security/limits.d/90-nproc.conf


kudu soft nproc unlimited
impala soft nproc unlimite
```