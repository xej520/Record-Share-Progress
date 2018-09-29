# PaaS平台的网络需求 
## 网络模型应该满足：
- 让每个容器拥有自己的网络栈，特别是独立的IP地址  
- 能够进行跨服务器的容器间通信，同时不依赖特定的网络设备  
- 有访问控制机制，不同应用之间互相隔离，有调用关系的能够通信 

## 调研了几个主流的网络模型  
- Docker 原生的 Bridge 模型：NAT 机制导致无法使用容器 IP 进行跨服务器通讯（后来发现自定义网桥可以解决通讯问题，但是觉得方案比较复杂）
- Docker 原生的 Host 模型：大家都使用和服务器相同的 IP，端口冲突问题很麻烦  
- Weave OVS 等基于隧道的模型：由于是基于隧道的技术，在用户态进行封包解包，性能折损比较大，同时出现问题时网络抓包调试会很蛋疼 

## 网络方案？  
从基于实现的角度来分析的话，大概有两个方向？  
### 隧道方案 
通过隧道，或者说Overlay Networking的方式：  
- Weave，UDP广播，本机建立新的BR，通过PCAP互通。
- Open vSwitch（OVS），基于VxLAN和GRE协议，但是性能方面损失比较严重
- Flannel，UDP广播，VxLan   

隧道方案在IaaS层的网络中应用也比较多，大家共识是随着节点规模的增长复杂度会提升，而且出了网络问题跟踪起来比较麻烦，大规模集群情况下这是需要考虑的一个点  
### 路由方案 
典型的代表：  
- Calico
    - 基于BGP协议的路由方案，
    - 支持很细致的ACL控制，
    - 对混合云亲和度比较高  
- Macvlan
    - 从逻辑和Kernel层来看隔离性和性能最优的方案，
    - 基于二层隔离，所以需要二层路由器支持，
    - 大多数云服务商**不支持**，
    - 所以混合云上比较难以实现  

路由方案一般是从3层或者2层实现隔离和跨主机容器互通的，出了问题也很容易排查
****
## 网络模型  
- CNM 
- CNI 

### Docker Libnetwork Container Network Model（CNM）阵营 
- Docker Swarm overlay  
- Macvlan & IP network drivers  
- Calico  
- Contiv（from Cisco） 

cnm 优点？  
Docker Libnetwork的优势就是原生，而且和Docker容器生命周期结合紧密
CNM 缺点？  
缺点也可以理解为是原生，被Docker“绑架  
#### CNM的网络模型  ？  
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/79BEC5E433AC4D6CB7AF88FDB63A722D/20230)  
CNM基于3个主要概念： 
- Sandbox，包含容器网络栈的配置，包括Interface，路由表及DNS配置，对应的实现如：Linux Network Namespace；一个Sandbox可以包含多个Network；
- Endpoint，做为Sandbox接入Network的介质，对应的实现如：veth pair、TAP；一个Endpoint只能属于一个Network，也只能属于一个Sandbox；
- Network，一组可以相互通信的Endpoints；对应的实现如：Linux bridge、VLAN；Network有大量Endpoint资源组成；

CNM还需要依赖另外两个关键的对象来完成Docker的网络管理功能，他们分别是：
- NetworkController，对外提供分配及管理网络的APIs，Docker Libnetwork支持多个活动的网络driver，NetworkController允许绑定特定的driver到指定的网络；
- Driver，网络驱动对用户而言是不直接交互的，它通过插件式的接入来提供最终网络功能的实现；Driver（包括IPAM）负责一个Network的管理，包括资源分配和回收





### Container Network Interface（CNI）阵营

- Kubernetes  
- Weave  
- Macvlan  
- Flannel  
- Calico  
- Contiv  
- Mesos CNI  

CNI的优势  ?  
是兼容其他容器技术（e.g. rkt）及上层编排系统（Kuberneres & Mesos)，而且社区活跃势头迅猛，Kubernetes加上CoreOS主推；  
缺点？  
是非Docker原生



