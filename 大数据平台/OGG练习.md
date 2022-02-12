### 1. oracle字符集

操作：
修改源库为 AMERIVCAN_AMERICA.ZHS16GBK字符集，测试传送数据的编码。

测试内容：
通过设置replicat环境变量、退出参数等，转换编码为UTF8

```sql
# 关闭数据库
SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.

# 以mount方式启动数据库
SQL> startup mount;
ORACLE instance started.

Total System Global Area 4999610368 bytes
Fixed Size                  8630952 bytes
Variable Size            1040190808 bytes
Database Buffers         3942645760 bytes
Redo Buffers                8142848 bytes
Database mounted.

# 设置session
SQL> alter system enable restricted session;

System altered.

SQL> alter system set job_queue_processes=0;

System altered.

SQL> alter system set aq_tm_processes=0;

System altered.

# 启动数据库
SQL> alter database open;

Database altered.

# 旧的编码为UTF8，直接修改为ZHS16GBK会有如下提示
SQL> ALTER DATABASE CHARACTER SET ZHS16GBK;
ALTER DATABASE CHARACTER SET ZHS16GBK
*
ERROR at line 1:
ORA-12712: new character set must be a superset of old character set
# 提示新字符集必须为旧字符集的超集

# 可以跳过超集的检查做更改
SQL>  ALTER DATABASE character set INTERNAL_USE ZHS16GBK;

Database altered.

# 重新启动数据库
SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.

SQL> startup;
ORACLE instance started.

Total System Global Area 4999610368 bytes
Fixed Size                  8630952 bytes
Variable Size            1040190808 bytes
Database Buffers         3942645760 bytes
Redo Buffers                8142848 bytes
Database mounted.
Database opened.
SQL>
```

目标端处理编码转换

编辑replicate，增加编码说明
```
SOURCECHARSET zhs16gbk
```

最后，当编码转换成功生效时，会在replicat的日志文件中发现如下日志输出
```
Performing implicit conversion of column data from character set zhs16gbk to UTF-8.
```

### 2. oracle表结构变更

操作：
ogg正常运行时，修改oracle源库表结构

测试内容：
1. 观察extract、pump、replicat进程表现
2. replicat报错排除
3. 测试修改表结构时的正常操作流程
```bash
# 目标端停止复制replcat进程
GGSCI (jason) 64> stop redemo

Sending STOP request to REPLICAT REDEMO ...
Request processed.

# 目标端修改表结构

# 源端停止extrac抽取进程
GGSCI (leson as ggadm@orcl) 50> stop extdemo

Sending STOP request to EXTRACT EXTDEMO ...
Request processed.

# 启动目标端replicat
GGSCI (jason) 67> start redemo

Sending START request to MANAGER ...
REPLICAT REDEMO starting

# 启动源端extract
GGSCI (leson as ggadm@orcl) 52> start extdemo

Sending START request to MANAGER ...
EXTRACT EXTDEMO starting
```

最后经测试，在OGG19中，是因为抽取extract进程没有成功解析新字段，只需要重启源段extract进程即可

上诉需要手动重启extract是因为没有开启ddl同步，开启ddl步骤参考**OGG安装手册**

值得注意的是，如果在安装ogg的生成表结构定义文件的步骤中，包含了a表，那么a表不支持ddl，即是说：a表定义在了表结构文件中，同时开启了ddl，会导致replicat进程有如下所示的错误：
```
2021-02-08 12:45:55  ERROR   OGG-00513  Table with SOURCEDEF cannot have DDL operations (table SN.ACTIVITY_FORM). Either remove SOURCEDEF or filter out table from DDL operations.
```

同时上文也提示了解决方法：
- 从def文件中移除a表
- 从ddl配置中移除a表


### 3. 强制数据切换

操作：
ogg正常运行时，修改oracle源库表结构，在源库大量进行数据修改操作，累积数据

测试内容：
1. 尝试重新启动数据同步
2. 忽略错误数据，使用etrollover切换数据文件，忽略全部错误

### 4. ogg数据重放

操作：
ogg正常运行，在源端进行数据操作

测试内容：
1. 使用logdump查看trail文件数据，了解rba号、scn号、数据内容查看，使用filter进行数据过滤
2. 修改replicat extseqno、 extrba号，尝试重放数据
```
trail文件是一条数据一条数据追加在文件里面，每条数据在文件里的起始位置，就是他的rba号

数据在文件里面是二进制存储的，每条数据都有一定的大小，对于第一条数据，他的起始位置在文件的开头，rba就认为是0，假设第一条数据有1000bytes

那么对于第二条数据，文件位置开头就是以1001开始，rba号就是1001

rba号就是数据在文件里面的二进制位置序号

搭ogg的时候不是还要查看oracle的scn然后还要配置到配置文件里面去个嘛

scn号，system change number，是oracle记录的一个顺序号，记录某个事务发生的顺序。从数据库原理来看，数据库本身是指数据文件，对数据库进行操作，就是对数据文件进行操作，以oracle来说就是操作oradata下的dbf文件。数据文件发送变更时，会修改数据文件，某次变更发生的顺序号就是scn号

为了支持事务、回滚等操作，会引入undo和redo log文件，用于事务控制，以及存放最近发生的数据文件变更。redo log里面会记录有最近多次的事务数据，指定scn号是为了指定从某次变更开始进行抽取，主要目的是在oracle to oracle的同步中，控制初始化数据的准确性。一般比较大的数据库进行初始化时，是从源库按scn号导出全库数据，直接应用到目标库。再ogg从指定scn号进行抽取，这样两边的数据才能保持一致

发送到kafka实际上就没有这么精确的要求了

用logdum查看trail文件，消息头上就会显示本条数据rba号是多少，消息有多大。用pos  rba号就能跳转到文件指定位置，实际应该就是调用了lseek函数，指定了文件偏移量。
```

### 5. zookeeper集群服务

操作：
分别搭建3、4、5节点的zookeeper集群，再后台kill服务

测试内容：
测试最大可容忍节点失效数量，理解zookeeper选举、投票机制，N/2 + 1防脑裂等内容

### 6. kafka集群服务

操作：
在正常运行的kafka集群，手动kill一个节点

测试内容：
测试节点失效情况下，Kafka提供服务能力


### 7. kafka replication factor策略

操作：
新建3个topic，replication分别设置为1，2，3，再手动kill服务节点

测试内容：
不同备份策略下，topic分区失败容忍度及对外服务能力


### 8. kafka读写练习

操作：
1. 使用kafka命令行工具创建、删除topic，了解partition、replcator机制
2. 使用python 创建producer向kafka写入数据，了解key、partition参数如何使用
3. 测试不指定key和partition时，数据如何确认写入的分区
4. 指定key或partition的方式，写入Kafka 分区
5. 使用python 创建consumer从Kafka读取数据，了解如何指定partition、offset、group等
6. 通过kafka命令行工具，查看group信息，重置offst至指定位置操作
7. 多进程使用相同group读取同一topic，了解Kafka group partition的分配策略
