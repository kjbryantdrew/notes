版本情况



```
Centos: 7.5 1804
CDH: 6.1.0
```



服务器



```
whbdp01: mysql01
whbdp02: mysql02
whbdp03: none
whbdp04: none
```



![MySQL双主](https://cdn.jsdelivr.net/gh/windrivder/cdn/20200919/20200919153531MdfTLB.png)



whbdp01 上修改 mysql 配置



```
[mysqld]
# 添加如下配置
server_id=1
log-bin= mysql-bin
relay_log=mysql-relay-bin
binlog_format = mixed
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
auto-increment-offset=1
auto-increment-increment=2
```



whbdp02 上修改 mysql 配置



```
[mysqld]
# 添加如下配置
server_id=2
log-bin= mysql-bin
relay_log=mysql-relay-bin
binlog_format = mixed
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
auto-increment-offset=2
auto-increment-increment=2
```



注意：

mysql1 和 mysql 只有 server-id 不同和 auto-increment-offset 不同,其他必须相同。



部分配置项解释如下：



```bash
binlog_format= mixed：指定 mysql 的 binlog 日志的格式，mixed 是混合模式。
relay-log：开启中继日志功能
relay-log-index：中继日志清单
auto-increment-increment= 2：表示自增长字段每次递增的量，其默认值是 1。它的值应设为整个结构中服务器的总数，本案例用到两台服务器，所以值设为 2。
auto-increment-offset= 2：用来设定数据库中自动增长的起点(即初始值)，因为这两能服务器都设定了一次自动增长值 2，所以它们的起点必须得不同，这样才能避免两台服务器数据同步时出现主键冲突。
```



重启 whbdp01/02 上的 mysql 实例



```bash
systemctl restart mysqld
```



在 whbdp01 上创建授权用户，允许在 whbdp02 上主机上进行连接



```sql
mysql -uroot -p
Enter password:
mysql> grant replication slave on *.* to 'slaver'@'whbdp02' identified by 'slaver314159';
```



在 whbdp02 上配置，将 master 设置为 whbdp01



```sql
mysql -uroot -p
Enter password:
mysql> change master to master_host='whbdp01',master_user='slaver', master_password='slaver314159',master_log_file='mysql-bin.000001',master_log_pos=0;
Query OK, 0 rows affected, 2 warnings (0.13 sec)


mysql> start slave;
Query OK, 0 rows affected (0.00 sec)


mysql> show slave status
```



在 whbdp02 上创建授权用户，允许在 whbdp01 上主机上进行连接



```sql
mysql -uroot -p
Enter password:
mysql> grant replication slave on *.* to 'slaver'@'whbdp01' identified by 'slaver314159';
Query OK, 0 rows affected, 1 warning (0.00 sec)
```



在 whbdp01 上配置，将 master 设置为 whbdp01



```sql
mysql -uroot -p
Enter password:
mysql> change master to master_host='whbdp02',master_user='slaver', master_password='slaver314159',master_log_file='mysql-bin.000001',master_log_pos=0;
Query OK, 0 rows affected, 2 warnings (0.13 sec)


mysql> start slave;
Query OK, 0 rows affected (0.00 sec)


mysql> show slave status
```



注意



如果主库是已经存在的，则需要先将主库备份后在从库上恢复。

1. 然后在主库上使用命令：



```sql
mysql> show master status;
+------------------+----------+--------------+--------------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB         | Executed_Gtid_Set |
+------------------+----------+--------------+--------------------------+-------------------+
| mysql-bin.000002 |    42969 |              | mysql,information_schema |                   |
+------------------+----------+--------------+--------------------------+-------------------+
1 row in set (0.01 sec)
mysql>
```



2. 在从库上 change master 命令后的 master_log_pos=42969;