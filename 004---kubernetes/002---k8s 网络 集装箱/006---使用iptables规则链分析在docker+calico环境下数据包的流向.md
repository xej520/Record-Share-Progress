# 一、背景介绍  
&ensp;&ensp;&ensp;&ensp;在docker+calico环境下，分析数据包都经过了哪些iptables规则链。
分析目的：    
- 了解数据包的走向 
- 为接下来重点分析kubernetes+calico+docker环境下，数据包的走向打下基础   

&ensp;&ensp;&ensp;&ensp;本文分析使用iptables规则链，也就是分析数据包在网络层是如何走的，ebtables规则链暂不考虑。   
&ensp;&ensp;&ensp;&ensp;主要使用nat表，filter表。  
# 二、环境介绍  
## 2.1 物理环境介绍  
|**系统类型**|**IP**|**role**|**cpu**|**memory**|**hostname**|
|:---|:---|:---|:---|:---|:---|
|CentOS 7.4.1708|172.16.91.195|master|4|2G|master|
|CentOS 7.4.1708|172.16.91.196|worker|2|1G|slave1|
|CentOS 7.4.1708|172.16.91.197|worker|2|1G|slave2|  
## 2.2 运行服务介绍  
1. docker 版本介绍  
![docker版本](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/0B5DA5F8CDED41A697608B27F60C8CDA/21734)  

2. calico 版本介绍  
![calico版本](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/D819A38718114E3FB75E098E31110850/21736)  

3. etcd 版本介绍  
![etcd版本](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/3519301AC09144B284BF19457F23FEFE/21738)  

4. ip路由转发规则要开启
    - 经测试，如果不开启的话，calico集群的其他节点不当访问当前节点上的容器
    - 不开启的话，本节点访问当前节点上的容器是没有问题的。   
    - sysctl -a | grep net.ipv4.ip_forward  如果为1，说明开启了。   
    - vim /etc/sysctl.conf  
        - 添加net.ipv4.ip_forward = 1
        - sysctl -p (保存即可了)


# 三、准备工作  
## 3.1 设置iptables调试日志(master节电，slave1节点)     
### 步骤一 设置iptables规则

    iptables -t raw -A OUTPUT -p icmp -j LOG  
    iptables -t raw -A PREROUTING -p icmp -j LOG 
 
    iptables -t raw -A OUTPUT -p icmp -j TRACE   
    iptables -t raw -A PREROUTING -p icmp -j TRACE   

![设置iptables](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/503CE4312E4E43ADAFB811C0DD015994/22343)          
`备注：`   
- 不知道为什么，我这里必须同时设置LOG,TRACE这两个动作，调式日志才能打印出来  
- 重启虚拟机后，这些规则会消失   
### 步骤二  
设置日志存储路径  
```
vim /etc/rsyslog.conf    
kern.*  /var/log/iptables.log  
```
### 步骤二
重启rsyslog服务   
```
service rsyslog status  
service rsyslog restart
```
## 3.2 镜像准备   
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

## 3.3 创建测试容器   

1. 创建calico网络net1(master节点)  
    ```
    docker network create --driver calico --ipam-driver calico-ipam net1
    ```
2. 创建容器(master节点)  
    ```
    docker run --net net1 --name web1 -itd  mybusybox    
    docker run --net net1 --name web2 -itd  mybusybox  
    ```  
3. 创建容器(slave1节点)  
    ```
    docker run --net net1 --name web3 -itd  mybusybox      
    ``` 

# 四、测试  
下图展示了iptables规则链的方向:  
![iptables规则链](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/B2B5B81747E84FC58A15940F8D126954/22340)    
我的困惑是，在calico+docker环境下，数据包是如何在网卡之间进行通信的，方向是什么？   
所以才有了下面的场景测试，希望通过这些测试，了解数据包的走向。   
`为什么，要了解一下呢？`  
好多上层应用功能的实现，底层几乎都是`iptables规则链`来实现的。如:  
- kubernetes中的networkpolicy  
- calico中的profile  
- kubernetes中的kube-proxy的原理   
如果对`iptables，ipset`有了深入的了解，假如线上生产环境出了相关问题，知道如何去排查。   

`接下来，从以下场景进行测试:`  
- 容器与宿主机之间的通信   
- 容器与calico集群中非宿主机之间的通信  
- 同节点上，不同容器之间的通信  
- 跨节点环境下，不同容器之间的通信   


## 4.1 场景一: 从宿主机到容器   
1. 展示图：  
![从宿主机到容器](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/F2A287B846CC47BC8BA89213C3A11D93/22346)   
2. ping -c 1 172.20.219.80  
![ping](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/D9419572DF764E42B0E229A7001CFC31/22348)  
3. 查看ens33, 以及web1容器对应的网卡
![ens33网卡](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/E5DFE4C2C1684937BA0E4F8290D432A1/22352)  
![web1网卡](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/87FF42A6965F41CDA89BF431242A439E/22350)   
4. tai -f /var/log/iptables   
调试日志，很多，只留下nat，filter表日志，其他表日志删除。   
```
Dec  1 03:35:44 localhost kernel: TRACE: nat:OUTPUT:rule:1 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45708 DF PROTO8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 
Dec  1 03:35:44 localhost kernel: TRACE: nat:cali-OUTPUT:rule:1 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45708 DF TYPE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 
Dec  1 03:35:44 localhost kernel: TRACE: nat:cali-fip-dnat:return:1 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45708CMP TYPE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 
Dec  1 03:35:44 localhost kernel: TRACE: nat:cali-OUTPUT:return:2 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45708 DP TYPE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 
Dec  1 03:35:44 localhost kernel: TRACE: nat:OUTPUT:policy:3 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45708 DF PROE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 

Dec  1 03:35:44 localhost kernel: TRACE: filter:OUTPUT:rule:1 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45708 DF PRPE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 
Dec  1 03:35:44 localhost kernel: TRACE: filter:cali-OUTPUT:rule:2 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45708 MP TYPE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 
Dec  1 03:35:44 localhost kernel: TRACE: filter:OUTPUT:policy:2 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45708 DF TYPE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 

Dec  1 03:35:44 localhost kernel: TRACE: nat:POSTROUTING:rule:1 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45708 DF TYPE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 
Dec  1 03:35:44 localhost kernel: TRACE: nat:cali-POSTROUTING:rule:1 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=4570ICMP TYPE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 
Dec  1 03:35:44 localhost kernel: TRACE: nat:cali-fip-snat:return:1 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45708CMP TYPE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 
Dec  1 03:35:44 localhost kernel: TRACE: nat:cali-POSTROUTING:rule:2 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=4570ICMP TYPE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 
Dec  1 03:35:44 localhost kernel: TRACE: nat:cali-nat-outgoing:return:2 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=4TO=ICMP TYPE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 
Dec  1 03:35:44 localhost kernel: TRACE: nat:cali-POSTROUTING:return:3 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45O=ICMP TYPE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 
Dec  1 03:35:44 localhost kernel: TRACE: nat:POSTROUTING:policy:3 IN= OUT=calid55c454dfaa SRC=172.16.91.225 DST=172.20.219.80 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45708 DP TYPE=8 CODE=0 ID=3584 SEQ=1 UID=0 GID=0 

Dec  1 03:35:44 localhost kernel: TRACE: filter:INPUT:rule:1 IN=calid55c454dfaa OUT= MAC=d6:c3:32:3b:a6:bb:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.80 DST=172.16.91.225 LE00 PREC=0x00 TTL=64 ID=29622 PROTO=ICMP TYPE=0 CODE=0 ID=3584 SEQ=1 MARK=0x4000000 
Dec  1 03:35:44 localhost kernel: TRACE: filter:cali-INPUT:rule:2 IN=calid55c454dfaa OUT= MAC=d6:c3:32:3b:a6:bb:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.80 DST=172.16.91.2OS=0x00 PREC=0x00 TTL=64 ID=29622 PROTO=ICMP TYPE=0 CODE=0 ID=3584 SEQ=1 MARK=0x4000000 
Dec  1 03:35:44 localhost kernel: TRACE: filter:cali-wl-to-host:rule:1 IN=calid55c454dfaa OUT= MAC=d6:c3:32:3b:a6:bb:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.80 DST=172.16=84 TOS=0x00 PREC=0x00 TTL=64 ID=29622 PROTO=ICMP TYPE=0 CODE=0 ID=3584 SEQ=1 MARK=0x4000000 
Dec  1 03:35:44 localhost kernel: TRACE: filter:cali-from-wl-dispatch:rule:10 IN=calid55c454dfaa OUT= MAC=d6:c3:32:3b:a6:bb:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.80 DST225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=29622 PROTO=ICMP TYPE=0 CODE=0 ID=3584 SEQ=1 MARK=0x4000000 
Dec  1 03:35:44 localhost kernel: TRACE: filter:cali-from-wl-dispatch-d:rule:1 IN=calid55c454dfaa OUT= MAC=d6:c3:32:3b:a6:bb:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.80 DS.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=29622 PROTO=ICMP TYPE=0 CODE=0 ID=3584 SEQ=1 MARK=0x4000000 
Dec  1 03:35:44 localhost kernel: TRACE: filter:cali-fw-calid55c454dfaa:rule:1 IN=calid55c454dfaa OUT= MAC=d6:c3:32:3b:a6:bb:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.80 DS.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=29622 PROTO=ICMP TYPE=0 CODE=0 ID=3584 SEQ=1 MARK=0x4000000
```  
![TYPE=8 CODE=0与TYPE=0 CODE=0说明](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/56D9B43D7C8D4ACF936FB3EC57A4F065/22354)     

`将上面的iptables 日志，转换成下面的流向图:`  
![从宿主机到容器的数据包](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/F9BBEBC1FF554879AE8DD8E1D2773599/22356)      

`iptables日志格式说明`  
![iptables日志格式说明](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/87944C6F7BDE466088D1135026993977/22358)  
https://blog.csdn.net/liukuan73/article/details/78635655
![iptables日志格式说明](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/440AF221A1F9438EB3479A380B405B2F/22360)   

5. 总结(从宿主机到容器的数据包流向图):  
- 从`宿主机到容器` 走的是OUTPUT ---> POSTROUTING  
- `容器给宿主机`的答复，走的是 INPUT链  
- 只能是主链之间的跳转，不能是子链调主链   


## 4.2 场景二：从容器到宿主机   
1. 数据包流向图:  
![从容器到宿主机数据包流向图](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/E02B340446E141E48FB3B155988FEE93/22362)    

2. docker exec -it web1 ping -c 1 172.16.91.225
![ping](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/FD1FD461CD1344F086E3E59FA83F167E/22364)   

3. tail -f iptables.log  
```
Dec  2 03:10:15 master kernel: TRACE: nat:PREROUTING:rule:1 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: nat:cali-PREROUTING:rule:1 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: nat:cali-fip-dnat:return:1 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: nat:cali-PREROUTING:return:2 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: nat:PREROUTING:rule:2 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: nat:DOCKER:return:2 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: nat:PREROUTING:policy:3 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 


Dec  2 03:10:15 master kernel: TRACE: filter:INPUT:rule:1 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: filter:cali-INPUT:rule:2 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: filter:cali-wl-to-host:rule:1 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: filter:cali-from-wl-dispatch:rule:3 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: filter:cali-from-wl-dispatch-2:rule:1 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: filter:cali-fw-cali282b0776936:rule:3 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: filter:cali-fw-cali282b0776936:rule:4 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: filter:cali-pro-net1:rule:1 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x4000000 
Dec  2 03:10:15 master kernel: TRACE: filter:cali-pro-net1:rule:2 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x5000000 
Dec  2 03:10:15 master kernel: TRACE: filter:cali-fw-cali282b0776936:rule:5 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x5000000 
Dec  2 03:10:15 master kernel: TRACE: filter:cali-wl-to-host:rule:2 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x5000000 

Dec  2 03:10:15 master kernel: TRACE: nat:INPUT:policy:1 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45306 DF PROTO=ICMP TYPE=8 CODE=0 ID=12 SEQ=1 MARK=0x5000000 

Dec  2 03:10:15 master kernel: TRACE: filter:OUTPUT:rule:1 IN= OUT=cali282b0776936 SRC=172.16.91.225 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=20759 PROTO=ICMP TYPE=0 CODE=0 ID=12 SEQ=1 
Dec  2 03:10:15 master kernel: TRACE: filter:cali-OUTPUT:rule:2 IN= OUT=cali282b0776936 SRC=172.16.91.225 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=20759 PROTO=ICMP TYPE=0 CODE=0 ID=12 SEQ=1 
Dec  2 03:10:15 master kernel: TRACE: filter:OUTPUT:policy:2 IN= OUT=cali282b0776936 SRC=172.16.91.225 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=20759 PROTO=ICMP TYPE=0 CODE=0 ID=12 SEQ=1 

```   
![iptables规则说明](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/7AE28A8210D4498FB8E20C1C9BAAC89D/22369)   
4. 将上面的日志，转换成下面的流向图    
![流向图说明](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/1DC6B8EEBACE47C29294FB8BC0B973D1/22366) 

5. 总结(从容器到宿主机数据包流向图)
- 从`容器到宿主机`走的是路线是: PREOUTING ---> INPUT
- `宿主机 答复 容器`的 走的路线是:  OUTPUT

## 4.3 场景三：从容器访问calico集群中的非宿主机   
1. 数据包流向图:  
![从容器到宿主机数据包流向图](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/BCDA416A56894AF885CF791E58A904F2/22373)    

2. docker exec -it web1 ping -c 1 172.16.91.226
![web1容器 ping slave1](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/1D60B3BD168149008C1442D9D116DB5C/22376)   

3. tail -f iptables.log  
master节点上iptables的日志
```
Dec  2 03:38:09 master kernel: TRACE: nat:PREROUTING:rule:1 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x4000000 
Dec  2 03:38:09 master kernel: TRACE: nat:cali-PREROUTING:rule:1 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x4000000 
Dec  2 03:38:09 master kernel: TRACE: nat:cali-fip-dnat:return:1 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x4000000 
Dec  2 03:38:09 master kernel: TRACE: nat:cali-PREROUTING:return:2 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x4000000 
Dec  2 03:38:09 master kernel: TRACE: nat:PREROUTING:policy:3 IN=cali282b0776936 OUT= MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x4000000 

Dec  2 03:38:09 master kernel: TRACE: filter:FORWARD:rule:1 IN=cali282b0776936 OUT=ens33 MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x4000000 
Dec  2 03:38:09 master kernel: TRACE: filter:cali-FORWARD:rule:1 IN=cali282b0776936 OUT=ens33 MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x4000000 
Dec  2 03:38:09 master kernel: TRACE: filter:cali-from-wl-dispatch:rule:3 IN=cali282b0776936 OUT=ens33 MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x4000000 
Dec  2 03:38:09 master kernel: TRACE: filter:cali-from-wl-dispatch-2:rule:1 IN=cali282b0776936 OUT=ens33 MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x4000000 
Dec  2 03:38:09 master kernel: TRACE: filter:cali-fw-cali282b0776936:rule:3 IN=cali282b0776936 OUT=ens33 MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x4000000 
Dec  2 03:38:09 master kernel: TRACE: filter:cali-fw-cali282b0776936:rule:4 IN=cali282b0776936 OUT=ens33 MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x4000000 
Dec  2 03:38:09 master kernel: TRACE: filter:cali-pro-net1:rule:1 IN=cali282b0776936 OUT=ens33 MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x4000000 
Dec  2 03:38:09 master kernel: TRACE: filter:cali-pro-net1:rule:2 IN=cali282b0776936 OUT=ens33 MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x5000000 
Dec  2 03:38:09 master kernel: TRACE: filter:cali-fw-cali282b0776936:rule:5 IN=cali282b0776936 OUT=ens33 MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x5000000 
Dec  2 03:38:09 master kernel: TRACE: filter:cali-FORWARD:rule:3 IN=cali282b0776936 OUT=ens33 MAC=4a:0c:7a:74:5d:26:ee:ee:ee:ee:ee:ee:08:00 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x5000000 

Dec  2 03:38:09 master kernel: TRACE: nat:POSTROUTING:rule:1 IN= OUT=ens33 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x5000000 
Dec  2 03:38:09 master kernel: TRACE: nat:cali-POSTROUTING:rule:1 IN= OUT=ens33 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x5000000 
Dec  2 03:38:09 master kernel: TRACE: nat:cali-fip-snat:return:1 IN= OUT=ens33 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x5000000 
Dec  2 03:38:09 master kernel: TRACE: nat:cali-POSTROUTING:rule:2 IN= OUT=ens33 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x5000000 
Dec  2 03:38:09 master kernel: TRACE: nat:cali-nat-outgoing:rule:1 IN= OUT=ens33 SRC=172.20.219.81 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 MARK=0x5000000 

Dec  2 03:38:09 master kernel: TRACE: filter:FORWARD:rule:1 IN=ens33 OUT=cali282b0776936 MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=45474 PROTO=ICMP TYPE=0 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 master kernel: TRACE: filter:cali-FORWARD:rule:2 IN=ens33 OUT=cali282b0776936 MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=45474 PROTO=ICMP TYPE=0 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 master kernel: TRACE: filter:cali-to-wl-dispatch:rule:3 IN=ens33 OUT=cali282b0776936 MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=45474 PROTO=ICMP TYPE=0 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 master kernel: TRACE: filter:cali-to-wl-dispatch-2:rule:1 IN=ens33 OUT=cali282b0776936 MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=45474 PROTO=ICMP TYPE=0 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 master kernel: TRACE: filter:cali-tw-cali282b0776936:rule:1 IN=ens33 OUT=cali282b0776936 MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=45474 PROTO=ICMP TYPE=0 CODE=0 ID=17 SEQ=1 

```
slave1节点上iptables的日志
```
Dec  2 03:38:09 slave1 kernel: TRACE: nat:PREROUTING:rule:1 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: nat:cali-PREROUTING:rule:1 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: nat:cali-fip-dnat:return:1 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: nat:cali-PREROUTING:return:2 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: nat:PREROUTING:rule:2 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: nat:DOCKER:return:2 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: nat:PREROUTING:policy:3 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 

Dec  2 03:38:09 slave1 kernel: TRACE: filter:INPUT:rule:1 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: filter:cali-INPUT:rule:3 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: filter:cali-INPUT:rule:4 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: filter:cali-from-host-endpoint:return:1 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: filter:cali-INPUT:return:6 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: filter:INPUT:policy:2 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: nat:INPUT:policy:1 IN=ens33 OUT= MAC=00:0c:29:5e:b2:88:00:0c:29:2e:0f:7b:08:00 SRC=172.16.91.225 DST=172.16.91.226 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=13891 DF PROTO=ICMP TYPE=8 CODE=0 ID=17 SEQ=1 

Dec  2 03:38:09 slave1 kernel: TRACE: filter:OUTPUT:rule:1 IN= OUT=ens33 SRC=172.16.91.226 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45474 PROTO=ICMP TYPE=0 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: filter:cali-OUTPUT:rule:3 IN= OUT=ens33 SRC=172.16.91.226 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45474 PROTO=ICMP TYPE=0 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: filter:cali-OUTPUT:rule:4 IN= OUT=ens33 SRC=172.16.91.226 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45474 PROTO=ICMP TYPE=0 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: filter:cali-to-host-endpoint:return:1 IN= OUT=ens33 SRC=172.16.91.226 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45474 PROTO=ICMP TYPE=0 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: filter:cali-OUTPUT:return:6 IN= OUT=ens33 SRC=172.16.91.226 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45474 PROTO=ICMP TYPE=0 CODE=0 ID=17 SEQ=1 
Dec  2 03:38:09 slave1 kernel: TRACE: filter:OUTPUT:policy:2 IN= OUT=ens33 SRC=172.16.91.226 DST=172.16.91.225 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=45474 PROTO=ICMP TYPE=0 CODE=0 ID=17 SEQ=1 

```

4. 将上面的日志，转换成下面的流向图    
![master节点流向图说明](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/387840D73E114452BC5B6A4FA1C02C60/22388)  
![slave1节点流向图说明](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/DE703D87679441ACB4865B59FAAC5854/22386)   

5. 总结(从容器到宿主机数据包流向图)  
- 从`web1容器到slave1宿主机`走的是路线是: 
    - master节点
        - PREROUTING ---> FORWARD ---> POSTROUTING 
    - slave1节点上 
        - PREROUTING ---> INPUT   
- `slave1宿主机 答复 web1容器`的 走的路线是:  
    - slave1节点上
        - OUTPUT
    - master节点上
        - FORWARD   

## 4.4 场景四：从calico集群中非宿主机访问容器    
1. 数据包流向图:  
![从容器到宿主机数据包流向图](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/B4A06FC17156407A9DB16EBA6F6287E6/22392)    

2. ping -c 1 172.20.219.81
![slave1 ping web1](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/8C46F2AC68514601A6BB5941FE1C5DF9/22397)   
默认情况下，是ping不通的 
![slave1网卡ens33](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/280836AED2724E8086E4B8FD9E299852/22399)   
3. 将上面的日志，转换成下面的流向图    
![slave1节点流向图说明](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/E31C228440CA486690D9DEA9A1E83C38/22401)    
![master节点流向图说明](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/F8B3D82A2B784D9E9F15245928D8982D/22403)   

4. 总结(从容器到宿主机数据包流向图)
- 从`slave1宿主机 到 web1容器`走的是路线是: 
    - slave1节点上 
        - OUTPUT ---> POSTROUTING  
    - master节点上  
        - PREROUTING ---> FORWARD   
5. 解析下，为什么默认情况下，slave1节点ping不同master节点上的web容器  
默认情况下profile策略里规定了进入容器的规则里，含有标签net1，也就是同一个calico网络里的容器才能访问，而slave1节点不属于net1网络，因此进不去。  
![profile](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/55BD0CCF69284B59B507477929C8379D/22424)   
下面从iptables规则链里查看原因？   
master节点上，查看iptables的日志，如下:  
```
Dec  2 04:20:14 master kernel: TRACE: nat:PREROUTING:rule:1 IN=ens33 OUT= MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=21102 DF PROTO=ICMP TYPE=8 CODE=0 ID=2916 SEQ=1 
Dec  2 04:20:14 master kernel: TRACE: nat:cali-PREROUTING:rule:1 IN=ens33 OUT= MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=21102 DF PROTO=ICMP TYPE=8 CODE=0 ID=2916 SEQ=1 
Dec  2 04:20:14 master kernel: TRACE: nat:cali-fip-dnat:return:1 IN=ens33 OUT= MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=21102 DF PROTO=ICMP TYPE=8 CODE=0 ID=2916 SEQ=1 
Dec  2 04:20:14 master kernel: TRACE: nat:cali-PREROUTING:return:2 IN=ens33 OUT= MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=21102 DF PROTO=ICMP TYPE=8 CODE=0 ID=2916 SEQ=1 
Dec  2 04:20:14 master kernel: TRACE: nat:PREROUTING:policy:3 IN=ens33 OUT= MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=21102 DF PROTO=ICMP TYPE=8 CODE=0 ID=2916 SEQ=1 

Dec  2 04:20:14 master kernel: TRACE: filter:FORWARD:rule:1 IN=ens33 OUT=cali282b0776936 MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=21102 DF PROTO=ICMP TYPE=8 CODE=0 ID=2916 SEQ=1 
Dec  2 04:20:14 master kernel: TRACE: filter:cali-FORWARD:rule:2 IN=ens33 OUT=cali282b0776936 MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=21102 DF PROTO=ICMP TYPE=8 CODE=0 ID=2916 SEQ=1 
Dec  2 04:20:14 master kernel: TRACE: filter:cali-to-wl-dispatch:rule:3 IN=ens33 OUT=cali282b0776936 MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=21102 DF PROTO=ICMP TYPE=8 CODE=0 ID=2916 SEQ=1 
Dec  2 04:20:14 master kernel: TRACE: filter:cali-to-wl-dispatch-2:rule:1 IN=ens33 OUT=cali282b0776936 MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=21102 DF PROTO=ICMP TYPE=8 CODE=0 ID=2916 SEQ=1 
Dec  2 04:20:14 master kernel: TRACE: filter:cali-tw-cali282b0776936:rule:3 IN=ens33 OUT=cali282b0776936 MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=21102 DF PROTO=ICMP TYPE=8 CODE=0 ID=2916 SEQ=1 
Dec  2 04:20:14 master kernel: TRACE: filter:cali-tw-cali282b0776936:rule:4 IN=ens33 OUT=cali282b0776936 MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=21102 DF PROTO=ICMP TYPE=8 CODE=0 ID=2916 SEQ=1 
Dec  2 04:20:14 master kernel: TRACE: filter:cali-pri-net1:return:3 IN=ens33 OUT=cali282b0776936 MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=21102 DF PROTO=ICMP TYPE=8 CODE=0 ID=2916 SEQ=1 
Dec  2 04:20:14 master kernel: TRACE: filter:cali-tw-cali282b0776936:rule:6 IN=ens33 OUT=cali282b0776936 MAC=00:0c:29:2e:0f:7b:00:0c:29:5e:b2:88:08:00 SRC=172.16.91.226 DST=172.20.219.81 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=21102 DF PROTO=ICMP TYPE=8 CODE=0 ID=2916 SEQ=1 
```
我们重点看下filter表里的规则链  
![master节点上分析iptables规则链1](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/24E36B9D23B44BF9BB60598CA53813D0/22405)   
![master节点上分析iptables规则链2](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/F4E02AE72E174EFFA367B3858ECFEBC6/22407)  
![master节点上分析iptables规则链3](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/3BC11365C3234484B8D1E0115BBD9222/22409)
![master节点上分析iptables规则链4](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/3BC11365C3234484B8D1E0115BBD9222/22409)   
![master节点上分析iptables规则链5](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/B475117386EB4F248341A78D4A1FAC95/22411)   
![master节点上分析iptables规则链6](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/9D26E3F003794307B056ABD97F0B9F91/22416)  
![master节点上分析iptables规则链7](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/13A17AE6E60844A4874C3A09C368A386/22429)
![master节点上分析iptables规则链8](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/BDB504A7AA734052ACBBB543D5741A06/22420)   
![master节点上分析iptables规则链9](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/1F8E3B7C30804FF2919392E0E2228A9B/22422)   
`profile文件中source属性里的tag 是由ipset模块来控制的`  
可以通过删除tag，添加tag来观察不同之处   
![source属性](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/E5C38EA069914ACEBCD3AD707F191ACD/22426)  





## 4.5 场景五：同节点上，不同容器之间的访问   
1. 数据包流向图:  
![从容器到宿主机数据包流向图](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/46CAB9D2C8E0427E83FD32A027424B75/22431)    

2. docker exec -it web1 ping -c 1 172.20.219.84
![同节点上不同容器之间的ping](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/D25739F0589F4592AB2A3400916C7BB7/22435)  

3. tail -f iptables.log    
日志太多了，就不贴了。   

4. 将上面的日志，转换成下面的流向图    
![流向图说明](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/1AB1801084C44D80921DFBA87A159BCB/22433) 
5. 总结(从容器到宿主机数据包流向图)
- 从`web1容器到web2容器`走的是路线是:  
    - PREROUTING ---> FORWARD ---> POSTROUTING
- `web2容器 答复 web1容器`的 走的路线是:  
    - FORWARD   

## 4.6 场景六：跨节点，容器间互相访问    
1. 数据包流向图:  
![从web1容器到web3容器数据包流向图](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/A826D958B19D48C9B4A059DC603E05FA/22445)    

2. docker exec -it web1 ping -c 1 web3
![跨节点容器ping测试](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/57CB15FE15AE4DD9B46BB25D5F0C8CE9/22443)   

3. tail -f iptables.log  
日志太多，不上传了。  

4. 将上面的日志，转换成下面的流向图    
![master节点流向图说明](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/5999121E26134BC2A962D3DD42D7175B/22439)  
![slave1节点流向图说明](https://note.youdao.com/yws/public/resource/2c4c8762633914127de39b5d8680640f/xmlnote/2E61A0C4774B4A32B2528DD8B4B4AEFB/22441)    
5. 总结(从容器到宿主机数据包流向图)
- master节点上 从`web1容器到web3容器`走的是路线是: 
    - PREROUTING ---> FORWARD ---> POSTROUTING 
    - FORWARD    
- slave1节点上 走的路线是:  
    - PREROUTING ---> FORWARD ---> POSTROUTING 
    - FORWARD  



























