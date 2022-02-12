# OGG+PIPELINE+KAFKA+TUNNEL搭建实时总账


## 1. 安装源端OGG

**ATTENTION**: Oracle必须是归档状态！

### 1.1 开启源端ORACLE归档日志

#### 1.1.1 查看归档状态

```bash
SQL> archive log list;
Database log mode              No Archive Mode
Automatic archival             Disabled          #此时为非归档
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     288
Current log sequence           290
```

#### 1.1.2 开启主动归档

```sql
SQL> alter system set log_archive_start=true scope=spfile;

System altered.
```

#### 1.1.3 重启并开启归档

```sql
sql> shutdown immediate;  
sql> startup mount;    #打开控制文件，不打开数据文件  
sql> alter database archivelog; #将数据库切换为归档模式  
sql> alter database open;   #将数据文件打开  
```

#### 1.1.4 查看此时归档状态

```bash
SQL> archive log list;
Database log mode              Archive Mode
Automatic archival             Enabled   #归档已开启
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     288
Next log sequence to archive   290
Current log sequence           290
```

### 1.2 开启源端ORACLE全量日志
#### 1.2.1 查看当前数据库日志状态

```bash
SQL> SELECT supplemental_log_data_min min,supplemental_log_data_pk pk,supplemental_log_data_ui ui,supplemental_log_data_fk fk,supplemental_log_data_all allc FROM v$database;

MIN      PK  UI  FK  ALL
-------- --- --- --- ---
NO       NO  NO  NO  NO        #最小化日志

```

#### 1.2.2 开启全量日志

```bash
SQL> alter database add supplemental log data(all,primary key,unique,foreign key) columns;
Database altered.
```

#### 1.2.3 查看当前数据库日志状态

```bash
SQL> SELECT supplemental_log_data_min min,supplemental_log_data_pk pk,supplemental_log_data_ui ui,supplemental_log_data_fk fk,supplemental_log_data_all allc FROM v$database;

MIN      PK  UI  FK  ALL
-------- --- --- --- ---  
IMPLICIT YES YES YES YES        #全量日志
```
#### 1.2.4 将ogg_replica置为true

```bash
SQL> alter system set enable_goldengate_replication=true;
# Oracle 11.2.0.1.1 没有该参数

System altered
```

### 1.3 创建OGG用户
**ATTENTON**: 不建议将ogg安装在oracle用户下

```bash
[root@whbdp01 ~] useradd ogg
[root@whbdp01 ~] passwd ogg
Changing password for user ogg.
New password:
BAD PASSWORD: The password is shorter than 8 characters
Retype new password:
passwd: all authentication tokens updated successfully.
```

### 1.4 安装OGG
#### 1.4.1 解压安装包
操作系统版本： `Centos 7.4`
安装包： `123014_fbo_ggs_Linux_x64_shiphome.zip`

```bash
[oracle@whbdp01 ogg]$ ls
121202_fbo_ggs_Linux_x64_shiphome.zip
fbo_ggs_Linux_x64_shiphome
OGG-12.1.2.0.2-README.txt
OGG_WinUnix_Rel_Notes_12.1.2.0.2.pdf
perl5
[oracle@whbdp01 ogg]$ pwd
/opt/ogg
```

#### 1.4.2 切换到oracle用户
* ogg目录权限如下
```bash
drwxr-xr-x 7 ogg    oinstall 323 Jun 13 10:17 ogg
```

#### 1.4.3 修改配置文件

```bash
[oracle@dev94 response]$ pwd
/home/oracle/software/ggs/fbo_ggs_Linux_x64_shiphome/Disk1/response
[oracle@dev94 response]$  vi oggcore.rsp
INSTALL_OPTION=ORA11g
SOFTWARE_LOCATION=/opt/ogg/ogg
UNIX_GROUP_NAME=oinstall
```

#### 1.4.4 安装ogg

```bash
[oracle@whbdp01 Disk1]$  ./runInstaller  -silent -responseFile /opt/ogg/software/fbo_ggs_Linux_x64_shiphome/Disk1/response/oggcore.rsp
Starting Oracle Universal Installer...

Checking Temp space: must be greater than 120 MB.   Actual 204659 MB    Passed
Checking swap space: must be greater than 150 MB.   Actual 91350 MB    Passed
Preparing to launch Oracle Universal Installer from /tmp/OraInstall2019-06-13_11-21-30AM. Please wait ...[oracle@whbdp01 Disk1]$ You can find the log of this install session at:
 /ssd/oracle/inventory/logs/installActions2019-06-13_11-21-30AM.log
The installation of Oracle GoldenGate Core was successful.
Please check '/ssd/oracle/inventory/logs/silentInstall2019-06-13_11-21-30AM.log' for more details.
Successfully Setup Software.
```

* 安装成功后：

```bash
demo_ora_pk_befores_insert.sql   prvtlmpg_uninstall.sql
demo_ora_pk_befores_updates.sql  remove_seq.sql
diagnostics                      replicat
diretc                           retrace
dirout                           role_setup.sql
dirsca                           sequence.sql
emsclnt                          server
extract                          SQLDataTypes.h
freeBSD.txt                      sqlldr.tpl
ggcmd                            srvm
ggMessage.dat                    tcperrs
ggparam.dat                      ucharset.h
ggsci                            ulg.sql
healthcheck                      UserExitExamples
help.txt                         usrdecs.h
install                          zlib.txt
[ogg@whbdp01 ogg]$ pwd
/opt/ogg/ogg
[ogg@whbdp01 ogg]$
```

### 1.5 在源端ORACLE创建ogg用户


```bash
SQL> create tablespace oggtbs datafile '/opt/ogg/data/tablespace/db01.dbf'
size 5G autoextend on;

Tablespace created.

SQL> create user ogg identified by ogg default tablespace oggtbs;

User created.

SQL> grant dba to ogg;

Grant succeeded.
```

### 1.6 配置源端ogg

#### 1.6.1 切换到ogg用户

```bash
[ogg@whbdp01 ogg]$ ./ggsci
./ggsci: error while loading shared libraries: libnnz11.so: cannot open shared object file: No such file or directory  #出现此报错信息，需加环境变量
```
**ATTENTION**： 环境变量中的值根据实际oracle环境变量填写
```bash
[ogg@whbdp01 ogg]$ cat ggs.env
export ORACLE_BASE=/ssd/oracle
export ORACLE_HOME=$ORACLE_BASE/product/11.2.4
export ORACLE_SID=bdpapp           #sid根据实际情况填写，且应始终保持一致
export PATH=$PATH:$ORACLE_HOME/bin
export LD_LIBRARY_PATH=$ORACLE_HOME/lib
alias ggsci='rlwrap ./ggsci'    #生产环境不建议使用rlwrap
```

#### 1.6.2 创建目录

```bash
GGSCI (whbdp01) 2> create subdirs

Creating subdirectories under current directory /data/ogg/ogg

Parameter file                 /data/ogg/ogg/dirprm: created.
Report file                    /data/ogg/ogg/dirrpt: created.
Checkpoint file                /data/ogg/ogg/dirchk: created.
Process status files           /data/ogg/ogg/dirpcs: created.
SQL script files               /data/ogg/ogg/dirsql: created.
Database definitions files     /data/ogg/ogg/dirdef: created.
Extract data files             /data/ogg/ogg/dirdat: created.
Temporary files                /data/ogg/ogg/dirtmp: created.
Credential store files         /data/ogg/ogg/dircrd: created.
Masterkey wallet files         /data/ogg/ogg/dirwlt: created.
Dump files                     /data/ogg/ogg/dirdmp: created.

```

#### 1.6.3 配置全局变量

```bash
GGSCI (dev94) 2> dblogin userid ogg password ogg;  #1.5中创建的用户
Successfully logged into database.
GGSCI (dev94 as ggs@bdpapp) 4> edit param ./globals
oggschema ogg
```

#### 1.6.4 配置管理器mgr

```bash
GGSCI (dev94 as ggs@bdpapp) 5> edit param mgr
PORT 7809
DYNAMICPORTLIST 7810-7909
AUTORESTART EXTRACT *,RETRIES 5,WAITMINUTES 3
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints, minkeepdays 30
LAGREPORTHOURS 1
LAGINFOMINUTES 30
LAGCRITICALMINUTES 45
GGSCI (dev94 as ggs@bdpapp) 7> start mgr
Manager started.
GGSCI (dev94 as ggs@bdpapp) 8> info mgr
Manager is running (IP port dev94.7809, Process ID 50173).
GGSCI (dev94 as ggs@bdpapp) 9>
```

#### 1.6.4 配置extract
##### 1. 查看scn

```bash
select dbms_flashback.get_system_change_number from dual;

GET_SYSTEM_CHANGE_NUMBER 
------------------------ 
7076371 
```
当前scn号为 7076371

##### 2.  添加extract

```bash
GGSCI (whbdp01) 2> add extract extcbs tranlog scn 7076371  #按实际情况填写
EXTRACT added.

GGSCI (whbdp01) 3> add exttrail ./dirdat/cb,extract extcbs,megabytes 100
EXTTRAIL added.

GGSCI (whbdp01) 4> edit params extcbs
extract extcbs
userid ogg,password ogg -- 根据实际情况填写
exttrail ./dirdat/cb
table cbs.*;
```
#### 1.6.4 配置pumps, pumpcbs, 推送给kafka端

```bash
GGSCI (whbdp01) 6> add extract pumpcbs, exttrailsource ./dirdat/cb
EXTRACT added.
GGSCI (whbdp01) 8> add rmttrail /opt/ogg/ogg/dirdat/cb, extract pumpcbs
RMTTRAIL added.

GGSCI (whbdp01) 4> edit params pumpcbs
extract pumpcbs
passthru
dynamicresolution
userid ogg,password ogg-- 根据实际情况填写
rmthost whbdp02 mgrport 7809   -- ogg for kafka所在服务器
rmttrail /opt/ogg/ogg/dirdat/cb   #注意结尾不要多空格
table cbs.*;
```
#### 1.6.5 配置pumps, pumppp， 推送给pipeline端

```bash
GGSCI (whbdp01) 1> add extract pumppp, exttrailsource ./dirdat/cb
GGSCI (whbdp01) 1> add rmttrail /home/ggs/ogg/dirdat/cb, extract pumppp
GGSCI (whbdp01) 1> edit params pumppp


extract pumppp
passthru
dynamicresolution
userid ogg,password ogg
rmthost whbdp04 mgrport 7809
rmttrail /home/ggs/ogg/dirdat/cb
table cbs.*;
```



#### 1.6.6 生成表结构定义文件

```bash
GGSCI (whbdp01) 33>  edit param cbsdefgen


USERID ogg, PASSWORD ogg
defsfile ./dirdef/cbs.def
table cbs.*;

[ogg@whbdp01 ogg]$ ./defgen  paramfile ./dirprm/cbsdefgen.prm
#然后将生成的文件拷贝至目标端

#kafka放一份
[ogg@whbdp01 ogg]$ scp ./dirdef/cbs.def   ogg@192.168.10.92:/opt/ogg/ogg/dirdef

#pipeline放一份
[ogg@whbdp01 dirdef]$ scp cbs.def ggs@192.168.10.94:/home/ggs/ogg/dirdef

```


## 2. 安装pipeline(v 0.9.9)

### 2.1 新建pipeline用户

```bash
[root@whbdp04 ~]$ useradd pipeline
[root@whbdp04 ~]$ passwd pipeline
更改用户 pipeline 的密码 。
新的 密码：
无效的密码： 密码包含用户名在某些地方
重新输入新的 密码：
passwd：所有的身份验证令牌已经成功更新。
```
### 2.2 安装pipeline

```bash
[leona@whbdp04 ~]$ sudo rpm -Uvh pipelinedb-0.9.9u3-centos7-x86_64.rpm
准备中...                         ################################# [100%])
    软件包 pipelinedb-0.9.9-1.x86_64 已经安装
```

### 2.3 启动pipeline
#### 2.3.1 创建pipeline启动目录

```bash
mkdir data
[pipeline@whbdp04 ~]$ cd data/
[pipeline@whbdp04 data]$ pwd
/ssd/pipeline/data
```

#### 2.3.2 启动pipeline数据库

```bash
[pipeline@whbdp04 data]$ cd ..
[pipeline@whbdp04 ~]$ pwd
/ssd/pipeline
[pipeline@whbdp04 data]$ pipeline-init --encoding=UTF8  --locale=en_US.UTF-8 -D data
The files belonging to this database system will be owned by user "pipeline
".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory data ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
中略
WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

[pipeline@whbdp04 data]$ pipeline-ctl -D data -l logfile start
```
#### 2.3.2 修改pipeline配置文件
* 参考配置文件svn地址： https://www.5181.me/svn/kanas/branches/wuhai/branches/kanas/src/settings/config/pipelinedb
* 按上述文件修改  /data/下的配置文件

#### 2.3.4 启动pipeline

```bash
[pipeline@whbdp04 ~]$ ls
data  perl5  pipelinedb
[pipeline@whbdp04 ~]$ pipeline-ctl -D ./data start -l ksc.log
```

#### 2.3.5 登录pipeline

```bash
[pipeline@whbdp04 ~]$ psql -p 5433
psql (9.2.24, server 9.5.3)
WARNING: psql version 9.2, server version 9.5.
         Some psql features might not work.
Type "help" for help.

pipeline=#
```

## 3. 安装 OGG for Postgre
### 3.1 安装包装备
#### 3.1.1  版本信息
* `122022_ggs_Linux_x64_PostgreSQL_64bit.zip`

#### 3.1.2  解压缩

```bash
[ggs@whbdp04 ~]$ unzip -e 122022_ggs_Linux_x64_PostgreSQL_64bit.zip
Archive:  122022_ggs_Linux_x64_PostgreSQL_64bit.zip
  inflating: ggs_Linux_x64_PostgreSQL_64bit.tar  ^[[O
  inflating: OGG-12.2.0.2-README.txt
  inflating: OGGCORE_12.2.0.2.2.pdf
```
### 3.2 配置环境变量

```bash
[ggs@whbdp04 ~]$ cat ggs.env
export GGDIR=$HOME/ggs
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$GGDIR:$GGDIR/lib:.

#export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.121-1.b13.el6.x86_64/
export JAVA_HOME=/usr/java/jdk1.8.0_181-amd64/
export JAVA_LIBDIR=/usr/share/java
export PATH=$JAVA_HOME/bin:$PATH:$GGDIR
export LD_LIBRARY_PATH=$JAVA_HOME/jre/lib/amd64/server:$LD_LIBRARY_PATH
export ODBCINI=$GGDIR/odbc.ini
alias gg='rlwrap ggsci'
```

### 3.3 配置OGG
#### 3.3.1 创建相应目录

```bash
GGSCI (whbdp04) 1> create subdirs

Creating subdirectories under current directory /home/ggs/ogg

Parameter files                /home/ggs/ogg/dirprm: created
Report files                   /home/ggs/ogg/dirrpt: created
Checkpoint files               /home/ggs/ogg/dirchk: created
Process status files           /home/ggs/ogg/dirpcs: created
SQL script files               /home/ggs/ogg/dirsql: created
Database definitions files     /home/ggs/ogg/dirdef: created
Extract data files             /home/ggs/ogg/dirdat: created
Temporary files                /home/ggs/ogg/dirtmp: created
Credential store files         /home/ggs/ogg/dircrd: created
Masterkey wallet files         /home/ggs/ogg/dirwlt: created
Dump files                     /home/ggs/ogg/dirdmp: created
```

#### 3.3.2 编辑mgr, replicat配置

* mgr配置

```bash
[ggs@whbdp04 dirprm]$ pwd
/home/ggs/ogg/dirprm
[ggs@whbdp04 dirprm]$ cat mgr.prm
PORT 7809
DYNAMICPORTLIST 7810-7909
AUTORESTART EXTRACT *, RETRIES 5, WAITMINUTES 3
PURGEOLDEXTRACTS ./dirdat/*, USECHECKPOINTS, minkeepdays 30
```
```bash
GGSCI (whbdp04) 2> add replicat reppp, exttrail ./dirdat/cb, nodbcheckpoint
REPLICAT added


GGSCI (whbdp04) 3> edit params reppp
replicat reppp
SETENV ( PGCLIENTENCODING = "UTF8" )
SETENV (ODBCINI="/home/ggs/odbc.ini" )    #注意这里odbc.ini的路径
SETENV (NLS_LANG="AMERICAN_AMERICA.AL32UTF8")
SOURCEDEFS ./dirdef/cbs.def
targetdb pipelinedb, userid pipeline, password pipeline
DISCARDFILE ./dirrpt/reppg.dsc

HANDLECOLLISIONS
OVERRIDEDUPS

MAP cbs.account, TARGET public.account;
MAP cbs.account_balance, TARGET public.account_balance;
MAP cbs.account_class, TARGET public.account_class;
MAP cbs.account_role, TARGET public.account_role;
MAP cbs.classification, TARGET public.classification;
MAP cbs.subject_item, TARGET public.subject_item;
MAP cbs.party_role, TARGET public.party_role;
MAP cbs.branch, TARGET public.branch;
MAP cbs.entry, TARGET public.entry;
MAP cbs.sys_para, TARGET public.sys_para;
MAP cbs.channel, TARGET public.channel;
MAP cbs.tran_jrnl, TARGET public.tran_jrnl;
```

#### 3.3.3 配置odbc.ini
* odbc.ini的存放路径为ogg.env里配置的路径

```bash
[ggs@whbdp04 ~]$ cat odbc.ini
[ODBC Data Sources]
PostgreSQL on ogg
[ODBC]
LANAAppCodePage=106
InstallDir=/home/ggs/ogg
[pipelinedb]
Driver=/home/ggs/ogg/lib/GGpsql25.so
Description=Postgres driver
Database=pipeline
Hostname=whbdp04
PortNumber=5433
LogonID=pipeline
Password=pipeline
```


## 4. 安装OGG for kafka
**ATTENTION** : 需先安装confluent-kafka(见 4.5)
* 软件版本： OGG_BigData_Linux_x64_12.3.1.1.1.tar 

### 4.1 新建用户

```bash
[root@whbdp02 ~]$ useradd ogg
[root@whbdp02 ~]$ mkdir /opt/ogg
[root@whbdp02 ~]$ passwd ogg
更改用户 ogg 的密码 。
新的 密码：
无效的密码： 密码少于 8 个字符
重新输入新的 密码：
passwd：所有的身份验证令牌已经成功更新。
[root@whbdp02 ~]$ chown -R ogg:ogg /opt/ogg 
[root@whbdp02 ~]$ usermod -d /opt/ogg ogg
```
### 4.2 使用ogg用户解压缩

```bash
[ogg@whbdp02 ~]$ tar -xvf OGG_BigData_Linux_x64_12.3.2.1.1.tar  -C ogg
```

### 4.3 编辑环境变量

```bash
[ogg@whbdp02 ~]$ cat ogg.env
export GGDIR=$HOME/ogg
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$GGDIR:$GGDIR/lib:.

#export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.121-1.b13.el6.x86_64/
export JAVA_HOME=/usr/java/jdk1.8.0_181-amd64/
export JAVA_LIBDIR=/usr/share/java
export PATH=$JAVA_HOME/bin:$PATH:$GGDIR
export LD_LIBRARY_PATH=$JAVA_HOME/jre/lib/amd64/server:$LD_LIBRARY_PATH
export ODBCINI=$GGDIR/odbc.ini
alias gg='rlwrap ./ggsci'
```
### 4.4 编辑ogg配置文件

#### 4.4.1 配置mgr
```bash
[ogg@whbdp02 ~]$ source ogg.env
[ogg@whbdp02 ~]$ cd ogg/
[ogg@whbdp02 ogg]$ ./ggsci
GGSCI (whbdp02) 1> create subdirs
GGSCI (whbdp02) 2> edit params mgr

PORT 7809
DYNAMICPORTLIST 7810-7909
AUTORESTART EXTRACT *,RETRIES 5,WAITMINUTES 3
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints, minkeepdays 30
```

#### 4.4.2 配置checkpoints

```bash
GGSCI (whbdp02) 3> edit params ./GLOBALS

CHECKPOINTTABLE cbs.checkpoint
```

#### 4.4.3 配置replicate 进程

* recbs ,给tunnel用

```bash
GGSCI (whbdp02) 4>  edit params recbs


REPLICAT recbs
TARGETDB LIBFILE libggjava.so SET property=dirprm/kafkaconnect_cbs.props
REPORTCOUNT EVERY 1 MINUTES, RATE
GROUPTRANSOPS 1000
MAXTRANSOPS 1000
SOURCEDEFS ./dirdef/cbs.def
MAP cbs.*, TARGET cbs.*; #乌海生产环境，是将map的表一一列举的

```

* ppcbsre, 给ksctunnel用

```bash
GGSCI (whbdp02) 4> edit param ppcbsre
REPLICAT ppcbsre
TARGETDB LIBFILE libggjava.so SET property=dirprm/kafkaconnect_ppcbsreplicat.props  #注意此处配置文件
REPORTCOUNT EVERY 1 MINUTES, RATE
GROUPTRANSOPS 1000
MAXTRANSOPS 1000
SOURCEDEFS ./dirdef/cbs.def
MAP cbs.account, TARGET kstcbs.account;
MAP cbs.account_balance, TARGET kstcbs.account_balance;
MAP cbs.account_class, TARGET kstcbs.account_class;
MAP cbs.account_role, TARGET kstcbs.account_role;
MAP cbs.classification, TARGET kstcbs.classification;
MAP cbs.subject_item, TARGET kstcbs.subject_item;
MAP cbs.party_role, TARGET kstcbs.party_role;
MAP cbs.branch, TARGET kstcbs.branch;
MAP cbs.entry, TARGET kstcbs.entry;
MAP cbs.sys_para, TARGET kstcbs.sys_para;
MAP cbs.ldm_deposit_acct_register, TARGET kstcbs.ldm_deposit_acct_register;

```

#### 4.4.4 配置kafka-connect (使用confluent-kafka)

```bash
[ogg@whbdp02 dirprm]$ cat kafkaconnect_cbs.props
gg.handlerlist=kafkaconnect
#The handler properties
gg.handler.kafkaconnect.type=kafkaconnect
gg.handler.kafkaconnect.kafkaProducerConfigFile=json.properties
gg.handler.kafkaconnect.mode=tx
gg.handler.kafkaconnect.topicMappingTemplate=ogg_oracle_${schemaName}
gg.handler.kafkaconnect.keyMappingTemplate=${primaryKeys}
gg.handler.kafkaconnect.includePrimaryKeys=true
gg.handler.kafkaconnect.includeTokens=true

#The formatter properties
gg.handler.kafkaconnect.messageFormatting=op
gg.handler.kafkaconnect.insertOpKey=I
gg.handler.kafkaconnect.updateOpKey=U
gg.handler.kafkaconnect.deleteOpKey=D
gg.handler.kafkaconnect.truncateOpKey=T                             [9/349]
                        gg.handler.kafkaconnect.mapLargeNumbersAsStrings=true
gg.handler.kafkaconnect.treatAllColumnsAsStrings=false
 
gg.handler.kafkaconnect.iso8601Format=false
gg.handler.kafkaconnect.pkUpdateHandling=abend

goldengate.userexit.timestamp=utc
goldengate.userexit.writers=javawriter

javawriter.stats.display=TRUE
javawriter.stats.full=TRUE

gg.log=log4j
gg.log.level=INFO
gg.report.time=30sec
##Set the classpath here
gg.classpath=/usr/share/java/kafka-serde-tools/*:/usr/share/java/kafka/*:/u
sr/share/java/confluent-common/*
#gg.classpath=dirprm/:/opt/ogg/ggjava/resources/lib:/opt/cloudera/parcels/K
AFKA/lib/kafka/libs/*
javawriter.bootoptions=-Xmx4196m -Xms2048m -Djava.class.path=.:ggjava/ggjav
a.jar:./dirprm
```

#### 4.4.5 custom_kafka_producer.properties

```bash
[ogg@whbdp02 dirprm]$ cat custom_kafka_producer.properties
bootstrap.servers=whbdp02:9092   #按实际情况作出调整
acks=1
reconnect.backoff.ms=1000

value.serializer=org.apache.kafka.common.serialization.ByteArraySerializer
key.serializer=org.apache.kafka.common.serialization.ByteArraySerializer
# 100KB per partition
batch.size=16384
linger.ms=0
```

#### 4.4.6 添加replicat 进程

```bash
GGSCI > add replicat recbs, exttrail ./dirdat/cb, checkpointtable cbs.checkpoint 
REPLICAT added. 
GGSCI > add replicat ppcbsre, exttrail ./dirdat/cb, checkpointtable cbs.checkpoint 
REPLICAT added. 
```

### 4.5 安装confluent-kafka
#### 4.5.1 下载rpm包，搭建本地yum源

* 参考 192.168.10.91:/home/dw/whbdp/software/confluent-kafka
* fixme 安装包下载地址？

```bash
[root@whbdp01 confluent-kafka]# ls
archive.key     confluent.repo.origin       packages
confluent.repo  kafka-manager-1.3.3.15.zip  repodata
[root@whbdp01 confluent-kafka]#
```

#### 4.5.2 在yum源搭建服务器，修改nginx配置

```bash
server {
        listen       8000;    #端口任意
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
        server_name  localhost;

        root   /home/dw/whbdp/software;  #路径按实际情况填写

        location / {
            root   /home/dw/whbdp/software; #路径按实际情况填写
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }
```

#### 4.5.3 配置其余服务器yum repository

```bash
[root@whbdp02 yum.repos.d]# ls
cloudera-cdh6.repo      confluent.repo  epel-testing.repo
cloudera-manager6.repo  epel.repo       tmp
[root@whbdp02 yum.repos.d]# pwd
/etc/yum.repos.d
```
* epel.repo

```bash
[root@whbdp02 yum.repos.d]# cat epel.repo
[epel]
name=Extra Packages for Enterprise Linux 7
baseurl=http://yum-server:8000/epel7/
failovermethod=priority
enabled=1
gpgcheck=0
gpgkey=http://yum-server:8000/epel7/RPM-GPG-KEY-EPEL-7
```
* confluent-kafka.repo

```bash
[root@whbdp02 yum.repos.d]# cat confluent.repo
[Confluent.dist]
name=Confluent repository (dist)
baseurl=http://yum-server:8000/confluent-kafka
gpgcheck=0
gpgkey=http://yum-server:8000/confluent-kafka/archive.key
enabled=1


[Confluent]
name=Confluent repository
baseurl=http://yum-server:8000/confluent-kafka
gpgcheck=0
gpgkey=http://yum-server:8000/confluent-kafka/archive.key
enabled=1
```
* 安装confluent-kafka

```bash
yum clean all && yum install confluent-platform-oss-2.11
```

## 5. Pipeline端实时开始
### 5.1 同步kudu数据库到pipeline
#### 5.1.1 脚本路径 

```bash
 https://www.5181.me/svn/kanas/branches/wuhai/branches/kanas/src/applications/ksc
```

#### 5.1.2  kpinit.py 参数调整

```bash
    parser.add_argument('--db-source', type=str,  default='oracle+cx_oracle://cbs:cbs@192.168.10.92:1521/bdpapp', metavar="cx_oracle+oracle://cbs:cbs@draco01:1521/cbs", required=False, help="source db lake")   # 同步数据源
    parser.add_argument('--db-sink', type=str,  default='postgresql+psycopg2://pipeline:pipeline@localhost:5433/', metavar="postgresql+psycopg2://pipeline:pipeline@localhost:5433/", required=False, help="default db sinker")      parser.add_argument('--database', type=str,  metavar="cbs", default="cbs", required=False, help="default database name") # 同步数据目标
    parser.add_argument('--schema', type=str,  metavar="public", default="public", required=False, help="sink database schema")  #目标schema
    parser.add_argument('--table', type=str,  metavar="table name", required=False, help="table name")  #单表同步
    parser.add_argument('--bulk-len', type=int, default=3000, metavar="5000", required=False, help="size of bulk") #默认即可
    parser.add_argument('--schema-only', type=str,  metavar="N", default="N", required=False, help="schema only?")  #设置为N
    parser.add_argument('--partition', type=str,  metavar="Y", default="Y", required=False, help="write partition?")
    parser.add_argument('--overwrite', type=bool,  metavar="True", default=True, required=False, help="overwrite?")  #设置为TRUE
    parser.add_argument('--cache-key', type=str,  metavar="id", default="id", required=False, help="primary key")  
    parser.add_argument('--timestamp', type=bool,  metavar="True", default=True, required=False, help="write timestamp")  
    parser.add_argument('--fiscal-date', type=str,  metavar="2018-05-01", default="2018-05-01", required=False, help="write timestamp")  
    parser.add_argument('--sql', type=str,  metavar="SQL", required=False, help="sql to run")

```

#### 5.1.3 执行数据同步

```bash
python kpinit1.py 
python kpinit2.py 
python kpinit3.py 
```
#### 5.1.4 生成ksc表

```bash
(kanas/src)[leona@whbdp04 ksc]$ python pipelinedb.py entry
```

### 5.2 建表及索引
#### 5.2.1 脚本路径

```bash
/home/leona/workspace/kanas/src/applications/ksc/domain
```
#### 5.2.2 建表  sh init.sh

```bash
(kanas/src)[leona@whbdp04 domain]$ sh init.sh
CREATE SCHEMA
CREATE SCHEMA
CREATE EXTENSION
 add_broker
------------
 success
(1 row)

2019-08-01 09:05:27,120 INFO hiveserver2.py 265 19328 Closing active operation
2019-08-01 09:05:27,234 INFO hiveserver2.py 265 19328 Closing active operation
```
#### 5.2.3  建索引 sh index.sh

```bash
(kanas/src)[leona@whbdp04 domain]$ sh index.sh
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
```

#### 5.2.4 生成规则 python setup_policy.py

```bash
(kanas/src)[leona@whbdp04 domain]$ python setup_policy.py
```


### 5.3 生成流等
#### 5.3.1 脚本路径

```bash
/home/leona/workspace/kanas/src/applications/ksc/stream
```
#### 5.3.2 python sc_date_gl.py   生成期初总账

```bash
(kanas/src)[leona@whbdp04 stream]$ python sc_date_gl.py            
2019-08-01 09:31:49,660 INFO kdsink.py 77 23516 sink scheme => postgresql+p
sycopg2
2019-08-01 09:31:49,723 INFO sqlalchemy.engine.base.Engine select version()
2019-08-01 09:31:49,723 INFO log.py 109 23516 select version()
2019-08-01 09:31:49,724 INFO sqlalchemy.engine.base.Engine {}
2019-08-01 09:31:49,724 INFO log.py 109 23516 {}
2019-08-01 09:31:49,725 INFO sqlalchemy.engine.base.Engine select current_schema()
2019-08-01 09:31:49,725 INFO log.py 109 23516 select current_schema()
2019-08-01 09:31:49,725 INFO sqlalchemy.engine.base.Engine {}
2019-08-01 09:31:49,725 INFO log.py 109 23516 {}
2019-08-01 09:31:49,726 INFO sqlalchemy.engine.base.Engine SELECT CAST('test plain returns' AS VARCHAR(60)) AS anon_1
2019-08-01 09:31:49,726 INFO log.py 109 23516 SELECT CAST('test plain returns' AS VARCHAR(60)) AS anon_1
2019-08-01 09:31:49,726 INFO sqlalchemy.engine.base.Engine {}
2019-08-01 09:31:49,726 INFO log.py 109 23516 {}
.
.
.
.
.
.
2019-08-01 09:34:09,034 INFO hiveserver2.py 265 23516 Closing active operation
2019-08-01 09:34:09,044 INFO sc_date_gl.py 304 23516 GL task elpased 139.30798912 seconds
```

#### 5.3.4 python sc_gl_entry.py  生成 gl_entry 流

```shell
python sc_gl_entry.py
python sc_gl_report.py
python sc_dim_account.py
```
