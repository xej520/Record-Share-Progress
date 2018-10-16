# Service  
### service对象的IP 和端口 是在哪里设置的？  
- 是在kube-apiserver进行控制的，  
也就是在kube-apiserver.service里的
--service-cluster-ip-range=10.68.0.0/16 \
#service的nodeport的端口范围限制
--service-node-port-range=20000-40000 \   
- Kubernetes Controller Manager 组件里也有设置  

service 是虚拟的  逻辑上的，底层是kube-proxy  
kube-proxy

# port, nodeport, targetport 的区别？ 
- port  
    The port that the service is exposed on the service’s cluster ip (virsual ip). Port is the service port which is accessed by others with cluster ip.  
　　即，这里的port表示：service暴露在cluster ip上的端口，<cluster ip>:port 是提供给集群内部客户访问service的入口。  
- nodePort  
　　On top of having a cluster-internal IP, expose the service on a port on each node of the cluster (the same port on each node). You'll be able to contact the service on any<nodeIP>:nodePortaddress. So nodePort is alse the service port which can be accessed by the node ip by others with external ip.  
　　首先，nodePort是kubernetes提供给集群外部客户访问service入口的一种方式（另一种方式是LoadBalancer），所以，<nodeIP>:nodePort 是提供给集群外部客户访问service的入口。  
- targetPort  
　　The port on the pod that the service should proxy traffic to.  
　　targetPort很好理解，targetPort是pod上的端口，从port和nodePort上到来的数据最终经过kube-proxy流入到后端pod的  targetPort上进入容器。  
