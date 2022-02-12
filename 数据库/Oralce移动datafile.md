1、查看表空间的文件分布

```sql
SQL> select TABLESPACE_NAME,FILE_NAME,BYTES/1024/1024 "MB" from dba_data_files;
```

2、将表空间离线
```sql
SQL> alter tablespace users offline;
```

3、在操作系统下将数据文件移到另一位置
```bash
SQL> host mv /u01/app/oracle/oradata/ocp/users01.dbf /u02/
SQL> host ls /u02/
```

4、修改控制文件的记录指针

```sql
SQL> alter database rename file '/u01/app/oracle/oradata/ocp/users01.dbf' to '/u02/users01.dbf';
```
或者
```sql
SQL> alter tablespace users rename datafile '/u01/app/oracle/oradata/ocp/users01.dbf' to '/u02/users01.dbf';
```
注：执行此项时，目标文件（TO后面的那一段）一定要存在

5、将表空间在线
```sql
SQL> alter tablespace users online;
```
对于那些不能offline的表空间，只能关闭数据，在mount状态下修改，修改后再OPEN
