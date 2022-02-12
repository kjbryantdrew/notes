### 分区表数据集群间同步
1、rename表

2、修改分区字段名，因为hive不会读取以"_"开始的字段，因此将分区字段中的下划线移动到字段名后

3、回写数据

4、hadoop同步文件
```bash
# 同步parquet文件
hadoop distcp -skipcrccheck -update(-overwrite) -m 20 hdfs://160.160.9.43/user/hive/warehouse/cbs.db /user/hive/warehouse/cbs.db同步表
```

5、目标数据库建表

6、刷新Hive表分区
```bash
[root@big-data02 ~]# beeline
WARNING: log4j.properties is not found. HADOOP_CONF_DIR may be incomplete.
WARNING: log4j.properties is not found. HADOOP_CONF_DIR may be incomplete.
WARNING: log4j.properties is not found. HADOOP_CONF_DIR may be incomplete.
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/lib/hive/lib/log4j-slf4j-impl-2.8.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/usr/lib/zookeeper/lib/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]
Beeline version 2.1.1-cdh6.1.0 by Apache Hive

beeline> !connect jdbc:hive2://big-data04:10001
Connecting to jdbc:hive2://big-data04:10001

Enter username for jdbc:hive2://big-data04:10001:
Enter password for jdbc:hive2://big-data04:10001:

Connected to: Apache Hive (version 2.1.1-cdh6.1.0)
Driver: Hive JDBC (version 2.1.1-cdh6.1.0)
Transaction isolation: TRANSACTION_REPEATABLE_READ
0: jdbc:hive2://big-data04:10001> MSCK REPAIR TABLE preset.cbs_d_account_20190728;

No rows affected (0.429 seconds)
0: jdbc:hive2://big-data04:10001>
```

7、刷新impala
```sql
invalidate metadata
```
