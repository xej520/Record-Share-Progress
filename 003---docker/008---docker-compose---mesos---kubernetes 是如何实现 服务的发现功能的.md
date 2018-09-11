# 服务编排框架是如何实现服务的发现功能？ 
服务发现，服务代理，服务转发  是不是都是一样的呢？(或者说只是称呼不一样)  
![](https://note.youdao.com/yws/public/resource/005b9d9146ba9fe1df87371df7ef8da7/xmlnote/2AA3CDD6DFE548549ED457E11DA0C395/20122)    

是不是还有点 负载均衡的效果呢？

&ensp; 
## docker-compose  
通过名字来进行服务发现的
>通过docker-compose.yaml里的link机制，设置的名字来进行服务的发现的  

缺点？  
>所有的服务都必须部署在同一个节点上,这样link机制才生效的

&ensp; 
## mesos  
mesos的官方文档中，提供了两种服务发现机制？
- Marathon-lb
- MesosDNS  
### 一、Marathon-lb
通过端口进行服务发现的;  

    1. 需要给每个服务定义一个服务端口  
    2. 注册到Marathon-lb里  
    3. 使用服务端口访问marathon-lb的时候，就会转发到实际的服务提供者的IP和端口
Marathon-lb 属于一个集中式的服务发现机制，   
也就是说所有的服务都注册到Marathon-lb里，服务之间的端口是不一样的  
(是不是有点Nginx的意思呢)

### 二、MesosDNS  
通过名字来进行服务发现的;  
        
    1. 在所有slave节点上，需要安装MesosDNS的组件
    2. MesosDNS组件需要跟mesos master进行沟通，从而获得每个服务的IP和端口  
    3. 然后在当前slave节点上，添加一条A记录，从而实现容器间可以通过名字来进行相互访问  
不足之处？  
mesosdns组件已经很长时间未更新了    


&ensp; 
## kubernetes  
k8s里实现服务发现主要依赖两个组件:  
- kubeDNS  
- kube-proxy

### kubeDNS  
    是k8s的一个插件
    服务集群内部的dns解析，在集群内部可以让pod之间通过名字进行访问

### kube-proxy 
主要分为两种类型:  
- ClusterIP  
    - 对于每一个服务添加一个iptables 规则，对相关的pod做一个虚拟IP，对虚拟IP的流量重定向到后端服务的一个集合里，这个虚拟机IP只能在集群内部访问，是固定的
- NodePort  
    - 在每一个节点上都会启动一个端口，用来监听，映射到对应的服务，这样，集群外部就可以通过nodeIP:nodePort 来访问集群内部的服务了


&ensp; 
## DockerSwarm  (未学)