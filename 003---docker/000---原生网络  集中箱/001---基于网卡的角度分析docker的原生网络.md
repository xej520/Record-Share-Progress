本文主要研究分析docker容器使用不同网络模式时，  
网卡会有什么变化    
做到心里有底       

# 一、总结   

|docker容器<br>网络模式|iptables规则跟<br>初始化环境对比|是否创建<br>新的虚拟网卡|是否有<br>独立网络|  
|:---|:---|:---|:---|
|bridge|没有变化|是|有|
|host|没有变化|没有创建|跟物理机共用一个|
|none|没有变化|没有创建|只有lo|   

备注；     
创建的是最简单的容器，没有端口映射等。如   
```
docker run --name b-web1 -itd busybox   
docker run --net=host h-web1 -itd busybox   
docker run --net=none n-web1 -itd busybox    
```  
因此，创建前后iptables规则没有变化。  

主要关心的是网卡的变化     

# 二、三种网络模式最大的区别是   
可以从以下两个角度来分析  
- 物理机是否产生了新的虚拟网卡，  

- 容器内部是否有新的虚拟网卡产生  

## 2.1 一句话总结：   
    只有bridge模式，产生了新的虚拟网卡，其他两种模式都没有产生   
    host模式，跟物理机共用一个网络  
    none模式，只有lo设备   

我们主要关心nat, &ensp;filter表的变化  
# 三、iptables规则  
1. nat表  
![nat表](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/C0C1E1852A70468EB5ED8F3BB810DA33/21988)   
2. filter表  
![filter表](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/4C268FD139D94F8585AE73647DF41F64/21990)   


# 四、网卡分析   
## 4.1 bridge模式下，网卡变化  
![物理机网卡状态](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/A8C3B97A6E7F4DC89AF9B2A97685A3D0/21993)   
![docker容器内网卡状态](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/6541818B46BD45F6AFDA5810635E003F/21995)   

## 4.2 host模式下，网卡变化  
![物理机网卡信息](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/0F750C42F2B142C499B796F60DF01414/21997)      
![docker容器内部网卡信息](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/97F7DAAB21A84FAB9C503234F644121C/21999)   

## 4.3 none模式下，网卡变化    
![物理机网卡信息](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/1D91D14CBD914BE18DB2534743A294E6/22001)   
![docker容器内部网卡信息](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/9BAB2B3015BF42FE98159227A9001FD0/22003)   

