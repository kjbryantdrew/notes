```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from __future__ import unicode_literals
from __future__ import absolute_import

import json
import time
from datetime import datetime, timedelta
from confluent_kafka import Consumer, TopicPartition, KafkaException


conf = {
    # 'bootstrap.servers': 'big-data01:9192,big-data02:9192,big-data03:9192,big-data04:9192',
    'bootstrap.servers': 'node42:9092,node43:9092,node44:9092',
    'group.id': 'read_test',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': False
}

now = datetime.now()
start_time = (now - timedelta(minutes=6)).replace(second=0,microsecond=0)
sample_time = start_time + timedelta(minutes=1)
end_time = start_time + timedelta(minutes=10)

# start_time = datetime.strptime('2022-05-17 22:30:00','%Y-%m-%d %H:%M:%S')
# end_time = datetime.strptime('2022-05-17 22:40:00','%Y-%m-%d %H:%M:%S')
start_time = datetime.strptime('2022-05-17 15:40:00','%Y-%m-%d %H:%M:%S')
end_time = datetime.strptime('2022-05-17 15:50:00','%Y-%m-%d %H:%M:%S')
# end_time = datetime.strptime('2022-05-17 22:30:10','%Y-%m-%d %H:%M:%S')


print('当前时间 %s 消费kafka时间段： %s - %s ，对比日志时间段： %s'
          % (now.strftime('%Y-%m-%d %H:%M:%S'),
            start_time.strftime('%Y-%m-%d %H:%M:%S'),
            end_time.strftime('%Y-%m-%d %H:%M:%S'),
            sample_time.strftime('%Y-%m-%d %H:%M:%S')))

topic = 'ens_ogg'
consumer = Consumer(conf)

c = consumer

tmp = c.list_topics(topic=topic).topics[topic].partitions

start_topic_partitions_to_search = list(
    map(lambda p: TopicPartition(topic, p, int(time.mktime(start_time.timetuple())*1000)), range(len(tmp))))
start_offset = c.offsets_for_times(start_topic_partitions_to_search)

end_topic_partitions_to_search = list(
    map(lambda p: TopicPartition(topic, p, int(time.mktime(end_time.timetuple())*1000)), range(len(tmp))))
end_offset = c.offsets_for_times(end_topic_partitions_to_search)


def read_kafka():
    #f = open('msg.txt', 'w')
    #
    for p in start_offset:
        c.assign([p])
        while True:
            msg = c.poll(1.0)
            if not msg:
                break
            if msg.error():
                raise KafkaException(msg.error())
            else:
                offset = msg.offset()
                if offset < end_offset[msg.partition()].offset:
                    #f.write(msg.value().decode() + '\n')
                    res = json.loads(msg.value().decode())
                    table = res['table']
                    # if table == 'ens_cbank.mb_acct_balance':
                    if table == 'ens_cbank.mb_acct':
                        internal_key = res['after']['INTERNAL_KEY']
                        print res
                        print internal_key
                        # if internal_key in ('21930539','21936168','21930541'):
                        #     print res
                        #     print internal_key
                else:
                    c.unassign()
                    break
                    
    c.close()
    #f.close()

read_kafka()
```

