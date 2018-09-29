# <center>calico配置 BGP Peers</center> 
参考文献: 
https://docs.projectcalico.org/v2.6/usage/configuration/bgp 
https://blog.csdn.net/ptmozhu/article/details/71099509 

## 注意：  
该文档使用的calico版本号是v2.6

本文档主要描述使用 calicoctl 来管理 BGP. 主要针对在私有云上部署calico的用户，使calico跟底层基础架构进行peer。   
包含以下4个方面: 
- 全局默认 AS 号
- node-to-node 全互联(mesh模式)
- 全局 BGP Peers
- 指定节点BGP Peers
****
# 一、 基本概念 
1. 全局默认 AS 号  
就是当calico节点没有明确指定AS号时，  
即是你所有的calico node节点使用**同一个AS号**。 

2. node-to-node 全互联  
就是所有 Calico node 自动跟网络中其他calico node节点进行BGP peer。  
**默认**  
&ensp;&ensp;&ensp;&ensp;开启node-to-node全互联,   
**适用场景**  
&ensp;&ensp;&ensp;&ensp;node-to-node 全互联适应用小规模部署以及calico node节点在同一个2层网络。  
**推荐做法:**  
&ensp;&ensp;&ensp;&ensp;关闭node-to-node 全互联，指定节点间的BGP Peer。

3. 指定BGP peers   
可配置成全局的或者部分节点间.  
    - 一个全局 BGP peer   
        就是一个跟网络中所有节点peer 的BGP agent. 
    - 典型用例  
        就是当中规模部署的时候，所有在同一个二层网络的Calico
 node节点都跟同一个Route Reflector (或一组 Route Reflector)配对，此处的Route Reflector就是一个全局BGP peer。

大规模部署的时候, 有多种网络拓扑可选。  
例如在AS per Rack model 里, 每个 Calico node节点 跟在ToR交换机上的Route Reflector配对。也就是说机架上的每个Calico node节点跟ToR Route Reflector配置成指定node的BGP peers模式。
#  二、 配置默认node节点 AS 号  
- 当创建calico node的时候，可以指定 AS 号。  

- 如果没有指定，使用默认的全局AS号。
    - 可使用 calicoctl config set asNumber  命令来配置全局的AS号. 
- 若没有指定，默认的 AS 号是64512（2.0.0版本以前默认是64511）。  
- 如果你所有的calico node节点都是同一个AS号，当你需要使用一个不同的 AS number(例如，当跟一个边界路由器进行配对时，以及中规模，大规模部署的时候需要自定义网络拓扑的时候)，这时候就需要修改AS  number  

## Example
1. 查看当前默认的AS number  
    ```
    #calicoctl config get asNumber   
    64512
    ```
2. 配置默认 AS number 为 64513(随便选择一个节点测试)  
    ```
    #calicoctl config set asNumber 64513  
    #calicoctl config get asNumber
    ```
