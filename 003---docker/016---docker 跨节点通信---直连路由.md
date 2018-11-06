参考文献:   
https://blog.csdn.net/wangshuaiwsws95/article/details/80687384    

- 本方案，未依赖任何第三方插件。    
- 本方案，仅仅是测试研究分析，不能用于生产。 
- 必须是新部署的docker环境，  
    - 因为文章中有些命令使用了规则的序号，  
    - 如果不是最新部署的docker环境，序号可能不一样，导致iptables命令失败，报查询不到chain/name等。 

# 一、背景   
&ensp;&ensp;&ensp;&ensp;目前正在研究docker的网络，kubernetes网络的相关解决方案，如calico；   
而calico的底层也是调用iptables, ipset来实现的。 如何解决跨主机通信是值得研究的。   
分析完这个后，有助于理解calico是如何在docker、k8s环境中应用的。  

# 二、概述 

1. docker默认情况下：  
    单台主机上的docker容器间的通信可以通过docker0网桥直接通信   
2. 如何解决呢？  
    其中一方案是，通过映射端口来进行通信；   
但是，此方案，不能满足我们的使用场景。  
3. 希望达到的效果是，通过docker容器自己的IP直接进行通信

# 三、测试环境介绍  
1. 物理环境介绍  

    |主机名|IP|系统|  
    |:---|:---|:---|
    |a-master|172.16.91.131|centos7.5|   
    |a-slave1|172.16.91.132|centos7.5|   

2. docker版本    
![docker版本](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/156BAD0075DB4FCA9DA7D2A9736DB872/21912)   

# 四、情景构造(描述需求)        
&ensp;&ensp;&ensp;&ensp;有两个物理主机1和主机2，在各自宿主机上启动一个容器，启动成功之后，两个容器分别运行在两个宿主机之上，默认的IP地址分配如图所示，这也是Docker自身默认的网络
![容器间如何通信？](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/C7499B8B605246B58B2569DBF39EC142/21967)   
__此时两台主机上的Docker容器如何直接通过IP地址进行通信？__   
每个节点开启ipv4路由转发功能，使得流量可以在docker0跟物理机网卡之间进行通信。   
然后修改跟docker有关的iptables规则。 接下来尝试尝试.  
![通信过程](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/8EB791E51FD44CCB8939F970D5D2F7CC/21971)   


# 五、环境准备
## 5.1 部署docker  
1. 更新yum源  
    ```
    yum install -y yum-utils device-mapper-persistent-data lvm2  
    yum-config-manager --add-repo  https://download.docker.com/linux/centos/docker-ce.repo
    ```  

2. 配置私有仓库地址(添加自己的加速器)
   
    ```
    mkdir -p /etc/docker
    tee /etc/docker/daemon.json <<-'EOF'
    {
    "registry-mirrors": ["https://XXXXXX.mirror.aliyuncs.com"]
    }EOF
    ```

3. 配置master节点的docker子网段信息      
    vim /lib/systemd/system/docker.service 
    ![docker.service](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/41DE93FDBF5740F4BFB8F27DD56EDA7F/21920)   

4. 配置slave节点的docker子网段信息     
    vim /lib/systemd/system/docker.service 
    ![docker.service](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/229DADFD9FC54E578CD4B07F8977F670/21922)   

5. 重新启动docker服务(master，slave1节点)   
    ```
    systemctl enable docker
    systemctl daemon-reload
    systemctl restart docker
    ```

## 5.2 创建容器  

1. 在master上创建容器  
    ```
    docker run --name m-web1 -itd busybox
    ```  
    ![m-web1](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/65E4D2B982DF466CA42DDD48332BAC9C/21944)   

2. 在slave上创建容器   
    ```
    docker run --name s-web1 -itd busybox
    ```  
    ![s-web1](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/02862C8DACC54849A8E67B52D6555CF4/21946)   


## 5.3 具体步骤  
0. 查看a-master，a-slave节点上的ipv4路由是否开启   
    - 查看是否开启？  
        ```
        sysctl -a | grep net.ipv4.ip_forward 
        ```   
        net.ipv4.ip_forward=0,说明未开启  

    - 如何开启？  
        ```
        echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf   
        sysctl -p  
        ```

1. 清理nat表中的POSTROUTING链(master节点上操作)
    - 清理前测试  
    ![nat表中的POSTROUTING链-清理前测试](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/FC2411EC8DD04969ABAD8F97F433E127/21948)    
    - 清理POSTROUTING链(目的：防止进行NAT转换)    
    iptables -t nat -F POSTROUTING   
    ![具体清理nat表中的POSTROUTING链](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/40CD91663DF844B78E56CA2F904748FC/21951)    
    - 清理后测试  
    ![nat表中的POSTROUTING链-清理后测试](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/C9CAEBBEAE554FEA93F6562B17FEC81F/21953)      

2. 查看a-slave节点上filter表   
    ![filter表](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/A40A507AD0BF4080A33AB1DD84B1FE56/21942)    

3. 修改a-slave节点上filter表中FORWARD的默认策略  
    iptables -P FORWARD ACCEPT    
    iptables -t filter -R DOCKER-ISOLATION-STAGE-2 1 -j ACCEPT 
    ![iptables -P FORWARD ACCEPT](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/4E818DAC269D40158478AE38D3C1265B/21957)    


# 六、测试    
在master节点进行测试   
    ![master节点](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/1C5A4113945F4A058498F0576CAACAB3/21961)    
    ![slave节点](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/FE507128E6DF463693FBB9391E9BB02A/21963)   

注意，这里仅仅可以master节点上去ping  slave节点， 反之不行，    
如果想实现双向通信的话， 操作步骤跟上面类似。   


# 七、扩展测试   
## 7.1 测试net.ipv4.ip_forward参数  
![net.ipv4.ip_forward](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/A2C63E3BD0E24BB5B7C87A4BB093434F/21965)   
也就是说，docker0跟物理机网卡ens33进行通信是通过这个参数进行设置的。   

## 7.2 修改a-slave节点上filter表中DOCKER-ISOLATION-STAGE-2链中的跳转动作   
1. 修改成DROP？ 查看  
    ```
    iptables -t filter -R DOCKER-ISOLATION-STAGE-2 1 -j DROP  

    ```  
    ![master节点](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/78927478C167466D897C97D016B5ABFA/21973)   
    ![slave节点](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/B1DF85A2AB49439DA60BDF0F6F252737/21977)   

2. 修改成REJECT? 查看  

    ```
    iptables -t filter -R DOCKER-ISOLATION-STAGE-2 1 -j REJECT  

    ```   
    ![master节点](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/6AF0D6CF818143688431455BA4BDDEE5/21980)   
    ![slave节点](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/7E4159CB634F4D86AFE3EC2FE87BEF67/21982)   

3. 总结DROP跟REJECT的区别？  
    - DROP  
        - 网络可达， 
        - 返回的数据可以到达docker0， 然后就被iptables的DROP过滤掉了  
    - REJECT  
        - 网络不可达，  
        - 报protocol 1 port 34841 unreachable, length 92

# 八、iptables 知识点补充    
https://www.cnblogs.com/davidwang456/p/3540837.html  

0. 默认情况下，查询的是filter表， 如果要操作nat有关的链，必须显示的指明  

1. 如何设置链的默认策略，通过-P参数进行  
    ```
    iptables -P FORWARD ACCEPT
    ```  
2. 如何更新规则的跳转动作  
    - 先查看规则的序号 (如查看DOCKER-ISOLATION-STAGE-2链里的规则) 
    ```
    iptables -t filter --line-number -nL  DOCKER-ISOLATION-STAGE-2
    ```
    - 重新设置跳转动作  
    ```
    iptables -t filter -R DOCKER-ISOLATION-STAGE-2 1 -j DROP 
    ```

3. 如何删除某条规则  
    - 先查看规则的序号 (如查看DOCKER-ISOLATION-STAGE-2链里的规则) 
    ```
    iptables -t filter --line-number -nL  DOCKER-ISOLATION-STAGE-2
    ```
    - 删除规则  
    ```
    iptables -t filter -D DOCKER-ISOLATION-STAGE-2 1
    ```

# 九、问题排查思路  
![数据包流向图](https://note.youdao.com/yws/public/resource/c6ad33a2c888dd260a639b03d33de2e5/xmlnote/E26A083E14E64D7BBB19D076EC7F498B/21985)  
1. 根据数据包走向图，进行排查，查询是哪里不通  
2. 使用tcpdump抓包分析  
3. ipv4是否开启  
4. iptables链是否正确      







