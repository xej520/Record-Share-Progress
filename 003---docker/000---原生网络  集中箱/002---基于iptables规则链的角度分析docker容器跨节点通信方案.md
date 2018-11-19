# 一、背景介绍   
&ensp;&ensp;&ensp;&ensp;[上篇文章](https://www.jianshu.com/p/14f313520df7)分析了docker容器在不同网络下，网卡会有什么样的变化； 那么, 接下来我们从iptables规则链的角度，去分析跨节点docker容器是如何互相访问彼此的服务的, 如nginx服务。   
通过分析iptables规则链，希望对iptables规则链有进一步的了解。   
最主要的是为以后分析docker+calico环境下的iptables链打下基础。   

# 二、说明  
&ensp;&ensp;&ensp;&ensp;本文章的结论和说明仅仅是我个人的研究分析总结整理。

# 三、测试环境
|**系统类型**|**IP**|**role**|**cpu**|**memory**|**hostname**|
|:---|:---|:---|:---|:---|:---|
|CentOS 7.4.1708|172.16.91.110|master|4|2G|master|
|CentOS 7.4.1708|172.16.91.230|worker|2|1G|slave|

# 四、环境准备 
## 4.1 部署docker    
1. 部署两台安装了docker的服务器  
    具体部署，可以参考网上的其他参考文章   
2. 更新docker.service配置文件 
添加网段参数  
- 配置master节点   
![master节点docker服务](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/353F601C67DB4B2D8F61D6BFC81C797E/22072)
- 配置slave节点  
![slave节点docker服务](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/8E0B7857592A4A6B8A17B5DC53C457C7/22074)     


## 4.2 准备测试镜像  
1. 准备测试镜像(master,node节点操作)    
    vim dockerfile   
    ```
    FROM nginx
    RUN apt-get update 
    RUN apt-get install -y iputils-ping iproute2 wget
    ```
2. 构建镜像(master,node节点操作)  
    ```
    docker build -t mybusybox .
    ```

## 4.3 创建测试容器   
0. 此处创建容器，采用的是docker原生的bridge模式    
docker原生的网络模式之间的不同，可以简单的参考[这篇博文](https://www.jianshu.com/p/14f313520df7)

1. 在master节点创建  
docker run -itd --name web1 -p 8080:80 mybusybox      
2. 在slave节点上创建   
docker run -itd --name web2 -p 8080:80 mybusybox   

# 五、测试  
## 5.1 创建容器后，查看iptables链  
1. nat表   
![master nat表](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/06A9B14488254B02BF3714BAA232EA29/22107)   

2. filter表   
![master filter表](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/6E4040FDC3824FB18038A6CA98123552/22109)   

3. 测试icmp协议是否支持   
![测试是否支持icmp协议](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/8A1299DEAE8844EB8BEA4B2EDB197F0A/22111)   

4. 测试是否支持tcp协议？  
![测试是否支持tcp协议](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/5366CA72690040B192744D565D8B3701/22113)   

## 5.2 从直连角度测试(删除掉NAT转换)   
主要思路：  
- 修改nat表，清除POSTROUTING链；
- 更新filter表；  
实现ICMP，TCP协议通信
1. 清理nat表的PREROUTINGl链(master,slave节点)  
iptables -t nat -F PREROUTING
![master节点上操作](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/B8A335E169194B9CABF1483FDFBF65ED/22116)   
![slave节点上操作](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/8F5983D3ED81478EBFAC76DCCB9FD3F0/22118)   

2. 更新filter表(master,slave节点)  
- 在master节点上操作     
iptables -t filter -A -d 172.20.1.2/32 -i ens33 -o docker0 -p icmp -j ACCEPT   
![master节点上操作filter表添加docker链](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/59D84C5466BE4C109802C54224818F38/22120)   
- 在slave节点上操作   
iptables -t filter -A -d 172.20.2.2/32 -i ens33 -o docker0 -p icmp -j ACCEPT
![slave节点上操作filter表添加docker链](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/7D3FF7438FEA4AC685E0B3722CBF00F0/22122)   

3. 进行测试  
- 在master节点上  
    - tcp协议测试  
    ![master节点 tcp协议](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/9484017E15534EC8BB16C59A2B635AD9/22132)   
    - icmp协议测试  
    ![master节点 icmp协议](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/03CF0F44406B4E9CA1D9A24F6DBD5F58/22134)   
- 在slave节点上   
    - tcp协议测试  
    ![slave节点 tcp协议](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/DB57183563324ACC89F7BB01DCA8E3F4/22138)   
    - icmp协议测试  
    ![slave节点 icmp协议](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/19087746544342A5B525A076638DBA91/22136)   

4. 注意
    在测试时，很怪异，有的时候不需要做任何修改就可以实现跨节点访问nginx服务，但有的时候又不行。 具体原因未找到。     


# 六、分析总结docker环境下iptables规则链    
分析iptables的规则链，希望可以得到在docker环境下，数据流的方向。   

## 6.1 下图显示的是没有docker的环境下：  
http://www.zsythink.net/archives/1199/   
![iptables 数据流方向(无docker环境)](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/D768237BFA004C4A96288191CB27EDAD/22103)   
从图中可以看出来，数据包是由方向的。  

## 6.2 iptables规则链分析   
1. 关于docker的iptables规则，并没有在原四表五链的基础上创建，而是专门针对docker新创建了几条链   
好处就是方便管理规划    
2. 以表作为入口来查询链，如：      
iptables -t nat -S   
iptables -t nat -nL   
3. iptables -t nat -S 跟 iptables -t nat -nL 有什么区别？   
最主要的不同就是 -S  参数，可以显示出数据包是从哪个网卡流入，流出的。  推荐使用-S参数来查询   
4. 常用到的表是nat表，filter表   
    - nat 是专门用来进行nat转换的，  不具有过滤功能，   
        - 因此，不能将此表中的链里的跳转规则设置成DROP，REJECT,RETURN等   
    - filter,专门用来进行过滤的  
5.  测试时，发现一现象：更新iptables规则后，可能不会立即生效。[原因未知]   

6. 容器重启后，某些iptables规则链会重置；如：POSTROUTING，DOCKER链   

7. 关于tcp协议测试，现象比较怪异：  
    - 某一次测试，什么都没有改动，可以从web1容器里访问web2容器里的nginx服务
    - 有的时候，又不行。原因未知    

8. 如何去分析数据包的走向呢？  
    docker容器主要用到了FORWARD链。   
    - 如何测试docker容器到底用到了哪些呢？  
    我用的最笨的方法，排除法，删除某些链再测试   
    - 分析FORWARD链   
        - FORWARD链有两个数据流方向，
            - 数据包进入本机，
            - 数据包出本机   
    - 场景一： 假设数据包来自master节点 上的web1容器中，打算发送到 slave节点的web2容器里，数据包的流向是？     
    web1数据包--->web1容器在master节点上的虚拟网卡veth9aa3f8f--->docker0网桥--->查询路由表进行路由判断--->iptables中的FORWARD链--->POSTROUTING链--->ens33网卡====》路由器===》到达slave节点--->ens33网卡--->PREROUTING链--->FORWARD链--->docker0--->web2容器的虚拟网卡vethcb0c9a7--->web2容器   
    这里主要分析一下FORWARD链(其他链功能单一，而且数据流方向都是单向)。   
        - 先分析master节点上的FORWARD链   
        - 先进入-A FORWARD -j DOCKER-USER链   
            - 根据跳转规则，进入DOCKER-USER链  
        - 进入DOCKER-USER链  -A DOCKER-USER -j RETURN 
            - 跳转规则是return  
            - 也就是，跳转到主规则上，继续执行下面的规则   
        - 进入-A FORWARD -j DOCKER-ISOLATION-STAGE-1 链   
            - 跳转到DOCKER-ISOLATION-STAGE-1链  
            ![规则的执行顺序:从上往下执行](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/27DE1467BD3545CF8F6C03E295C15C41/22142)      
        - -A DOCKER-ISOLATION-STAGE-1 -i docker0 ! -o docker0 -j DOCKER-ISOLATION-STAGE-2  
            - web1的数据包是从docker0进入，转发到非docker0网卡(也就是ens33物理网卡出去)   
            - 跳转到DOCKER-ISOLATION-STAGE-2链   
        - -A DOCKER-ISOLATION-STAGE-2 -o docker0 -j DROP  
            - 数据包并非从docker0流出，而是ens33流出，因此不满足此规则的条件，继续执行下面的规则  
        - 进入 -A DOCKER-ISOLATION-STAGE-2 -j RETURN 
            - 跳转规则是RETURN   
            - 返回主链，继续执行下面的规则   
        - -A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT   
            - 数据包并非从docker0流出，而是ens33流出，因此不满足此规则的条件，继续执行下面的规则  
        - -A FORWARD -o docker0 -j DOCKER                             
            - 数据包并非从docker0流出，而是ens33流出，因此不满足此规则的条件，继续执行下面的规则
        - -A FORWARD -i docker0 ! -o docker0 -j ACCEPT  
            - 数据包从docker0流入，从非docker0流出(ens33)，跳转规则是ACCEPT，  
            - 也就是允许数据包流出
        到此，master节点上FORWARD链，分析完成
        - 进入slave节点上，分析FORWARD链    
        - 先进入-A FORWARD -j DOCKER-USER链   
            - 根据跳转规则，进入DOCKER-USER链  
        - 进入DOCKER-USER链  -A DOCKER-USER -j RETURN 
            - 跳转规则是return  
            - 也就是，跳转到主规则上，继续执行下面的规则   
        - 进入-A FORWARD -j DOCKER-ISOLATION-STAGE-1 链   
            - 跳转到DOCKER-ISOLATION-STAGE-1链  
        - -A DOCKER-ISOLATION-STAGE-1 -i docker0 ! -o docker0 -j DOCKER-ISOLATION-STAGE-2 
            - 数据包是由ens33流入，转入docker0,因此，不满足此规则，继续执行下一条规则   
        - -A DOCKER-ISOLATION-STAGE-1 -j RETURN  
            - 只要数据包匹配，就进行return  
            - 回到主主链继续匹配   
        - -A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT  
            - 这条FORWARD规则，设置数据包的流入不管是哪个设备，只要输出的设备是docker0，
            - 数据包的状态是RELATED,ESTABLISHED   
            - 就进行放行。  
            - 如果是第一个数据包的话，不会走这条规则，因为第一个数据包的状态是NEW   
        - -A FORWARD -o docker0 -j DOCKER  
            - 跳转到DOCKER链  
        - -A DOCKER -d 172.20.1.2/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 80 -j ACCEPT   
            - 数据包的目的地址是172.20.1.2/32， 
            - 数据包的流入不是docker0   
            - 数据包转发到docker0               
            - 如果数据包的协议是TCP协议，也就是访问nginx服务，  
            - 端口是80  
            - 当前数据包满足条件，就进行ACCEPT放行 。  

        - -A DOCKER -d 172.20.1.2/32 -i ens33 -o docker0 -p icmp -j ACCEPT   
            - 数据包的目的地址是172.20.1.2/32， 
            - 数据包通过ens33流入   
            - 数据包转发到docker0               
            - 如果协议是ICMP，也就是ping命令测试时  
            - 当前数据包满足条件，就进行ACCEPT放行 。 

9. 默认的链都有默认策略， 先执行链自己的规则，如果都没有匹配到的话，就执行默认策略。如   
    - FORWARD链的默认策略是DROP  
    - 如果数据包都没有匹配到规则的话，就会执行该FORWARD的默认策略DROP，将此数据包丢弃    
    ![FORWARD 默认策略](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/67242E1AE9424D3CAE16B04ACBFA2FB4/22140)   

10. 关于类似于-A DOCKER-ISOLATION-STAGE-1 -i docker0 ! -o docker0 -j DOCKER-ISOLATION-STAGE-2规则的说明  
    - 只有FORWARD链，可以同时存在-i,-o参数   
    - 实现的目的是，网卡之间的数据转换，因此需要开启ipv4的路由转换功能  
        - sysctl -a | grep net.ipv4.ip_forward 为1的话，说明开启了。   
    - -i docker0 ! -o docker0是说，数据包流入docker0，然后转发到非docker0的设备上  
    ![FORWARD链中-i -o参数说明](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/24FB304152F943CE91DC7D920439C37C/22145)     

11. 删除nat表下某个链里的某个规则   
    iptables -t nat -D POSTROUTING 2  
    2是该条规则的序号  
12. 列出规则的序号   
    iptables -t nat --line-number    
    ![列出规则序号](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/4CD4D486221644029B0F0900F045CDEC/22147)   
13. 修改某一条规则   
    - 参数是-R 
    - iptables -t filter -R DOCKER-ISOLATION-STAGE-2 1 -o docker0 -j ACCEPT     
    ![修改规则](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/0233F1F0A3164098A73589EBAAFA068C/22149)       

14. 修改默认链的策略  
    - 参数是-P   
    - iptables -t filter FORWARD ACCEPT   

15. -A FORWARD -i docker0 -o docker0 -j ACCEPT  
    - 这条规则，针对的是，同一节点上，docker0网桥内，容器之间的通信
    - 假设想实现，同一个网络内，容器之间隔离的效果的话，可以这样设置   
        - iptables -t filter -R FORWARD 6 -i docker0 -o docker0 -j DROP   


16. 常见以太网协议类型字段  
![以太网协议类型字段](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/68456DFF6A35494DBEEB41718E1CB424/22191)   

17. ebtables中的broute表功能：？   
用于控制进来的数据包是需要进行bridge转发还是进行route转发，即2层转发和3层转发。   
18. 如果一个以太网接口eth1，它并没有桥接到br-lan0中，此时，从eth1进来的数据包不会走到ebtables中   

它会在下图中的bridge check点，检查数据包进入的接口是否属于某个桥，如果是则走ebtables，否则直接走iptables。

19. 如何查看iptables的日志呢？  


# Calico 
1. calico环境下，docker容器 是作为单点的局域网？   
![calico环境下容器作为单点的局域网](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/E103B63122E94518A9BE638117E82195/22193)   

2. 获取calico容器中cali0的对端号？  
进入docker容器里执行哦
ethtool -S cali0   
![ethtool获取对端的网络号](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/CA1FCDEDE0DF416BA64DBC90B1DF44FC/22197)  

3. 如何安装ethtool?  
apt-get update & ap-get install -y ethtool   



