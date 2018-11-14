
# 背景说明   
&ensp;&ensp;&ensp;&ensp;本文分析docker+calico组合下的数据包走向分析

# 初始环境介绍  
## 物理环境介绍  
|**系统类型**|**IP**|**role**|**cpu**|**memory**|**hostname**|
|:---|:---|:---|:---|:---|:---|
|CentOS 7.4.1708|172.16.91.195|master|4|2G|master|
|CentOS 7.4.1708|172.16.91.196|worker|2|1G|slave1|
|CentOS 7.4.1708|172.16.91.197|worker|2|1G|slave2|  
## 运行服务介绍  
1. docker 版本介绍  
![docker版本](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/0B5DA5F8CDED41A697608B27F60C8CDA/21734)  

2. calico 版本介绍  
![calico版本](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/D819A38718114E3FB75E098E31110850/21736)  

3. etcd 版本介绍  
![etcd版本](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/3519301AC09144B284BF19457F23FEFE/21738)  


## 当前实例运行情况介绍
![master节点](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/22AD0D97EC204EA2AAA1F5CB0F50F41C/21745)  
![slave1节点](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/2FC5B9C4ECEA4C5682BE969C3B70D60D/21741)  
![slave2节点](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/271308858C3E49BD91BBCB492FE85632/21743)  

## 准备测试镜像  
1. 创建Dockerfile  

vim Dockerfile  
```
FROM nginx
RUN apt-get update 
RUN apt-get install -y iputils-ping iproute2 wget
```
2. 构建镜像  
```
docker build -t mybusybox .
```
## 测试工具  
tcpdump   
进行抓包   

# 单节点容器流量走向分析  

## 测试准备阶段

1. 创建calico网络net1、net2  
```
docker network create --driver calico --ipam-driver calico-ipam net1
docker network create --driver calico --ipam-driver calico-ipam net2
```
![create net1](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/94D18ECCAFAE46048D68740E218599C2/21747)  
2. 查看net1的默认策略  
```
calicoctl get profile net1 -oyaml 
```  
![net1 profile](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/5B040349DE7C4E3DAFBB5B8A4850F0B3/21750)  
3. calico采用的模式  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/7C7A8663FB2B451099DB0548F25EEA2E/21789)  

4. 创建容器  
```
docker run --net net1 --name web1 -itd  mybusybox    
docker run --net net1 --name web2 -itd  mybusybox  
docker run --net net2 --name web3 -itd  mybusybox  
```  
![create container web1, web2, web3](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/281C4F7354424156B06028FF9CA017A1/21753)  
## 新生成的网卡说明    
1. 查看master节点路由信息  
```
ip r s
```
![master节点](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/1EFD9E2CAEA6424DB322BD544158D2AB/21772)  
![web1容器](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/2C9B4C6239F5458BA4B7F75B43455ACA/21771)  
2. 新生成的网卡calif51b85e3399跟容器中的网卡的关系  
![master新生成的网卡跟web1网卡之间的关系](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/F289277413D04EFFA616B4AAFEA1BA45/21774)  
3. 删除直连路由？  
![删掉直连路由](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/0C7C03F70EBB4072BE07E4E7488B206C/21777)  


## 测试阶段  
### 同一个calico网络内的容器通信测试  
### 物理机跟容器之间的通信，是如何完成的？  
#### master节点 ping web1？  
0. 数据包流向图   
![数据包流向图](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/B4E2080EC698496A9D3F52137C495781/21792)  
1. master ping web1  
```
ping 172.20.219.82
```
2. 删除web1的直连路由   
```
route del -host 172.20.219.82 dev calif51b85e3399 
```

3. ens33网卡，calif51b85e3399网卡 抓包  
```
tcpdump -nn -i ens33 icmp and dst 
tcpdump -nn -i calif51b85e3399 icmp 
```  
![master ping web1](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/3C9826D594AA4F14AEAA7E98C5BE2A9F/21784)  
![tcpdump](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/48109C88CAB6492095E00CD7AAA4D08C/21782)  

4. 总结： 
    - 从物理机直接访问本节点上的容器，不会走物理机的网卡eth0或者ens33
    - 直接发到容器的网卡  



### 同一节点上，容器之间的通信，是如何完成的？  
0. 数据包流向图  
![数据包流向图](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/D752553303AB4D02B7DCCD29D8B7F8D9/21798)    
1. web1 ping web2  
![ping命令](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/21B92D75B29245E9BF17DD2CC4DB66FB/21796)

- 如果数据包的方向是，web1容器 ping web2容器的话，用的是  
172.20.219.83 dev cali6eea1d7a4a3 scope link  
- 如果数据包的方向是，web2容器 ping web1容器的话，用的是  
172.20.219.82 dev calif51b85e3399 scope link   
- 如果ping不通的话，可以查看对应的路由是否存在  

### 不同calico网络间的容器通信测试
1. web1 ping web3    
![不同Calico网络间容器通信](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/B19E78375C834847B8D9811C90E603F7/21803)     

# 跨节点容器流量走向分析  
## 在slave1节点上，创建一容器b-web1  
1. 创建容器b-web1(slave1节点操作)   
```
docker run --net net1 --name b-web1 -itd mybusybox  
```
![create b-web1 container](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/BB8ECD83B62C475398FF429A30F92E4E/21805)  
2. 查看slave1节点上路由、网卡信息(slave1节点操作)   
```
ip r s  
ip a s 
``` 
![路由，网卡信息](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/8D74388F42B0463CB099B47045035399/21808)    
3. 查看容器b-web1的网卡信息(slave1节点操作)   
```
docker exec -it b-web1 ip a s
```
![b-web1网卡信息](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/D820C118F2214DBC8EFE3D724860DFBA/21810)    
## 物理机master节点 ping b-web1 
0. 数据包流向图  
![数据包流向图](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/889AE636AAA5411F8DEFE327CDC985CB/21817)  

1. 登陆到master节点上(master节点操作)   
```
ping 172.20.140.193
```  
![物理机ping容器b-web1](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/20C2BB237A4141CD8EC8421A84A87043/21812)  
![物理机ping容器b-web1](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/026CAD1604F147AD9CA172630D12EE50/21815)   

## b-web1 ping web1(master节点操作)  
1. 数据包流向图  
![数据包流向图](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/4F7D2098AD734F0ABAF9C5BAF9D2EE45/21822)  
2. web1 ping b-web1   
```
docker exec -it web1 ping -c 2 b-web1     
```  
- master节点 上 进行抓包   
![web1 ping b-web1](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/C8015870FA7944AFB64B86E83B8764CA/21828)  
- slave1节点 上 进行抓包   
![web1 ping b-web1](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/8E861A5E21354955975E90700327DB31/21824)  

## 总结：   
- 默认情况下，其他节点物理IP不能访问本节点上的容器  
- 不同节点上的容器的IP是唯一的，对方可以看的到，不用进行NAT转换，从抓包中可以看出来，ip并没有进行变化。  




# 小知识点   
## 

1. 流量有几种状态？  
https://blog.csdn.net/leoysq/article/details/60579009    
- 新建NEW
- 已建立ESTABLISHED  
- 相关RELATED
- 无效INVALID
2. iptables数据包、连接标记模块MARK/CONNMARK的使用（打标签）  
https://www.cnblogs.com/EasonJim/p/8414943.html    
- ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/E653A57A12D7435984BF73B8BB082D58/22049)    

3. CNI 的实现可以被分成3种？  
- 所有 Pod 之间无需 NAT 即可互通
- 主机和 Pod 之间无需 NAT 即可互通
- Pod 自省的 IP 地址和之外部看到该 Pod 的地址一致   

4. Pod的网络必须满足以下3个条件？    
- 3 层路由实现
- Overlay 实现
- 2 层交换实现
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/26914967F78643EEBED7AD81C0F75D77/22063)  




## iptables 规则链 解析   
1. -A cali-fw-cali2bdb33ebbb0 -m comment --comment "cali:DFFsrMjLW8jAFQ7D" -m conntrack --ctstate INVALID -j DROP   
- 将删除所有具有“INVALID”状态匹配的流量    
- “DROP”目标将丢弃一个数据包而没有任何响应，与拒绝该数据包的REJECT相反。  
- 我们使用DROP，因为没有对INVALID的数据包的正确的“REJECT”响应，我们不想确认我们收到这些数据包

2. iptables -A INPUT -p icmp --icmp-type 8 -m conntrack --ctstate NEW -j ACCEPT    
- 将接受所有新的传入ICMP回显请求，也称为ping。   
- 只有第一个数据包计数为NEW，其余数据包将由RELATED，ESTABLISHED规则处理。   
- 由于计算机不是路由器，不需要允许具有状态NEW的其他ICMP流量。

3. iptables规则链默认策略的说明   
![链的默认策略说明](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/5CAA25D3397B462BBAD71245446FDEF7/22067)     


4. iptables规则的执行顺序   
- iptables执行规则时，是从规则表中从上至下顺序执行的，如果没遇到匹配的规则，就一条一条往下执行，  
- 如果遇到匹配的规则后，那么就执行本规则，执行后根据本规则的动作(accept, reject, log等)，决定下一步执行的情况，后续执行一般有三种情况：    

    - 一种是继续执行当前规则队列内的下一条规则。比如执行过Filter队列内的LOG后，还会执行Filter队列内的下一条规则。

    - 一种是中止当前规则队列的执行，转到下一条规则队列。比如从执行过accept后就中断Filter队列内其它规则，跳到nat队列规则去执行
    - 一种是中止所有规则队列的执行。

5. iptables 中-i, -o参数说明   
- -i参数说明
![iptables -i](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/6DB35A28CD6B4C6D83189200D5E832EB/22089)    

- -o参数说明   
![iptables -o](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/3000AD2B1083414F8A3B19EF57DD1AA0/22094)   

- 总结：  
    - INPUT,PREROUTING链，只能使用-i参数  
    - OUTPUT,POSTROUTING链，只能使用-o参数   
    - FORWARD链，-i, -o参数都可以使用的

6. PREROUTING, POSTROUTING链，跟路由表的关系  
    - 对于进来的数据包，先经过 PREROUTING， 然后再经过路由表  
    - 对于出去的数据包，先经过路由表， 再经过POSTROUTING

7. 容器发包出去的过程：  
- ip包会从container发往自己默认的网关docker0  
- __当包到达docker0时就时到达了主机__  
- 这时候会查询主机的 __路由表__，发现包应该从主机的网卡eth0出去   
![](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/0A6C6CA1A6CA404DB20B7A3B33D2D78A/22097)    
- 容器--->docker0--->路由表--->FORWARD--->postrouting

8. 网卡/网口转发配置  FORWARD    
- 转发配置，数据包的流向是由，内网到外网    
![FORWARD](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/4019CAAE633A4BD1BBF1BC67EC78EE68/22100)   
- 注意，在docker环境下，FORWARD链，跟其他链不一样，  其他链是单向的，要么进，要么出  
- 而FORWARD链，有两个方向：  
    - 数据包从本机到外面， 
    - 数据包从外面到本机   
    - 如何判断这条规则，对应的是 数据包由内到外？  还是  由外到内？     
        - 可以根据-i 接口设备，    
        - 查看接口设备 是 内网网卡，而是外网网卡     
            - 如果是内网网卡接口，说明是 由内到外   
            - 如果是外网网卡接口，说明是 由外到内     













