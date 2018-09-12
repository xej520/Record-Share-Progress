# Service  
### service对象的IP 和端口 是在哪里设置的？  
- 是在kube-apiserver进行控制的，  
也就是在kube-apiserver.service里的
--service-cluster-ip-range=10.68.0.0/16 \
#service的nodeport的端口范围限制
--service-node-port-range=20000-40000 \   
- Kubernetes Controller Manager 组件里也有设置  
