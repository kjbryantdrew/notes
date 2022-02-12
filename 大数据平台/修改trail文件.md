```bash
DELETE EXTTRAIL ./dirdat/ee, EXTRACT EXTENS12
ADD EXTTRAIL ./dirdat/dd, EXTRACT EXTENS12
```

```bash
DELETE RMTTRAIL /home/ogg12/dirdat/dd, EXTRACT PPENS12
ADD RMTTRAIL /home/ogg12/dirdat/de , EXTRACT PPENS12
```

```bash
DELETE replicat reens
alter replicat reens, exttrail ./dirdat/de, checkpointtable ogg.checkpoint
# 重新设置文件序号和RBA
ALTER REPLICAT reens, EXTSEQNO 25, EXTRBA 0
# 如果只是重新设置rba，然后重新消费数据，需要添加NOFILTERDUPTRANSACTIONS参数
GGSCI (node43) 29> ALTER REPLICAT reens, EXTRBA 136501699

2021-08-12 16:14:10  INFO    OGG-06594  Replicat REENS has been altered. Even the start up position might be updated, duplicate suppression remains active in next startup. To override duplicate suppression, start REENS with NOFILTERDUPTRANSACTIONS option.

REPLICAT altered.


GGSCI (node43) 30> start reens NOFILTERDUPTRANSACTIONS

Sending START request to MANAGER ...
REPLICAT REENS starting
```
