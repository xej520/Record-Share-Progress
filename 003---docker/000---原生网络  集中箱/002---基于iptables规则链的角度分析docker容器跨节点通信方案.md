# 背景介绍   
&ensp;&ensp;&ensp;&ensp;[上篇文章](https://www.jianshu.com/p/14f313520df7)分析了docker容器在不同网络下，网卡会有什么样的变化； 那么, 接下来我们从iptables规则链的角度，去分析跨节点docker容器是如何互相访问彼此的服务的, 如nginx服务。   
通过分析iptables规则链，希望对iptables规则链有进一步的了解。   
最主要的是为以后分析docker+calico环境下的iptables链打下基础。   

# 测试环境






# 测试   
## 测试准备  
### 部署docker  
1. 重新部署两台安装了docker的服务器  

2. 更新docker.service配置文件 
添加网段参数  
- 配置master节点   
![master节点docker服务](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/353F601C67DB4B2D8F61D6BFC81C797E/22072)
- 配置slave节点  
![slave节点docker服务](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/8E0B7857592A4A6B8A17B5DC53C457C7/22074)     


### 准备测试镜像  
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

### 准备测试容器    
1. 创建测试容器   
- 在master节点创建  
docker run -itd --name web1 -p 8080:80 mybusybox
![master 上创建容器](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/6797C58E91C943058B814F1B967FCA90/22081)    
- 在slave节点上创建   
docker run -itd --name web2 -p 8080:80 mybusybox
![slave 上创建容器](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/B24B7C9429AC427FBE78E504920AC124/22079) 

## 测试是否支持tcp 协议   
0. 注意，测试tcp时，并没有修改任何iptables规则链，直接用的就是默认的。   

1. 在master节点上，web1容器是否可以访问web2容器的80端口呢？   
![web1容器访问web2容器80端口](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/851735EFC0554CC7B224D8C9C5AF966D/22083)    

2. slave节点，web2容器访问master节点上web1容器呢？  
![web2访问web1的80端口](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/8F22379D86664CE89914FE00900E3E70/22085)  
3. 分析原因？  
![web1能够访问web2容器的80端口的原因](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/6EB9938E24AE440EA5F8C868A2DAC3BC/22087)        


# 分析iptables规则链    
分析iptables的规则链，希望可以得到在docker环境下，数据流的方向。   
下图显示的是没有docker的环境下：  
![iptables 数据流方向(无docker环境)](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/D768237BFA004C4A96288191CB27EDAD/22103)   








