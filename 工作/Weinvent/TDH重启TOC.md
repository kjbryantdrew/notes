```bash
# 相关服务重启
systemctl restart docker
systemctl restart kubelet
systemctl restart haproxy
```

然后再manager尝试重启各节点的toc，如果不行，重新加载**各节点**的docker镜像

```bash
docker load -i /etc/tos/conf/tos.tar.gz
```

然后再重启相关服务，进入前端重启toc

```bash
# 查看k8s各组件状态
[root@node42 ~]# kubectl get pods -n kube-system
NAME                        READY     STATUS             RESTARTS   AGE
tos-apiserver-tos-node42    1/1       Running            42         2y
tos-apiserver-tos-node43    1/1       Running            0          1y
tos-apiserver-tos-node44    1/1       Running            3          194d
tos-controller-tos-node42   1/1       Running            1155       2y
tos-controller-tos-node43   1/1       Running            1193       1y
tos-controller-tos-node44   1/1       Running            1179       194d
tos-etcd-tos-node42         1/1       Running            22         2y
tos-etcd-tos-node43         0/1       CrashLoopBackOff   69049      1y
tos-etcd-tos-node44         1/1       Running            2          194d
tos-proxy-tos-node42        1/1       Running            12         2y
tos-proxy-tos-node43        1/1       Running            0          1y
tos-proxy-tos-node44        1/1       Running            2          194d
tos-registry-tos-node42     1/1       Running            22         2y
tos-registryui-tos-node42   1/1       Running            20         2y
tos-scheduler-tos-node42    1/1       Running            1177       2y
tos-scheduler-tos-node43    1/1       Running            1244       1y
tos-scheduler-tos-node44    1/1       Running            362        194d
```

删除pod

```bash
kubectl delete pod tos-etcd-tos-node43 -n kube-system
```

查看docker进程

```bash
docker ps -a | grep etcd
```

