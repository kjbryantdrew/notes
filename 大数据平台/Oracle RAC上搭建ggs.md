[TOC]

# 服务器信息

| ip           |      |      数据库版本       | ogg版本    | 软件包                               | 操作系统版本     | OGG安装路径 |
| ------------ | ---- | :-------------------: | ---------- | ------------------------------------ | ---------------- | ----------- |
| 172.16.8.12  | 源   | Oracle RAC 11.2.0.1.0 | 11.2.1.0.4 | ggs_AIX_ppc_112104_64bit.tar         | AIX 6.1          | /opt/ggs    |
| 172.16.8.161 | 目标 |         kafka         | 12.3.1.1.1 | OGG_BigData_Linux_x64_12.3.1.1.1.zip | CentOS  7.6.1810 | /opt/ogg    |

由于 ogg 源端为 AIX 上的部署的 Oracle RAC，所以部署方式与单机部署略有不同，总结如下：

1. 按照 Oracle RAC 的方式开启归档日志
2. 由于归档日志存储在 ASM 磁盘组中，ogg 进程无法连接到 ASM 实例读取归档日志
3. 配置 extract 进程时需要多配置一组 thread

# 源端配置

## 准备

root 用户

```bash
mkdir /opt/ggs
chown -R oracle:dba /opt/ggs
```

切换到 oracle 用户

```bash
pwd
/opt/ggs
tar xf ggs_AIX_ppc_112104_64bit.tar
```

oracle 创建复制用户 ggs

```bash
mkdir -p /data/app/oracle/data/ggs
mkdir -p /data/app/oracle/data/ggs
```

```sql
create tablespace ggstbs datafile '+DATA01/d012band/datafile/ggstlp.dbf' size 100M autoextend on;
create user ggs identified by ggs default tablespace ggstbs;
grant dba to ggs;
```

## 开启归档日志

> 注意此步骤需要对 Oracle RAC 进行启停，测试环境操作前与李军华等人进行了沟通确认。

查询归档日志

```sql
-- sqlplus / as sysdba
SQL> archive log list;
Database log mode           No Archive Mode
Automatic archival           Disabled
Archive destination           USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     23
Current log sequence           24
```

查询节点实例状态

```sql
SQL> select instance_name,host_name,status from gv$instance;

INSTANCE_NAME     HOST_NAME            STATUS
---------------- ------------------------------ ------------
d012band1         tscfs1                OPEN
d012band2         tscfs2                OPEN
```

切换到 grid 用户，将双节点关闭，再将节点1启动到 mount 状态

```bash
srvctl stop database -d d012band
srvctl stop database -d d012band -i d012band1 -o mount
```

切换到 oracle 用户，查询节点1数据库实例状态

```sql
SQL> select instance_name,status from v$instance;

INSTANCE_NAME     STATUS
---------------- ------------
d012band1         MOUNTED
```

修改数据库为归档模式，同时打开日志相关配置（最小日志，全量日志等）

```sql
SQL> alter system set log_archive_dest_1='location=+ARCH' scope=spfile;
SQL> alter database archivelog;
Database altered.
SQL> alter database force logging;
SQL> alter database add supplemental log data; # 最小量日记
SQL> alter database add supplemental log data (primary key, unique, foreign key) columns; # 开启全量日志
```

关闭节点1数据库

```sql
SQL> shutdown immediate;
ORA-01109: database not open
Database dismounted.
ORACLE instance shut down.
```

切换到 grid 用户，启动双节点数据库

```bash
srvctl start database -d d012band
srvctl status database -d d012band
```

**注意，如果 oracle 版本是11.2.0.4和12.1.0.2及以上，则需要进行如下配置：**

> 彬哥的文档解释是：为了更好的监视你使用 OGG，所以把 gg 绑定到 DB 中，只有设置了改参数为 true,才能使用 OGG 的一些功能。
>
> 推测其实就是方便 ogg 能够寻找到归档日志的路径，但由于目前行里面重新给出 oracle 版本是 11.2.0.3，所以此种配置无效，具体配置见后文。

```sql
SQL> ALTER SYSTEM SET ENABLE_GOLDENGATE_REPLICATION = TRUE SCOPE=BOTH;
```

## mgr 配置

切换到 oracle 用户，进入 ggs 主目录：

```bash
create subdirs
dblogin userid ggs password ggs;
edit param mgr
```

mgr 配置如下：

```
PORT 7809
DYNAMICPORTLIST 7810-7909
AUTORESTART EXTRACT *, RETRIES 5, WAITMINUTES 3
PURGEOLDEXTRACTS ./dirdat/*, usecheckpoint, minkeepdays 30
```

mgr 启动：

```bash
start mgr
```

## extract 配置

配置 extract 进程有两种方式：一是从当前时间开始抽取，二是根据给定的 SCN 进行抽取：

```bash
add extract ex01, tranlog, begin now
# 或者
add extract ex01, tranlog scn 15765157081, threads 2
# 此处根据 ogg 的日志提示，多配置了一个 thread

# 指定队列大小，本处设置表示100M
add exttrail ./dirdat/to extract ex01, megabytes 100
```

由于是在 Oracle RAC 环境下，需要特别处理 ogg 读取归档日志的问题，两种方式：

一是在 extract 进程的配置中添加 DBLOGREADER，配置如下：

```
extract ex01
DYNAMICRESOLUTION
USERID GGS, PASSWORD GGS
EXTTRAIL /opt/ggs/dirdat/to
DBOPTIONS ALLOWUNUSEDCOLUMN
TRANLOGOPTIONS DELOGREADER
TRANLOGOPTIONS ALTARCHIVELOGDEST primary instance d012band1 +ARCH/d012band/archivelog, ALTARCHIVELOGDEST primary instance d012band2 +ARCH/d012band/archivelog

table FNSONLD.*;
```

二是通过 ASM 用户连接 ASM 实例读取归档日志：

1. 查看 RAC 节点是否有 ASM 的监听注册，`lsnrctl services`；

2. 如果没有，需要用 grid 用户在 `$ORACLE_HOME/network/admin/listener.ora` 文件中添加静态注册，然后`reload listener`；
3. 用 oracle 用户编辑 `$ORACLE_HOME/network/admin/tnsnames.ora` 文件，使其用别名可连接 ASM 实例和数据库；
4. 最后配置文件可直接使用方式一的配置文件。

## pump 配置

```bash
add extract pump01, exttrailsource ./dirdat/to
add exttrail ./dirdat/kf, extract pump01
add rmttrail /opt/ogg/dirdat/kf, extract pump01
```

pump 进程配置如下：

```
extract pump01
passthru
dynamicresolution
userid ggs,password ggs
rmthost 172.16.8.161 mgrport 7809
rmttrail /opt/ogg/dirdat/kf

table FNSONLD.*;
```

## 生成表结构

```bash
# ./ggsci
GGSCI > edit param fnsonld_defgen
USERID ggs, PASSWORD ggs
defsfile ./dirdef/fnsonld.def
table FNSONLD.*;
# ./defgen paramfile ./dirprm/fnsonld_defgen.prm
```

最后将生成的 def 文件拷贝到目录端的 dirdef 中。

# 目标端配置

## 准备

root 下新建 ogg 用户：

```bash
useradd ogg
mkdir /opt/ogg
chown ogg:ogg  /opt/ogg
```

检查 Java 的安装并配置

```bash
rpm -ivh jdk-8u181-linux-x64.rpm

export LD_LIBRARY_PATH=/usr/java/jdk1.8.0_181-amd64/jre/lib/amd64/server:/usr/java/jdk1.8.0_181-amd64/jre/lib/amd64/jli:$LD_LIBRARY_PATH
export JAVA_HOME=/usr/java/jdk1.8.0_181-amd64
export PATH=/usr/java/jdk1.8.0_181-amd64/bin:$PATH
```

切换到 ogg 用户的 /opt/ogg，解压 ogg 包：

```bash
cd /opt/ogg
tar xvf ~/OGG_BigData_Linux_x64_12.3.1.1.1.tar 
```

> 注意，此安装包需使用版本库里面的安装包，针对攀商行数据有补丁处理。

ogg 用户下添加环境变量

```bash
export OGG_HOME=/opt/ogg
```

## mgr 配置

```bash
create subdirs
edit params mgr
```

```
PORT 7809
DYNAMICPORTLIST 7810-7909
AUTORESTART EXTRACT *,RETRIES 5,WAITMINUTES 3
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints, minkeepdays 30
```

配置 checkpoints：

```bash
GGSCI (dev95) 3> edit params ./GLOBALS
CHECKPOINTTABLE cbs.checkpoint
```

## replicat 配置

```bash
add replicat rekafka exttrail ./dirdat/kf, checkpointtable ogg.checkpoint
```

```
REPLICAT rekafka
sourcedefs /opt/ogg/dirdef/fnsonld.def
TARGETDB LIBFILE libggjava.so SET property=dirprm/kafka.props
REPORTCOUNT EVERY 1 MINUTES, RATE
GROUPTRANSOPS 10
MAXTRANSOPS 10

MAP FNSONLD.*, TARGET FNSONLD.* EXITPARAM "GBF";
CUSEREXIT ExitKanas.so CUSEREXIT
```

对接 kafka 的相关配置文件 kafka.props 和 json.properties 已提交到版本库。

# 启动

```bash
# 源端
start mgr
start ex01
start pump01

# 目标端
start mgr
start rekafka

./kafka-console-consumer --bootstrap-server 172.16.8.161:9092 --topic ogg_fnsonld --from-beginning --group ogg_pzh
./kafka-console-producer --broker-list 172.16.8.161:9092 --topic ogg_fnsonld
```