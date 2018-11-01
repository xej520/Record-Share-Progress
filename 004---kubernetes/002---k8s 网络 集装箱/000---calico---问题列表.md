
1. Calico node 'calico01' is already using the IPv4 address 172.16.91.205:  

    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/2092478BC26C456380653D09630AEA25/20874)  
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/563E5488282C4F1EA0951E9EDEFC9BAC/20876)  
    - 方法一：  
        如果calico数据不重要的话，可以根据图片中的操作步骤进行删除  
        etcdctl rm /calico --recusive  
    - 方法二：  
        根据原因是因为：  
        多个calico服务，绑定在同一个IP上导致的，  
        因此，需要检查calico服务的配置文件，看看是不是ip绑定重复了。  

    calicoctl rm /calico --recursive  
2. calicoctl node status no IPv4 peers found
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/C60E939916E14903A5B7674B7AB4A0BB/20878)  
    原因： 
    - 其他节点calico服务没有启动 
    - ip绑定错误，如下所示：  
        ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/7C1BE91F4151482480DB58D4FEB815AC/20882)  

3. Error response from daemon: Post http://%2Frun%2Fdocker%2Fplugins%2Fcalico-ipam.sock/IpamDriver.RequestPool: dial unix /run/docker/plugins/calico-ipam.sock: connect: connection refused  
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/B38B23E3EB314BFFA8B231EA343CB854/20885)  

4. docker calico 环境，创建容器的容器不能ping通主机  
现象一：  
没有创建路由， 如下图所示：  
![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/313DF703D5124BE79BAF008F1E0E584A/20887)  
主机和容器互相不能ping通。  
也不报错。  
原因是：配置文件
![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/02A1B7F8BA674844A5FCE148F55A5EF3/20889)  
也不报错。  

5. docker network create --driver calico --ipam-driver calico-ipam net2 创建网络后，其他节点上没有同步网络信息？ 
    ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/63A9834E52F6450CB876ECB046FF2D34/21642)   
    ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/59E739D08AAD42E69E28C7387499B10F/21644)  

    原因之一：   
    &ensp;&ensp;&ensp;&ensp;其中两个节点上的docker，并没有从etcd里获取数据，因此并没有进行同步。  
    &ensp;&ensp;&ensp;&ensp;__解决措施：__  
        &ensp;&ensp;&ensp;&ensp;添加----cluster-store参数即可。 

    原因之二：   
    &ensp;&ensp;&ensp;&ensp;calico的libnetwork进程出问题  
    ![libenetwork进程](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/797232FD3C8841C7BA8D46C46C72AA60/21646)  
    原因之三：   
    &ensp;&ensp;&ensp;&ensp;etcd出问题  
 
6. ERROR: Unable to access datastore to query node configuration  
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/DA7A57DE369B495BBA642348BCE3F410/21714)   

7. Idle BGP Error: Bad BGP identifier  
现象如下图所示：  
![master节点](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/30E758362B9A45579AB639F02D81BEA0/21720)  
![node1节点](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/F0ADA6572C1643349EEB743AF22DDBBA/21721)  
![node2节点](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/4B061D3078564A998C3B8B66250335A7/21723)   
但是，这种情况，服务好像还可以使用  






