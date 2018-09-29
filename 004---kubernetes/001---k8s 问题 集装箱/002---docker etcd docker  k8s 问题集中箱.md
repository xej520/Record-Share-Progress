# <center>docker and k8s问题集中箱</center>  
1. Failed to get resources: client: etcd cluster is unavailable or misconfigured; error #0: dial tcp 127.0.0.1:2379: getsockopt: connection refused **事故现场回顾** 
![](https://note.youdao.com/yws/public/resource/75a2d26febf925ba29e8f16ee51f93fd/xmlnote/B31D0C0EE27049738142E064261BEC8C/20168)  
**原因分析:** 
    因为calicoctl客户端去访问etcd集群时，不知道访问那个etcd集群，也就是说，缺少配置文件  
**解决措施**  
calicoctl默认查找的配置文件是：  
vim /etc/calico/calicoctl.cfg
当然也可用通过参数来指定--config  
calicoctl.cfg文件内容如下: 
    ```
    apiVersion: v1
    kind: calicoApiConfig
    metadata:
    spec:
    datastoreType: "etcdv2"
    etcdEndpoints: "http://172.16.91.175:2379,http://127.0.0.1:2379"  
    ```  

    如果etcd需要认证的话，可以使用下面的方式:  
    ```
    apiVersion: v1
    kind: calicoApiConfig
    metadata:
    spec:
    datastoreType: "etcdv2"
    etcdEndpoints: "https://192.168.204.3:2379"
    etcdKeyFile: "/etc/kubernetes/pki/kubernetes-key.pem"
    etcdCertFile: "/etc/kubernetes/pki/kubernetes.pem"
    etcdCACertFile: "/etc/kubernetes/pki/ca.pem"

    ```

    **总结**  
&ensp;&ensp;&ensp;&ensp;有点类似于kubectl， 同样需要配置文件才能访问k8s集群  
2. calico网络模式下，容器间ping不同的原因？  
    - 不同容器属于不同的calico网络，calico网络默认是隔离的 
    - 禁止ping这个命令，也就是禁止了icmp协议










