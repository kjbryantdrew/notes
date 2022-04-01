# 基本命令

```bash
# 1.显示记录头
Logdump 1> GHDR ON
记录头中包含有记录对应的一些辅助信息，如操作类型、操作时间等

# 2.显示字段信息
Logdump 2> DETAIL ON
此开关打开之后，会显示数据对应的字段序号和ASCII值

# 3.增加HEX和ASCII数据到记录显示界面
Logdump 3> DETAIL DATA
```

# `filter`命令

```bash
# 过滤表
> filter include / exclude FILENAME FNSONLD.GLIF
# 过滤字段内容
> filter include / exclude string 'abc123'
# 匹配/清楚
> filter match/clear all
# 统计匹配数
> count
```

