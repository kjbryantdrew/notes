```sql
select count(*) from dba_log_groups where log_group_name like 'GGS_%'
```

> dba_log_groups记录的是各个表是如何记录redo log日志的， 通过ogg添加的日志组， 开头多是GGS