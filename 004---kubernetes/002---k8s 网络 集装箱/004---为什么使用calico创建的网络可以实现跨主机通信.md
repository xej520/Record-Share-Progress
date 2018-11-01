为什么docker容器使用Calico创建的网络可以实现跨主机通信呢？  

# 本篇文章的想法仅供参考  

docker容器要想实现跨主机通信的话，至少有两个问题要解决：  
- 不同主机间的容器是如何通信的？
    - 就是主机A的容器是如何找到/发现主机B上的容器的
- 不同主机间的容器网段 不能 __冲突__  

开源社区提供的方案有很多中，如flannel，calico，weave；  
本文从calico的角度进行简单的分析：  

问题一：  
    calico是通过路由进行查询的，并不像flannel，进行了再次封装。  
问题二：  
    将每个网段的信息存储在etcd或者consul里，calico根据网段信息，就可以从ip pool池里分配不同的子网段给每个节点。  
    docker的配置文件里，需要添加--cluster-store参数

# 测试环境介绍  
1. 物理环境介绍  

|**系统类型**|**IP**|**role**|**cpu**|**memory**|**hostname**|
|:---|:---|:---|:---|:---|:---|
|CentOS 7.4.1708|172.16.91.195|master|4|2G|master|
|CentOS 7.4.1708|172.16.91.196|slave1|2|1G|slave1|
|CentOS 7.4.1708|172.16.91.197|slave2|2|1G|slave2|  

2. 当前集群服务状态介绍 
![集群服务](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/6D64D781144B48B0BD6226E433A55715/21727)  

# 测试  
## 创建calico网络，以及2个测试容器  
1. 在master节点上创建一个calico网络  


2. 分别在master节点和node1节点上创建一个容器，   


3. 查看master节点和node1节点上的路由情况  


### 测试点1： calico类型的容器通过路由查询到对方  
1. 测试在master节点的web1容器ping node1节点上的web2容器   

2. 登陆到node1上，删除掉直连路由，查看master节点上的web1情况  

3. 大概等待90秒钟，calico会自动再次再node1上创建路由，再次查看web1情况  

说明，通过路由就可以跨主机通信了。  

### 测试点2： 不同节点上的网络段不冲突    
- 




1. 为什么在master节点上创建的calico网络，在其他节点上可以看的到  
- 在master节点上docker的配置文件里有--cluster-store参数的设置，可以设置为etcd或者consul类型， 
    - 用来存储一些集群信息，如网络信息  
- 在master节点上，创建calico类型的网络后，docker守护进程会判断网络的类型是local，global, host,null； 
    - 如果是global类型的话，会将这些网络信息存储到etcd的/docker/network/v1.0/目录下  
- 其他节点上，也必须配置--cluster-store参数，
    - 当其他节点node1,node2上使用docker network ls查询时，就会查询etcd里这个目录/docker/network/v1.0/  
- 其他节点上，创建容器时，都使用calico类型的网络，这样的话，在同一个calico网络里的容器就可以互相ping通
- calico类型的网络设备是全局的，而bridge类型的网络设备的作用域只能在一个节点上






