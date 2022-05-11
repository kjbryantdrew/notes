### 检查数据库示例是否启动

```sql
SQL> select status from v$instance;

STATUS
------------
OPEN
```

### 监听异常

```sql
# 主要体现在启动监听的时候提示The listener supports no services
# 没有监听到服务会导致远程连接的时候跑错ORA-12514:TNS:listener does not currently know of service requested in connect descriptor
# 在listener.ora增加
SID_LIST_LISTENER =
(SID_LIST = (SID_DESC = (GLOBAL_DBNAME = orcl) (SID_NAME = orcl)))

# 办法二 强制注册服务
SQL> show parameter service_names
强制注册服务：
SQL> alter system register;
# 但方法二在本地实测没用
```

### 创建表空间

```sql
# 查看表空间
SQL> select TABLESPACE_NAME,FILE_NAME,BYTES/1024/1024 "MB" from dba_data_files;
# 创建表空间
SQL> create tablespace sn datafile '/home/oracle/app/oracle/oradata/orcl/sn.dbf' size 500M autoextend on next 50M maxsize  unlimited;
# 增加表空间大小
SQL> alter database datafile '/home/oracle/app/oracle/oradata/orcl/sn.dbf' resize 5000m;
```

### 修改用户表空间

```sql
# 查看当前用户的表空间
SQL> select USERNAME, DEFAULT_TABLESPACE from dba_users where username='CBS'

# 修改表空间
SQL> alter user CBS default tablespace sn;

User altered.
```

### 新增用户

```sql
create user cbs identified by cbs default tablespace cbs
grant connect, resource to cbs
```

### 查看用户下的所有表

```sql
select OWNER, TABLE_NAME, TABLESPACE_NAME from all_tables where OWNER='CBS';
```

### Oracle查看DDL语句

```sql
SELECT DBMS_METADATA.GET_DDL('TABLE','DD_MST','CBS') FROM DUAL;
```

### 删除用户

```sql
# 将用户的数据库数据一并删除，并没有删除相应的表空间
drop user cbs CASCADE;
# 查看HWM
select file_name, ceil((nvl(hwm, 1) * 8192) / 1024 / 1024) as "HWM(MB)"
  from dba_data_files a,
       (select file_id, max(block_id + blocks - 1) hwm
          from dba_extents
         group by file_id) b
 where a.file_id = b.file_id(+);
# 将表空间设置到合适大小，即可释放磁盘空间
ALTER DATABASE DATAFILE "/home/oracle/app/oracle/oradata/orcl/users01.dbf" RESIZE 100M;
# 删除表空间，及对应的表空间文件也删除掉
# 如果多个用户使用一个表空间能否这样操作？
drop tablespace user including contents and datafiles cascade constraint;
```

### Oracle强制断开连接

```sql
select username,sid,serial# from v$session where username='CBS';
alter system kill session '268,12859' # SID,SERIAL#
```

### Oracle资源繁忙

```sql
# 查看什么进程占用了锁
select username,sid,serial#,logon_time from v$locked_object,v$session where v$locked_object.session_id=v$session.sid;
# 查看具体操作
select sql_text from v$session,v$sqltext_with_newlines where decode(v$session.sql_hash_value,0,prev_hash_value,sql_hash_value)=v$sqltext_with_newlines.hash_value and v$session.sid=137 order by piece;
# kill 会话
alter system kill session '137,9064';
# 或者提交操作
commit;
```

### Oracle清理undo log

```sql
# 创建一个新的undo tablespace
create undo tablespace undotBS2 datafile '/home/oracle/app/oracle/oradata/orcl/undotbs02.dbf' size 500m;
# 设置新undo tablespace
alter system set undo_tablespace=undotbs2 scope=both;
# 删除旧空间
drop tablespace UNDOTBS1 including contents and datafiles;
```

### Oracle数据库中删除了表空间物理文件XXX.ora后导致用drop tablespace删除表空间失败

```sql
alter database datafile '/home/oracle/jgbs/TBS_DATA_CDK_2_01.dbf' offline drop;
```
