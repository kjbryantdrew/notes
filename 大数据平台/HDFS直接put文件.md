# 1. 创建文本表
```sql
CREATE TABLE preset.business_duebill (
  serialno STRING,
  ...
  lastinststdate STRING
)
--重点是下面的表结构设置
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
WITH SERDEPROPERTIES ('field.delim'=',', 'serialization.format'=',')
STORED AS TEXTFILE
LOCATION 'hdfs://nameservice1/user/hive/warehouse/preset.db/business_duebill'
```

# 2. put文件
```sh
hadoop fs -put <local_dir> <hdfs_dir>
```

> 注意：在put文件的时候，如果本地文本文件有包围符，需要将包围符去掉

```sh
sed -i "s/\"//g" <local_file>
```

# 3. 刷新元数据
```sql
invalidate metadata
```
