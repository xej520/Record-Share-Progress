# 部署kubernetes在AWS上  

说明：  
在AWS上部署kuberntes 可以使用下面的两个工具？  
- kubeadm 
- kops(是在建立在kubeadm的基础之上的) 

服务器选择？  
- 国内服务器的话，选择北京区域，不要选择 宁夏区域，因为宁夏区域目前没有kops 
- 

本文使用kops命令进行安装部署    
官网文档：  
https://kubernetes.io/docs/setup/custom-cloud/kops/  

# 安装部署工具kops  
## 在linux上安装 kops工具  
```
wget https://github.com/kubernetes/kops/releases/download/1.10.0/kops-linux-amd64
chmod +x kops-linux-amd64
mv kops-linux-amd64 /usr/local/bin/kops
```  
# 创建集群  
## 创建route53域  


## 创建一个S3存储  


## 构建集群配置  


## 开始创建集群在AWS上 




# 添加网络插件  

# 注意：  
- 认证时长？ 
- 是否必须按照S3， 不安装可以不？  
- 修改配置文件 

# 如何使用kops 来操作kubernetes 集群  











