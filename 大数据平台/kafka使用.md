```bash
# kafka查看消息
kafka-console-consumer --bootstrap-server node43:9092  --topic ogg_ens_sit

# kafka指定消费组
kafka-console-consumer --bootstrap-server node43:9092 --topic ens_ogg --consumer-property group.id=ogg_tunnel_ens

kafka-console-consumer --bootstrap-server big-data01:9192 --topic ens_ogg --consumer-property group.id=ogg_tunnel_ens
kafka-console-consumer --bootstrap-server big-data01:9192 --topic ens_ogg --consumer-property group.id=ogg_tunnel_ens

# 删除消费组
kafka-consumer-groups --bootstrap-server node43:9092 --delete --group ogg_tunnel_ens

# 启动
JMX_PORT=9988 kafka-server-start -daemon /etc/kafka/server.properties

kafka-consumer-groups  --bootstrap-server localhost:9092 --group ogg_tunnel_ens --reset-offsets --to-latest --topic ens_ogg --execute
```
