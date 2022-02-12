* 查看节点状态

```bash
hdfs haadmin -getServiceState nn1
```

* nn1指向的具体节点：

```bash
/etc/hdfs1/conf/hdfs-site.xml
```

* 查看namenode是否处于safemode状态：

```bash
hdfs dfsadmin -safemode get
```
