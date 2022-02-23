```bash
# kafka查看消息
kafka-console-consumer --bootstrap-server node43:9092  --topic ogg_ens_sit

# kafka指定消费组
kafka-console-consumer --bootstrap-server node43:9092 --topic ens_ogg --consumer-property group.id=ogg_tunnel_ens
```

