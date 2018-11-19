# 背景介绍   
&ensp;&ensp;&ensp;&ensp;前篇文章从网卡和路由的角度分析了calico网络下容器间是如何通信的；  
本文章主要从iptables角度去分析容器间是如何通信的。      
# 初始测试环境    
## 虚拟机    
|**系统类型**|**IP**|**role**|**cpu**|**memory**|**hostname**|
|:---|:---|:---|:---|:---|:---|
|CentOS 7.4.1708|172.16.91.225|master|4|2G|master|
|CentOS 7.4.1708|172.16.91.226|worker|2|1G|slave1|
|CentOS 7.4.1708|172.16.91.227|worker|2|1G|slave2|

## docker版本  
![docker版本](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/ED5265EFFCB14D03B7715B2EC6C3CF3C/22153)   

## calico集群状态       
![calico集群状态](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/CF5741C43BEF478D9FA3D131AD3920BE/22158)   

## etcd   
单节点部署   
![etcd版本](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/9E1AB6BDC08E448D893CA41EB7A1B495/22155)   

## nat表(测试前)   
![master节点nat表](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/E0043BEE3F5141B2B9DDC9A75C4A43A0/22161)      

## filter表(测试前)    
```
-P INPUT ACCEPT
-P FORWARD DROP
-P OUTPUT ACCEPT
-N DOCKER
-N DOCKER-ISOLATION-STAGE-1
-N DOCKER-ISOLATION-STAGE-2
-N DOCKER-USER
-N cali-FORWARD
-N cali-INPUT
-N cali-OUTPUT
-N cali-failsafe-in
-N cali-failsafe-out
-N cali-from-host-endpoint
-N cali-from-wl-dispatch
-N cali-to-host-endpoint
-N cali-to-wl-dispatch
-N cali-wl-to-host
-A INPUT -m comment --comment "cali:Cz_u1IQiXIMmKD4c" -j cali-INPUT
-A FORWARD -m comment --comment "cali:wUHhoiAYhphO9Mso" -j cali-FORWARD
-A FORWARD -j DOCKER-USER
-A FORWARD -j DOCKER-ISOLATION-STAGE-1
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A OUTPUT -m comment --comment "cali:tVnHkvAo15HuiPy0" -j cali-OUTPUT
-A DOCKER-ISOLATION-STAGE-1 -i docker0 ! -o docker0 -j DOCKER-ISOLATION-STAGE-2
-A DOCKER-ISOLATION-STAGE-1 -j RETURN
-A DOCKER-ISOLATION-STAGE-2 -o docker0 -j DROP
-A DOCKER-ISOLATION-STAGE-2 -j RETURN
-A DOCKER-USER -j RETURN
-A cali-FORWARD -i cali+ -m comment --comment "cali:X3vB2lGcBrfkYquC" -j cali-from-wl-dispatch
-A cali-FORWARD -o cali+ -m comment --comment "cali:UtJ9FnhBnFbyQMvU" -j cali-to-wl-dispatch
-A cali-FORWARD -i cali+ -m comment --comment "cali:Tt19HcSdA5YIGSsw" -j ACCEPT
-A cali-FORWARD -o cali+ -m comment --comment "cali:9LzfFCvnpC5_MYXm" -j ACCEPT
-A cali-FORWARD -m comment --comment "cali:7AofLLOqCM5j36rM" -j MARK --set-xmark 0x0/0xe000000
-A cali-FORWARD -m comment --comment "cali:QM1_joSl7tL76Az7" -m mark --mark 0x0/0x1000000 -j cali-from-host-endpoint
-A cali-FORWARD -m comment --comment "cali:C1QSog3bk0AykjAO" -j cali-to-host-endpoint
-A cali-FORWARD -m comment --comment "cali:DmFiPAmzcisqZcvo" -m comment --comment "Host endpoint policy accepted packet." -m mark --mark 0x1000000/0x1000000 -j ACCEPT
-A cali-INPUT -m comment --comment "cali:i7okJZpS8VxaJB3n" -m mark --mark 0x1000000/0x1000000 -j ACCEPT
-A cali-INPUT -i cali+ -m comment --comment "cali:JaoDb6CLdcGw8g0Y" -g cali-wl-to-host
-A cali-INPUT -m comment --comment "cali:c5eKVW2VdKQ_LiSM" -j MARK --set-xmark 0x0/0xf000000
-A cali-INPUT -m comment --comment "cali:hwQKYSlSCkpE_9uN" -j cali-from-host-endpoint
-A cali-INPUT -m comment --comment "cali:ttp8-serzKCP-bKZ" -m comment --comment "Host endpoint policy accepted packet." -m mark --mark 0x1000000/0x1000000 -j ACCEPT
-A cali-OUTPUT -m comment --comment "cali:YQSSJIsRcHjFbXaI" -m mark --mark 0x1000000/0x1000000 -j ACCEPT
-A cali-OUTPUT -o cali+ -m comment --comment "cali:KRjBsKsBcFBYKCEw" -j RETURN
-A cali-OUTPUT -m comment --comment "cali:3VKAQBcyUUW5kS_j" -j MARK --set-xmark 0x0/0xf000000
-A cali-OUTPUT -m comment --comment "cali:Z1mBCSH1XHM6qq0k" -j cali-to-host-endpoint
-A cali-OUTPUT -m comment --comment "cali:N0jyWt2RfBedKw3L" -m comment --comment "Host endpoint policy accepted packet." -m mark --mark 0x1000000/0x1000000 -j ACCEPT
-A cali-failsafe-in -p tcp -m comment --comment "cali:wWFQM43tJU7wwnFZ" -m multiport --dports 22 -j ACCEPT
-A cali-failsafe-in -p udp -m comment --comment "cali:LwNV--R8MjeUYacw" -m multiport --dports 68 -j ACCEPT
-A cali-failsafe-out -p tcp -m comment --comment "cali:73bZKoyDfOpFwC2T" -m multiport --dports 2379 -j ACCEPT
-A cali-failsafe-out -p tcp -m comment --comment "cali:QMFuWo6o-d9yOpNm" -m multiport --dports 2380 -j ACCEPT
-A cali-failsafe-out -p tcp -m comment --comment "cali:Kup7QkrsdmfGX0uL" -m multiport --dports 4001 -j ACCEPT
-A cali-failsafe-out -p tcp -m comment --comment "cali:xYYr5PEqDf_Pqfkv" -m multiport --dports 7001 -j ACCEPT
-A cali-failsafe-out -p udp -m comment --comment "cali:nbWBvu4OtudVY60Q" -m multiport --dports 53 -j ACCEPT
-A cali-failsafe-out -p udp -m comment --comment "cali:UxFu5cDK5En6dT3Y" -m multiport --dports 67 -j ACCEPT
-A cali-from-wl-dispatch -m comment --comment "cali:zTj6P0TIgYvgz-md" -m comment --comment "Unknown interface" -j DROP
-A cali-to-wl-dispatch -m comment --comment "cali:7KNphB1nNHw80nIO" -m comment --comment "Unknown interface" -j DROP
-A cali-wl-to-host -m comment --comment "cali:Ee9Sbo10IpVujdIY" -j cali-from-wl-dispatch
-A cali-wl-to-host -m comment --comment "cali:nSZbcOoG1xPONxb8" -m comment --comment "Configured DefaultEndpointToHostAction" -j ACCEPT
```
![filter表](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/02C2D50063C8422BB019A1F1AA496029/22164)   
# 测试准备    
## 部署tcpdump工具(master，slave节点)  
&ensp;&ensp;&ensp;&ensp;yum install -y tcpdump

## 镜像准备(master,slave1节点)    
1. 下载镜像  
    docker pull registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/mybusybox  

2. 修改tag  
    docker tag 252a8571b6b1 mybusybox
    ![准备镜像](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/2D82E9B71BAF416AA6496DDA16E05462/22166)   
3. 镜像说明  
&ensp;&ensp;&ensp;&ensp;本镜像主要提供了nginx服务，以及wget测试命令    
## 创建calico网络     
&ensp;&ensp;&ensp;&ensp;docker network create --driver calico --ipam-driver calico-ipam net1     
&ensp;&ensp;&ensp;&ensp;![创建calico网络](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/DD1539ED3D9B419DA31BC76EF51CCC3E/22168)     
## 创建测试容器    
1. master节点  
    docker run --name web1 --net net1 -itd mybusybox 
    docker run --name web2 --net net1 -itd mybusybox     
    ![创建容器](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/9A4AC5867D2A49B5A828AAE7EB88600E/22170)      
2. slave1节点   
    docker run --name web3 --net net1 -itd mybusybox   
    ![创建容器](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/C972301227064936AB917DA50550D964/22172)   

3. web1,web2，web3   

|**容器名字**|**IP**|**服务**|**网络**|**宿主机**|**宿主机IP**|
|:---|:---|:---|:---|:---|:---|
|web1|172.20.219.64|nginx|net1|master|172.16.91.225|
|web2|172.20.219.65|nginx|net1|slave1|172.16.91.226|

## 查看网卡、路由信息   
1. master节点
- ip a s
![master节点 网卡信息](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/21714030CFBD4F3FACCC9D0EE27AC21C/22174)  

- ip r s 
![master节点 路由信息](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/5769121DC76D46FF925C9369ADF7F042/22176)   


2. slave1节点   
- ip a s
![slave1节点 网卡信息](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/1237400A81EB489690C236A24027CA7A/22178)   

- ip r s 
![slave1节点，路由信息](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/C5B2712E064F462286715A4E293CCCFE/22180)   

## nat表(创建calico网络后)    
![nat表，发生了什么变化](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/074D69DD27414BBFB63587FEC8F3EE38/22184)    

## filter表(创建calico网络后)     
![filter表，发生了什么变化](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/61F9C51028F94746B7F22BBBDC73BBF5/22182)   

# 测试   
## 场景一：同一节点，物理节点与calico网络内的容器，如何通信？   


## 场景二: 同一节点，同网络内的容器, 如何通信？   


## 场景三：跨节点，如何与本节点的calico网络内容器，如何通信？   


## 场景四：跨节点，不同calico节点上，同一网络内的容器，如何通信？   



# calico+docker环境下，iptables链分析   
## 规则链名词解析   
|**名字**|**含义**|
|:---|:---|
|wl|workload endpoint简写<br>workload就是容器|
|fw|from workload endpoint简写|
|tw|to workload endpoint简写|
|from-XXX|XXX发出的报文|
|to-XXX|将数据报文发送到XXX|
|po|policy outbound|
|pi|policy inbound|
|pro|profile outbound|
|pri|profile inbound|

























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


9. -A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER？  
https://www.aliyun.com/jiaocheng/1388147.html  
把目标地址类型属于主机系统的本地网络地址的数据包,在数据包进入NAT表PREROUTING链时,都让它们直接jump到一个名为DOCKER的链

10. -j -g 参数的区别？    


11. -m conntrack --ctstate ESTABLISHED,RELATED -j ...  与  
-m state --state ESTABLISHED,RELATED -j ...  区别？   
http://blog.chinaunix.net/uid-27057175-id-5119553.html  
![](https://note.youdao.com/yws/public/resource/87831b99c8e3c62a12b6651dc393d95e/xmlnote/B3DF48FB8DCA42868648F84E09E7E8A4/22186)   
其实，就是对数据包的状态进行检测；   
已经建立tcp链接的包以及该连接相关的包允许通过！   
    数据包的状态，也可用作为匹配条件的    

12. 如何查看数据包的传输过程   
- iptables -t raw -A OUTPUT -p icmp -j TRACE
- iptables -t raw -A PREROUTING -p icmp -j TRACE   
- 如何查看追踪日志呢？  
    - /var/log/syslog
