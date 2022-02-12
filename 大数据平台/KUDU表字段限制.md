- 修改配置

修改kudu-master的配置文件`/etc/kudu/conf/master.gflagfile`
增加
```
--max_num_columns=500
```

- 重启master和所有节点的tserver

