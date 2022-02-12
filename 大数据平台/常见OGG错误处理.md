# 一、表字段长度变化

## 问题

```bash
# 示例错误
2021-08-12 13:08:21  ERROR   OGG-01163  Bad column length (5) specified for column AGREEMENT_TYPE in table ENS_CBANK.MB_AGREEMENT, maximum allowable length is 4.
```

## 解决

1. 重新生成def文件

```bash
# 1. 重新生成def文件
[ggs@hxdb ogg]$ ./defgen paramfile dirprm/ensdefgen_v12.prm NOEXTATTR
# 2.def复制到目标端
[ggs@hxdb ogg]$ scp dirdef/ens_v12.def  ogg12@160.161.12.43:~/dirdef/
```

2. 修改replicat配置

```bash
GGSCI (node43) 27> edit param reens

# 修改前
SOURCEDEFS ./dirdef/ens_v12.def
# 修改后
SOURCEDEFS ./dirdef/ens_v12.def OVERRIDE
```

> 虽然源端和目标端该字段的长度已修改，且def文件也都已修改，但生成的trail文件中的meta信息并不会更新。replicat进程默认按照trail文件中的meta信息进行操作。因此：注意一定添加OVERRIDE选项，新的def内容才能覆盖trail中的meta信息。