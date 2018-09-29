# <center>etcd命令</center>
## 注意etcd的v2版本和v3版本，命令不一样

1. 查看etcd集群健康状态？  
> ETCDCTL_API=3 etcdctl --endpoints=https://172.16.91.222:2379 --cacert=/root/ca/ca.pem --cert=/etc/kubernetes/ca/etcd/etcd.pem --key=/etc/kubernetes/ca/etcd/etcd-key.pem endpoint health  

2. 查看etcd的版本信息 
```
#ETCDCTL_API=3 etcdctl --endpoints=https://172.16.91.222:2379 --cacert=/root/ca/ca.pem --cert=/etc/kubernetes/ca/etcd/etcd.pem --key=/etc/kubernetes/ca/etcd/etcd-key.pem version

etcdctl version: 3.2.11
API version: 3.2
```