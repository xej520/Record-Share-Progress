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
3. 如何将etcd的版本升级，如由3.2升级到3.3  
直接替换旧的etcd,etcdctl即可，如下
- wget https://github.com/etcd-io/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz  
- tar -zxvf etcd-v3.3.9-linux-amd64.tar.gz 
- cd etcd-v3.3.9-linux-amd64  
然后，将最新的etcd,etcdctl 替换掉旧的即可，
- 重新启动etcd服务  
    - systemctl restart etcd  
- 查看etcd的版本号:  
    - ETCDCTL_API=3 etcdctl version  
        ```
        etcdctl version: 3.3.9
        API version: 3.3
        ```