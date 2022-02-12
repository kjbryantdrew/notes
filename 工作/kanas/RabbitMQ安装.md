版本情况



```
Centos: 7.5 1804
CDH: 6.1.0
```



服务器



```
whbdp01: slaver
whbdp02: slaver
whbdp03: slaver
whbdp04: master
```



安装



```
rpm -ivh erlang-18.2-1.el6.x86_64.rpm
rpm -ivh rabbitmq-server-3.6.0-1.noarch.rpm
```



启动 whbdp04 上的 rabbitmq 成功启动后，会在 /var/lib/rabbitmq/ 目录下生成 cookie 文件



```
systemctl start rabbitmq-server.service
```



读取 whbdp04 上的 cookie, 复制到其他服务器相应目录



```
scp /var/lib/rabbitmq/.erlang.cookie whbdp01:/var/lib/rabbitmq/.erlang.cookie
ssh whbdp01
chown rabbitmq.rabbitmq /var/lib/rabbitmq/.erlang.cookie
```



将其他服务器节点加入已 whbdp04 为 master 的集群，在 whbdp01 上执行



```bash
systemctl start rabbitmq-server.service
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster rabbit@whbdp04
rabbitmqctl start_app
```



在 whbdp04 上查看集群状态

```bash
[root@whbdp04 rabbitmq]# rabbitmqctl cluster_status
Cluster status of node rabbit@whbdp04 ...
[{nodes,[{disc,[rabbit@whbdp01,rabbit@whbdp02,rabbit@whbdp03,
                rabbit@whbdp04]}]},
 {running_nodes,[rabbit@whbdp03,rabbit@whbdp02,rabbit@whbdp01,rabbit@whbdp04]},
 {cluster_name,<<"rabbit@whbdp04">>},
 {partitions,[]}]
[root@whbdp04 rabbitmq]#
```



设置

```bash
rabbitmqctl add_vhost /airflow
rabbitmqctl add_user airflow airflow
rabbitmqctl set_user_tags airflow administrator
rabbitmqctl set_permissions -p /airflow airflow ".*" ".*" ".*"


rabbitmqctl list_vhosts
rabbitmqctl list_permissions -p /whreport
rabbitmqctl list_user_permissions whrepor
```