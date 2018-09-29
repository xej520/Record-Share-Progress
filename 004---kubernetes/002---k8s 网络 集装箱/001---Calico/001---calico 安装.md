# <center>Calico安装部署</center> 

## 当前环境
### 服务器情况 
| IP地址 | 主机名 |  网络模式  |系统|服务|
| :------|:------|:-------|:---|:---|
|172.16.91.205|calico01|桥接|centos7|calico|
|172.16.91.206|calico02|桥接|centos7|calico|
|172.16.91.207|calico03|桥接|centos7|calico|
|172.16.91.222|harbor|桥接|centos7|etcd|  
### 路由情况 
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/E98B4B11532547C588BD0B306BE178CC/20152)  

### iptables情况  
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/BFD42D34D8244F5A8DDAEAFA845CCD4B/20149)  

## 部署calico(calico01,calico02,calico03节点)  

### 下载calicoctl 


### 启动calico服务  


## 创建calico网络  

## 查看calico集群ip pool情况
>calicoctl  get ipPool -o yaml 


# 测试  
## 创建calico网络  
1. 查看当前主机的网络状态     
docker network ls   
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/CEFC2C05C691407CA3EA324174DAB94F/20163)  

2. 创建calico网络cal_net1  
docker network create --driver calico --ipam-driver calico-ipam cal_net1 
![](https://note.youdao.com/yws/public/resource/75a2d26febf925ba29e8f16ee51f93fd/xmlnote/5C90657D4AF041478E2E636103913046/20187)  
3. 在calico中运行容器 
docker run --net cal_net1 --name bbox1  -it -d busybox  
4. 进入bbox1容器  
docker exec -it bbox1 sh  
5. 查看bbox1的网络栈信息  
![](https://note.youdao.com/yws/public/resource/75a2d26febf925ba29e8f16ee51f93fd/xmlnote/74599A86EA984F3C8ABD713B7829E64F/20189)  
6. 查看bbox1的路由信息  
![](https://note.youdao.com/yws/public/resource/75a2d26febf925ba29e8f16ee51f93fd/xmlnote/178765744D53487ABA13CA4C8A2D5331/20192)  
7. 查看cal_net1网络设备的详细信息  
![](https://note.youdao.com/yws/public/resource/75a2d26febf925ba29e8f16ee51f93fd/xmlnote/AEA4D901C9CD4917BDF22FCD86612912/20194)  
![](https://note.youdao.com/yws/public/resource/75a2d26febf925ba29e8f16ee51f93fd/xmlnote/0BD3414B50B94E9991B163E336754282/20196)  
8. 测试在同一个节点上，同一个calico网络下，不同容器之间是否可以ping通？  
docker exec -it bbox2 ip addr  
docker exec -it bbox1 ip addr  
docker exec -it bbox1 ping 172.20.219.80 
![](https://note.youdao.com/yws/public/resource/75a2d26febf925ba29e8f16ee51f93fd/xmlnote/90534303A64540FEBE228181A1C7262C/20200) 
9. 测试同一个节点上，不同calico网络下，不同容器之间是否可以ping通呢？  
    ![](https://note.youdao.com/yws/public/resource/75a2d26febf925ba29e8f16ee51f93fd/xmlnote/45D98465EC304903B7EA6BF31751EEBF/20202) 
10. 测试 跨节点测试calico网络下不同容器之间的连通性？  
    ![](https://note.youdao.com/yws/public/resource/75a2d26febf925ba29e8f16ee51f93fd/xmlnote/5AD487C80F0C462CB41EE6A86EBD3365/20204)   
11. 使用calico网络后，主机的路由发生了什么变化？
    ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/502E0E292BA1453EB133BEA3F6DD2B22/20208)   
  
12.     








