> 单master升级多master注意事项：
>
> 1. master个数需要为奇数，偶数主节点比起单主节点没任何优势
> 2. 命令行命令需要以kudu的系统用户执行
> 3. 从kudu1.15.0开始，增加了`kudu master add` 命令，方便升级multiple masters，但是当前环境kudu1.8不适用，无语😒
> 4. 升级之前，确定单master指定了`--master_addresses`，因为后续需要将该配置升级到多master，如果没指定，需要在升级之前指定，可以通过`kudu master get_flags`进行检查
> 5. kudu apache官方文档建议为kudu节点配置DNS别名，本环境有hosts配置，无需特殊处理那个的
> 6. 如果有impala集成的kudu表，需要更新对应的hive元数据 `hive metastore`