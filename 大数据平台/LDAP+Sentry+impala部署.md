[TOC]

## Introduction

### 系统基础
* OS: CentOS 7.3
* impala: 3.1.0+cdh6.1.0
* sentry: 2.1.0+cdh6.1.0
* LADP: 2.4.40

### 目标

1. 使用sentry提供细粒度的权限控制
2. 使用LDAP提供用户、密码访问

### 限制

1. 本次未启用TSL/SSl加密
2. 仅Impala添加sentry权限控制，hive未添加

## Sentry 安装

- 组件安装
客户端不需要安装store
```bash
yum install sentry sentry-hdfs-plugin sentry-store
```

- 创建sentry database
以postgres数据库作为sentry store，安装以后创建用户和database并赋权
```bash
postgres=# CREATE USER sentry WITH PASSWORD 'sentry';
postgres=# CREATE DATABASE sentry OWNER sentry;
postgres=# GRANT ALL PRIVILEGES ON DATABASE sentry TO sentry;
```

- postgre驱动包
```shell
yum install postgresql-jdbc
ln -s /usr/share/java/postgresql-jdbc.jar /usr/lib/sentry/lib
```

- 初始化sentry database
```bash
sentry --command schema-tool --conffile /etc/sentry/conf/sentry-site.xml --dbType postgres --initSchema
# 输出如下信息初始化成功
Initialization script completed
Sentry schemaTool completed
```

- sentry配置文件
```bash
vim /etc/sentry/conf/sentry-site.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property>
        <name>sentry.service.server.rpc-address</name>
        <value>big-data04</value>
    </property>

    <property>
        <name>sentry.service.server.rpc-port</name>
        <value>8038</value>
    </property>
 
    <property>
        <name>sentry.service.admin.group</name>
        <value>hive,impala,hue,hdfs</value>
    </property>
 
    <property>
        <name>sentry.service.allow.connect</name>
        <value>hive,impala,hue,hdfs</value>
    </property>
 
    <property>
        <name>sentry.store.group.mapping</name>
        <value>org.apache.sentry.provider.common.HadoopGroupMappingService</value>
    </property>
 
    <property>
        <name>sentry.service.reporting</name>
        <value>JMX</value>
    </property>
 
    <property>
        <name>sentry.service.web.enable</name>
        <value>true</value>
    </property>
 
    <property>
        <name>sentry.service.web.port</name>
        <value>51000</value>
    </property>
 
    <property>
        <name>sentry.service.web.authentication.type</name>
        <value>NONE</value>
    </property>
 
    <property>
        <name>sentry.verify.schema.version</name>
        <value>true</value>
    </property>
 
    <property>
        <name>sentry.service.security.mode</name>
        <value>none</value>
    </property>
 
    <property>
        <name>sentry.store.jdbc.url</name>
        <value>jdbc:postgresql://big-data04:5432/sentry</value>
    </property>
 
    <property>
        <name>sentry.store.jdbc.driver</name>
        <value>org.postgresql.Driver</value>
    </property>
 
    <property>
        <name>sentry.store.jdbc.user</name>
        <value>sentry</value>
    </property>
 
    <property>
        <name>sentry.store.jdbc.password</name>
        <value>sentry</value>
    </property>
 
</configuration>
```

- 启动服务
```shell
sentry --command service --conffile <sentry-site.xml>
/etc/init.d/sentry-store start
```

## impala集成sentry

`sentry`提供的权限服务以impala进程（impalad）为准，同一集群的不同impalad可以配置不同的权限

- 修改impala配置

* `impala/conf`

```xml
vim /etc/impala/conf/sentry-site.xml

<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property>
       <name>sentry.service.client.server.rpc-port</name>
       <value>8038</value>
    </property>
    <property>
       <name>sentry.service.client.server.rpc-address</name>
       <value>big-data04</value>
    </property>
    <property>
       <name>sentry.service.client.server.rpc-connection-timeout</name>
       <value>200000</value>
    </property>
    <property>
       <name>sentry.service.security.mode</name>
       <value>none</value>
    </property>
</configuration>
```

> 注意区分此处的sentry配置与前文的安装sentry时的sentry配置

* `/ect/default/impala`

  在`IMPALA_SERVER_ARGS`中添加

  ```shell
  -sentry_config=/etc/impala/conf/sentry-site.xml
  -server_name=server1
  ```

 server_name表示当前impalad进程的别名，可用于sever级别的赋权，不同impalad名字应当不同。

  在`IMPALA_CATALOG_ARGS`中添加

  ```shell
  -sentry_config=/etc/impala/conf/sentry-site.xml
  ```

- 服务

  * 重启impala catalog
  * 重启配置sentry控制的impalad节点

- impala赋权

  * sentry使用SQL语句赋权，hive与impala同时兼容
  * 赋权逻辑为：权限-->角色-->组
  * 创建角色role，并赋权（Operation：all、select、insert，Object：server、database、table、columns）给组group
  * 根据连接impala的用户所属组检查对应权限，用户和组均为操作系统用户和组

> Example:
```sql
[big-data04:21000] default> create role test_role;
Query: create role test_role

[big-data04:21000] default> grant select on database preset to role test_role;
Query: grant select on database preset to role test_role
Query submitted at: 2019-09-01 15:23:56 (Coordinator: http://big-data04:25000)
Query progress can be monitored at: http://big-data04:25000/query_plan?query_id=d744467fe29c2215:67e69b000000000
+---------------------------------+
| summary                         |
+---------------------------------+
| Privilege(s) have been granted. |
+---------------------------------+
Fetched 1 row(s) in 0.05s
[big-data04:21000] default> grant role test_role to group test_group;
Query: grant role test_role to group test_group
Query submitted at: 2019-09-01 15:24:14 (Coordinator: http://big-data04:25000)
Query progress can be monitored at: http://big-data04:25000/query_plan?query_id=7244af9308a4a9a6:20203b500000000
+------------------------+
| summary                |
+------------------------+
| Role has been granted. |
+------------------------+
Fetched 1 row(s) in 0.09s
[big-data04:21000] default>
```

当impalad启用sentry，impala-shell连接该节点时，需要提供用户名，且此节点不能使用JDBC连接

```shell
[root@big-data04 ~]# impala-shell -u impala
```

## LDAP

- 安装组件

```shell
yum install  openldap-* migrationtools
```

- 配置管理员密码

```shell
[root@big-data04 ldap]# slappasswd
New password:
Re-enter new password:
{SSHA}Ku8bulyVy+RDjFJ2amsUvCM0Zi68xVN0
```

- 修改olcDatabase
```shell
[root@big-data04 ldap]# vim /etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif


dn: olcDatabase={2}hdb
objectClass: olcDatabaseConfig
objectClass: olcHdbConfig
olcDatabase: {2}hdb
olcDbDirectory: /var/lib/ldap
olcSuffix: dc=bigdata,dc=com
olcRootDN: cn=Manager,dc=bigdata,dc=com
olcRootPW: {SSHA}Ku8bulyVy+RDjFJ2amsUvCM0Zi68xVN0
olcDbIndex: objectClass eq,pres
olcDbIndex: ou,cn,mail,surname,givenname eq,pres,sub
structuralObjectClass: olcHdbConfig
entryUUID: 7d95b9de-60a1-1039-87f9-13aad8da3ce6
creatorsName: cn=config
createTimestamp: 20190901011346Z
entryCSN: 20190901011346.047292Z#000000#000#000000
modifiersName: cn=config
modifyTimestamp: 20190901011346Z
```

添加olcRootPW，值为上一步的密码哈希值，可以修改olcSuffix和olcRootDN为自己设置的域。

- 修改olcDatabase\={1}monitor
修改olcAccess为自己设置的域

```shell
[root@big-data04 ldap]# vim /etc/openldap/slapd.d/cn\=config/olcDatabase\=\{1\}monitor.ldif

dn: olcDatabase={1}monitor
objectClass: olcDatabaseConfig
olcDatabase: {1}monitor
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=extern
 al,cn=auth" read by dn.base="cn=Manager,dc=bigdata,dc=com" read by * none
structuralObjectClass: olcDatabaseConfig
entryUUID: 7d95ab60-60a1-1039-87f8-13aad8da3ce6
creatorsName: cn=config
createTimestamp: 20190901011346Z
entryCSN: 20190901011346.046923Z#000000#000#000000
modifiersName: cn=config
modifyTimestamp: 20190901011346Z
```

- 启动服务
```shell
[root@big-data04 ldap]# slaptest -u
5d6b2df8 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={1}monitor.ldif"
5d6b2df8 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif"
config file testing succeeded

[root@big-data04 ldap]# systemctl start slapd

[root@big-data04 ldap]# netstat -anp | grep 389
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      48427/slapd
tcp6       0      0 :::389                  :::*                    LISTEN      48427/slapd
```

- 配置数据库
```shell
[root@big-data04 ldap]# cp /usr/share/openldap-servers/DB_CONFIG.example  /var/lib/ldap/DB_CONFIG
[root@big-data04 ldap]# chown ldap:ldap /var/lib/ldap/DB_CONFIG
[root@big-data04 ldap]# chmod 700 /var/lib/ldap/DB_CONFIG
```

- 导入基础配置
```shell
[root@big-data04 ldap]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=cosine,cn=schema,cn=config"

[root@big-data04 ldap]#
[root@big-data04 ldap]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=nis,cn=schema,cn=config"

[root@big-data04 ldap]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=inetorgperson,cn=schema,cn=config"
```

- 生成系统用户配置文件

* 修改导入工具domain、base、schema

```shell
[root@big-data04 ldap]# vim /usr/share/migrationtools/migrate_common.ph

# Default DNS domain
$DEFAULT_MAIL_DOMAIN = "bigdata.com";

# Default base
$DEFAULT_BASE = "dc=bigdata,dc=com";

# Turn this on for inetLocalMailReceipient
# sendmail support; add the following to
# sendmail.mc (thanks to Petr@Kristof.CZ):
##### CUT HERE #####
#define(`confLDAP_DEFAULT_SPEC',`-h "ldap.padl.com"')dnl
#LDAPROUTE_DOMAIN_FILE(`/etc/mail/ldapdomains')dnl
#FEATURE(ldap_routing)dnl
##### CUT HERE #####
# where /etc/mail/ldapdomains contains names of ldap_routed
# domains (similiar to MASQUERADE_DOMAIN_FILE).
# $DEFAULT_MAIL_HOST = "mail.padl.com";

# turn this on to support more general object clases
# such as person.
$EXTENDED_SCHEMA = 1;
```

* 添加用户和组

从系统/etc/groups 和/etc/passwd将需要导入的用户和组文件生成ldif文件，导入ldap中，在导入前需给用户设置密码。建议将hive、impala、hadoop都添加进去，本次以测试用户guestuser为例

```shell
[root@big-data04 ldap]# groupadd  guestgroup
[root@big-data04 ldap]# man useradd
[root@big-data04 ldap]# useradd guestuser -g guestgroup -M
[root@big-data04 ldap]# passwd  guestuser
Changing password for user guestuser.
New password:
BAD PASSWORD: The password fails the dictionary check - it is too simplistic/systematic
Retype new password:
passwd: all authentication tokens updated successfully.
```

* 导出测试用户数据

```shell
[root@big-data04 ldap]# cat /etc/passwd | grep guest > /tmp/passwd
[root@big-data04 ldap]# cat /etc/group | grep guest > /tmp/group

[root@big-data04 ldap]# /usr/share/migrationtools/migrate_passwd.pl  /tmp/passwd  /tmp/passwd.ldif
[root@big-data04 ldap]# /usr/share/migrationtools/migrate_group.pl  /tmp/group  /tmp/group.ldif
```

* 导入配置

```shell
[root@big-data04 ldap]# ldapadd -x -D "cn=Manager,dc=bigdata,dc=com" -W -f /tmp/base.ldif
Enter LDAP Password:

[root@big-data04 ldap]# ldapadd -x -D "cn=Manager,dc=bigdata,dc=com" -W -f /tmp/passwd.ldif
Enter LDAP Password:
adding new entry "uid=guestuser,ou=People,dc=bigdata,dc=com"

[root@big-data04 ldap]# ldapadd -x -D "cn=Manager,dc=bigdata,dc=com" -W -f /tmp/group.ldif
Enter LDAP Password:
adding new entry "cn=guestgroup,ou=Group,dc=bigdata,dc=com"
```

## impala启用LDAP

- 配置impala

添加ldap相关参数

```shell
vim /etc/default/impala

IMPALA_SERVER_ARGS=" \
    -log_dir=${IMPALA_LOG_DIR} \
    -kudu_master_hosts=160.160.9.39:7051 \
    -catalog_service_host=${IMPALA_CATALOG_SERVICE_HOST} \
    -state_store_port=${IMPALA_STATE_STORE_PORT} \
    -state_store_host=${IMPALA_STATE_STORE_HOST} \
    -fe_service_threads=500 \
    -be_service_threads=500 \
    -sentry_config=/etc/impala/conf/sentry-site.xml \
    -server_name=server1 \
    -enable_ldap_auth=true \
    -ldap_tls=false \
    -ldap_passwords_in_clear_ok=true \
    -ldap_uri=ldap://big-data04 \
    -ldap_baseDN=ou=People,dc=bigdata,dc=com \
    -be_port=${IMPALA_BACKEND_PORT} \
    -enable_orc_scanner=true"
```

- 登录impala-shell

使用添加到LDAP的用户登录impala-shell，提示需要密码

* -u: 用户
* -l: 使用LDAP服务
* --auth_creds_ok_in_clear： 不启用TSL加密

```shell
[root@big-data04 ~]# impala-shell -u impala -l --auth_creds_ok_in_clear
Starting Impala Shell using LDAP-based authentication
LDAP password for impala:
```

## hive启用sentry

* 如无必要，hive可不需要配置sentry

## hive启用LDAP

在hive-server节点，编辑`/etc/hive/conf/hive-site.xml`配置文件

```xml
<!-- LDAP for hive server2 -->
<property>
  <name>hive.server2.authentication</name>
  <value>LDAP</value>
</property>
<property>
  <name>hive.server2.authentication.ldap.url</name>
  <value>ldap://big-data04</value>
</property>
<property>
  <name>hive.server2.authentication.ldap.baseDN</name>
  <value>ou=People,dc=bigdata,dc=com</value>
```

重启hive-server2

```shell
systemctrl restart hive-server2
```

## 连接启用权限的impala、hive

### JDBC for Impala

* 使用impala驱动
* authmech为认证方法，3代表LDAP
* UID： LADP用户
* PWD：用户密码

> JDBC连接串：jdbc:impala://big-data04:21050/;AuthMech=3;UID=guestuser;PWD=12345678;

### beeline for hive

* jdbc端口为hs2服务端口
* user、password为以添加至ldap的用户

```shell
[root@big-data03 conf]# beeline
Beeline version 2.1.1-cdh6.1.0 by Apache Hive
beeline> !connect jdbc:hive2://big-data03:10001
Connecting to jdbc:hive2://big-data03:10001
Enter username for jdbc:hive2://big-data03:10001: hive
Enter password for jdbc:hive2://big-data03:10001: ****
Connected to: Apache Hive (version 2.1.1-cdh6.1.0)
Driver: Hive JDBC (version 2.1.1-cdh6.1.0)
Transaction isolation: TRANSACTION_REPEATABLE_READ
0: jdbc:hive2://big-data03:10001>
```

### python

* python连接impala/hive使用HS2的接口，hiveServer2必须启用LDAP

- 组件依赖

```shell
[root@big-data03 log]# yum list installed | grep sasl
cyrus-sasl.x86_64                     2.1.26-23.el7                  @/cyrus-sasl-2.1.26-23.el7.x86_64
cyrus-sasl-devel.x86_64               2.1.26-23.el7                  @/cyrus-sasl-devel-2.1.26-23.el7.x86_64
cyrus-sasl-gssapi.x86_64              2.1.26-23.el7                  @/cyrus-sasl-gssapi-2.1.26-23.el7.x86_64
cyrus-sasl-ldap.x86_64                2.1.26-23.el7                  @/cyrus-sasl-ldap-2.1.26-23.el7.x86_64
cyrus-sasl-lib.x86_64                 2.1.26-23.el7                  @/cyrus-sasl-lib-2.1.26-23.el7.x86_64
cyrus-sasl-plain.x86_64               2.1.26-23.el7                  @/cyrus-sasl-plain-2.1.26-23.el7.x86_64
erlang-sasl.x86_64                    R16B-03.18.el7                 @local-epel
python-saslwrapper.x86_64             0.16-5.el7                     @local-epel
saslwrapper.x86_64                    0.16-5.el7                     @/saslwrapper-0.16-5.el7.x86_64
saslwrapper-devel.x86_64              0.16-5.el7                     @local-epe
```

- pip依赖

```shell
pip install impyla
pip install thrift
pip install sasl
pip install thrift_sasl
```

- 连接hive

```shell
In [1]: from impala.dbapi import  connect
In [3]: conn = connect(host='big-data03', port=10001, user='hive', password='hive', auth_mechanism='PLAIN')
In [24]: cur = conn.cursor()

In [25]: cur.execute('show databases')

In [26]: cur.fetchall()
```

- 连接impala

* TODO impyla不能连接21050，待测试
