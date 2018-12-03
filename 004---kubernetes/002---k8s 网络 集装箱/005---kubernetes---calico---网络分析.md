
# 场景介绍   
- 场景一: 容器内可以访问k8s宿主机   
- 场景二：容器内可以访问k8s集群内非宿主机  
- 场景三：集群内的主机访问容器   
- 场景四：集群外在同一网段，可以访问容器  
- 场景五：集群外非同一网段，也可以访问容器    


# 创建Pod
kubectl run nginx --image=mybusybox --port 80 --expose --labels="app=web,role=nginx" --image-pull-policy=IfNotPresent   


## 场景一: 容器内可以访问k8s宿主机   








