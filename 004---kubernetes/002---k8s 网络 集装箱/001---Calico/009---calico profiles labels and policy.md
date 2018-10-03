# Docker Calico Profiles Labels Policy  
版本号:v2.6  
参考文献：   
https://docs.projectcalico.org/v2.6/getting-started/docker/tutorials/security-using-calico-profiles-and-policy  
****
## 一、 Security using Calico Profiles 

### 1. 创建网络  
在一台服务器上(如master节点)，创建net1,net2,net3  
```
docker network create --driver calico --ipam-driver calico-ipam net1
docker network create --driver calico --ipam-driver calico-ipam net2
docker network create --driver calico --ipam-driver calico-ipam net3
```

### 2. 创建容器
在master节点上：
```
docker run --net net1 --name workload-A -tid busybox
docker run --net net2 --name workload-B -tid busybox
docker run --net net1 --name workload-C -tid busybox
```
在node2节点上：  
```
docker run --net net3 --name workload-D -tid busybox
docker run --net net1 --name workload-E -tid busybox
```
说明：  
A,C,E 都在同一个网络应该可以彼此互ping的  
B和D都在各自的网络不能ping通任何其他节点  

### 3. 测试容器间的连通性 
在master上  
```
docker exec workload-A ping -c 4 workload-C.net1
docker exec workload-A ping -c 4 workload-E.net1
```  
测试workload-A容器ping workload-B容器  
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/4B722A9E9AEF42F6B39861433ECEF151/20246)  

在node2上 
获取workload-D的IP地址  
>docker inspect --format "{{ .NetworkSettings.Networks.net3.IPAddress }}" workload-D 

在master节点上  
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/4B722A9E9AEF42F6B39861433ECEF151/20246)  

### 总结：  
    calico网络，默认情况下，只有同一个网络内的容器是互通
&ensp; 
&ensp; 
&ensp; 
&ensp; 
&ensp; 
&ensp; 
****   
## 二、 Security using Calico Profiles and Policy  
### 2.1、Policy applied directly by the profile
1. create the docker networks  
    ```
    docker network create --driver calico --ipam-driver calico-ipam database
    docker network create --driver calico --ipam-driver calico-ipam frontend
    ```
2. create the profiles  
    vim frontend-database.yaml 
    ```
    - apiVersion: v1
    kind: profile
    metadata:
        name: database
        labels:
          role: database
    spec:
        ingress:
        - action: allow
          protocol: tcp
          source:
            selector: role == 'frontend'
          destination:
            ports:
            -  2181
        - action: allow
          source:
            selector: role == 'database'
        egress:
        - action: allow
          destination:
            selector: role == 'database'
    - apiVersion: v1
    kind: profile
    metadata:
        name: frontend
        labels:
          role: frontend
    spec:
        egress:
        - action: allow
          protocol: tcp
          destination:
            selector: role == 'database'
            ports:
            -  2181
    ```  
>说明：      
    &ensp;A、使用database网络的容器，都带有标签role=database  
    &ensp;B、使用frontend网络的容器，都带有标签role=frontend   
    &ensp;C、database 策略中：  
&ensp;&ensp;&ensp;&ensp;&ensp;a. ingress入规则定义了两个，只要满足一条规则即可访问   
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;规则一：(针对的是不同calico网络，访问database网络的规则)  
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;1. 源网络带有标签role=frontend,  
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;2. 访问的端口号是2181  
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;3. 使用tcp协议  
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;规则二：(针对的是同一个网络内，容器之间的访问)  
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;1. 带有标签role=database 也可以访问  
&ensp;&ensp;&ensp;&ensp;&ensp;b.egress出规则:  
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;只能访问同一个网络database内的容器，其他网络里的容器不能访问  
配置文件database的整体意思，就是：   
A、使用同一个database网络的容器可以互相访问  
B、针对外部网络，只允许frontend网络里的容器访问2181端口  


3. apply profile   
calicoctl apply -f frontend-database.yaml  

### 2.2、test profile  
1. 在master节点上创建一个zk容器  zk.sh  

    ```
    docker stop xej-zk
    docker rm xej-zk
    docker run --network=database  --name xej-zk -p 2181:2181 --restart always -d zookeeper:3.5
    ```
2. 在node2节点上创建一个test-tools容器，test-tools.sh
    ```
    #!/bin/bash 
    docker stop test-tools
    docker rm test-tools
    docker run --network=frontend --name test-tools -itd --restart always busybox
    ```
3. 在master节点上，查看xej-zk容器的IP地址： 
    docker inspect --format "{{ .NetworkSettings.Networks.database.IPAddress }}" xej-zk  
    172.20.219.118
4. 在node2节点上，查看test-tools 容器的IP地址: 
    docker inspect --format "{{ .NetworkSettings.Networks.frontend.IPAddress }}" test-tools  
    172.20.104.5

5. 在node2节点上 进入test-tools容器，测试是否能ping通master节点上的zk容器， 
    ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/5E1764C69A074D299A319409A2BF0996/20257)  
6. 在node2节点上，直接ping master节点上xej-zk容器，是否能ping 通？ 
    ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/7F41BC70D69442EBB8FD30CDD0962721/20250)  
7. 在master节点上，直接ping master节点上xej-zk容器，是否能ping 通？         
    ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/BD5223C7CBF14CF086E98ADFDF9926E2/20253)  
8. 在node2节点上，进入test-tools容器，访问master节点上的2181端口，是否可以？ 
    ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/618881A55B2C469EB4BFC2A85B0DD3E6/20259)  
    
&ensp;  
&ensp;  
&ensp;  

****   
## 三、 Global policy applied through label selection  
### 3.1、 Create the Docker networks  
```
docker network create --driver calico --ipam-driver calico-ipam db  
docker network create --driver calico --ipam-driver calico-ipam ft 
``` 

### 3.2、Create the profiles  
为db,ft网络自定义profile，创建各自标签  
pdbft.yaml
不定义规则  
```
- apiVersion: v1
  kind: profile
  metadata:
    name: db
    labels:
      role: db
- apiVersion: v1
  kind: profile
  metadata:
    name: ft
    labels:
      role: ft
```  
calicoctl apply -f pdbft.yaml  


### 3.3、Create policy   
通过创建全局策略，实现网络隔离  
Policy资源是全局性的， 同样包括一些列的ingress和egress规则 ， 
每条规则根据source和destination定义的属性进行数据包的过滤  
dbft.yaml   
```
- apiVersion: v1
  kind: policy
  metadata:
    name: db
  spec:
    order: 0
    selector: role == 'db'
    ingress:
    - action: allow
      protocol: tcp
      source:
        selector: role == 'ft'
      destination:
        ports:
        -  2181
    - action: allow
      source:
        selector: role == 'db'
    egress:
    - action: allow
      destination:
        selector: role == 'db'
- apiVersion: v1
  kind: policy
  metadata:
    name: ft
  spec:
    order: 0
    selector: role == 'ft'
    egress:
    - action: allow
      protocol: tcp
      destination:
        selector: role == 'db'
        ports:
        -  2181
```
### 3.4、apply policy 
>calicoctl apply -f dbft.yaml  
### 3.5、test policy  
实现的效果跟上面的测试用例是一样的  



****    
## 四、遗留问题？  
profile资源 与 policy资源 不知何时使用？  
什么场景下使用profile资源 ?  
什么场景下使用policy资源  
policy资源对象中的order 到底有什么用 ？




