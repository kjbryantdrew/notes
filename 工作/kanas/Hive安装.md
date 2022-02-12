版本情况



```
Centos: 7.5 1804
CDH: 6.1.0
```



HIVE2 角色部署方案



```bash
whbdp01: HIVE METASTORE SERVER, MySQL
Whbdp02: HIVE METASTORE SERVER, MySQL
Whbdp03: HIVE METASTORE SERVER
Whbdp04: HIVE METASTORE SERVER
```



whbdp01/whbdp02 安装 MySQL



```bash
# 卸载
rpm -qa|grep mariadb
rpm -e mariadb-libs-5.5.56-2.el7.x86_64 --nodeps
rpm -e mariadb-server-5.5.56-2.el7.x86_64 --nodeps


# 安装
rpm -ivh mysql-community-common-5.7.23-1.el7.x86_64.rpm --force --nodeps
rpm -ivh mysql-community-libs-5.7.23-1.el7.x86_64.rpm  --force --nodeps
rpm -ivh mysql-community-client-5.7.23-1.el7.x86_64.rpm  --force --nodeps
rpm -ivh mysql-community-server-5.7.23-1.el7.x86_64.rpm  --force --nodeps
```



初始化数据库



为了保证数据库目录为与文件的所有者为 mysql 登陆用户，如果你是以 root 身份运行 mysql 服务，需要执行下面的命令初始化



```
mysqld --initialize --user=mysql
# 如果是以 mysql 身份运行，则可以去掉 --user 选项。
# 另外 --initialize 选项默认以“安全”模式来初始化，则会为 root 用户生成一个密码并将该密码标记为过期，登陆后你需要设置一个新的密码，
# 而使用 --initialize-insecure 命令则不使用安全模式，则不会为 root 用户生成一个密码。
# 这里使用的 --initialize 初始化，会生成一个 root 账户密码，密码在log文件里，红色区域的就是自动生成的密码
# mysql配置文件在/etc/my.cnf,里面指明了数据目录。日志位置/var/log/mysqld.log
```



MySQL配置



```
[mysqld]
plugin-load=validate_password.so
validate_password=FORCE_PLUS_PERMANENT
validate_password_policy=2


mysql>>INSTALL PLUGIN validate_password SONAME 'validate_password.so';
```



启动



```
systemctl start mysqld
systemctl enable mysqld
systemctl list-unit-files | grep mysqld
```



修改密码



```
mysql -uroot -p
Enter password:
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'We@Passw0rd#123';
```



MySQL 创建 Hive 用户



```
mysql> CREATE DATABASE hive; 
mysql> USE hive; 
mysql> CREATE USER 'hive'@'localhost' IDENTIFIED BY 'hive';
mysql> GRANT ALL ON hive.* TO 'hive'@'localhost' IDENTIFIED BY 'hive'; 
mysql> GRANT ALL ON hive.* TO 'hive'@'%' IDENTIFIED BY 'hive'; 
mysql> GRANT ALL ON hive.* TO 'hive'@'%' IDENTIFIED BY 'hive';
mysql> FLUSH PRIVILEGES; 
```



所有服务器安装 Hive



```
yum install -y hive*
```



whbdp01 配置



```
cat /etc/hive/conf/hive-env.sh
HADOOP_HOME=/usr/lib/hadoop
export HIVE_CONF_DIR=/etc/hive/conf


vi /etc/bashrc
export HIVE_HOME=/usr/lib/hive
export PATH=$HIVE_HOME/bin:$PATH
所有服务器拷贝 mysql-connector-java-5.1.22-bin.jar 放入 `$HIVE_HOME/lib` 下
```



初始化 hive metastore 数据库



```
pwd
/usr/lib/hive/bin
./schematool -dbType mysql -initSchema
```



所有服务器启动 metastore



```
pwd
/usr/lib/hive/bin


systemctl start hive-metastore.servic
```