### 登录服务器
```bash
ssh root@160.160.9.39
```

### 切换系统用户并配置informix server
```bash
# 切换至informix用户
su - informix
# 进入etc目录
cd etc
# 创建配置文件
cp sqlhosts.demo sqlhosts.server # 此处【server】后缀名可以自定义
# 修改配置文件
vim sqlhosts.server
# 在最后一行添加如下配置
node44 onsoctcp 160.160.9.44 sqlexec
```
![image-20210807002928804](../img/image-20210807002928804.png)


关于上诉配置说明：
- node44：informix的server名，根据具体的informix数据库进行修改，且同一配置为文件中不允许出现相同的server名
- onsoctcp：连接方式，一般无须修改
- 160.160.9.44：informix服务器ip
- sqlexec：informix端口别名，具体的对应关系可在`/etc/services`中查看

### 配置informix环境变量
> 为使前端能够正常访问，需在kanas用户下配置
```bash
# 切换用户
sudo su - kanas
# 进入sn目录
cd sn
# 编辑配置文件
vim informix.env
# 配置如下
export INFORMIXSERVER=snhx
export ONCONFIG=onconfig.snhx
export CLIENT_LOCALE=en_US.819
export DB_LOCALE=en_US.819
export SERVER_LOCALE=en_US.819
export INFORMIXSQLHOSTS=$INFORMIXDIR/etc/sqlhosts.snhx
export INFORMIXTERM=$INFORMIXDIR/etc/termcap
LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$INFORMIXDIR/lib:$INFORMIXDIR/lib/cli:$INFORMIXDIR/lib/esql:
export LD_LIBRARY_PATH
export TERMCAP=$INFORMIXDIR/etc/termcap
export INFORMIXTERM=TERMCAP
# 修改完成以后将配置加载到环境变量
source informix.env
```
主要参数说明：
- INFORMIXSERVER：server名，与sqlhosts.server配置中的server保持一致
- INFORMIXSQLHOSTS：sqlhosts配置文件路径，sqlhosts.server文件的路径
- CLIENT_LOCALE、DB_LOCALE、SERVER_LOCALE：根据informix数据库修改

### Python连接informix
```python
In [1]: from klake.lpool import DBLake

In [2]: url = 'informix+informixdb://fps:fps@snhx/snfps'

In [3]: conn = DBLake(url)
2019-11-05 16:55:05,102 INFO sqlalchemy.engine.base.Engine SELECT CURRENT_USER FROM sysmaster:"informix".sysdual
2019-11-05 16:55:05,102 INFO sqlalchemy.engine.base.Engine ()
2019-11-05 16:55:05,121 INFO sqlalchemy.engine.base.Engine SELECT CAST('test plain returns' AS VARCHAR(60)) AS anon_1 from systables where tabname = 'systables' 
2019-11-05 16:55:05,121 INFO sqlalchemy.engine.base.Engine ()
2019-11-05 16:55:05,154 INFO sqlalchemy.engine.base.Engine SELECT CAST('test unicode returns' AS VARCHAR(60)) AS anon_1 from systables where tabname = 'systables' 
2019-11-05 16:55:05,154 INFO sqlalchemy.engine.base.Engine ()

In [4]: conn.close()
```
