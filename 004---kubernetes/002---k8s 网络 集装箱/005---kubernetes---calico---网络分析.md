
# 场景介绍   
- 场景一: 容器内可以访问k8s宿主机   
- 场景二：容器内可以访问k8s集群内非宿主机  
- 场景三：集群内的主机访问容器   
- 场景四：集群外在同一网段，可以访问容器  
- 场景五：集群外非同一网段，也可以访问容器    


# 创建Pod
kubectl run nginx --image=mybusybox --port 80 --expose --labels="app=web,role=nginx" --image-pull-policy=IfNotPresent   


## 场景一: 容器内可以访问k8s宿主机   

1. 检测calico网络情况  
![](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/ADAF6993918E4AD0B1E85BCDDD1B393E/22815)   

2. 如何设置成BGP模式？  默认是IPIP模式
![](https://note.youdao.com/yws/public/resource/4d55e910268f5b5d344e9126be9c53d0/xmlnote/5C2FFB7FBB3E47E6AB284461118BC59D/22817)  


3. 如何部署calicoctl 工具？  



