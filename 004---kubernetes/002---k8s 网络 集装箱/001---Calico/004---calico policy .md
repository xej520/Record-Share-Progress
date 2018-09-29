# <center>calico policy 测试  </center>
# calico policy 测试  
默认情况下，不同calico网络的容器是隔离的，不能互相通信的； 
默认policy的策略配置如下:  



## 一、测试允许cal_net1网络下的容器，可以接收其他calico网络容器的数据包  
### A、修改步骤： 
1. 获取cal_net1的policy 配置文件  
         - calicoctl get policy cal_net1 -oyaml  > cal_net1_policy.yaml  
2. 更新cal_net1_policy.yaml中ingress属性里的source属性，如下:  
    ```
     -  apiVersion: v1
        kind: profile
        metadata:
            name: cal_net1
            tags:
            - cal_net1
        spec:
            egress:
            - action: allow
            destination: {}
            source: {}
            ingress:
            - action: allow
            destination: {}
            source: {}
    ``` 
3. 重新启动policy机制  
calicoctl apply -f cal_net1_policy.yaml (不需要重启calico服务)
### B、 效果测试  
1. 从其他calico网络的容器中，ping cal_net1网络中的容器 
    ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/B70F5F3A4FC74D688216C7A76E4524AF/20210)  




2. 测试cal_net1的80端口，对cal_net2网络中的容器 开放  
   - 获取cal_net1的policy 配置文件  
         - calicoctl get policy cal_net1 -oyaml  > cal_net1_policy.yaml   
   - 更新cal_net1_policy.yaml中ingress属性里的destination属性，并添加TCP协议，不然会报错的，如下：  
    ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/C4CF57EC61024EB0A06DD6836F695854/20216)  
    - calicoctl apply -f cal_net1_policy.yaml  
    - docker run --network=cal_net1 --name web1 -d httpd  
    - docker exec -it web1 ip addr (查看一下ip地址)  
    - docker exec -it bbox2 wget 172.20.219.85
        ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/840FC81B125F43E6BC3837D36FD5AC4C/20213)  
    - 注意，这里仅仅是开放了tcp协议，因此cal_net1网络中的容器是不接收ping命令的，如下： 
       ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/203C455BEA23408895D23F435BFF87F1/20219)   
3. 测试policy策略，允许接收tcp，icmp协议  
    ```
          - apiVersion: v1
            kind: profile
            metadata:
                name: cal_net1
                tags:
                - cal_net1
            spec:
                egress:
                - action: allow
                destination: {}
                source: {}
                ingress:
                - action: allow
                protocol: tcp
                destination: 
                    ports: 
                    - 80
                source: 
                    tag: cal_net2
                - action: allow
                protocol: icmp 
                source:
                    tag: cal_net2
    ```