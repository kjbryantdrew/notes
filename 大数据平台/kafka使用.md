```bash
# kafka查看消息
kafka-console-consumer --bootstrap-server node43:9092  --topic ogg_ens_sit

# kafka指定消费组
kafka-console-consumer --bootstrap-server node43:9092 --topic ens_ogg --consumer-property group.id=ogg_tunnel_ens

# 删除消费组
kafka-consumer-groups --bootstrap-server node43:9092 --delete --group ogg_tunnel_ens
```

