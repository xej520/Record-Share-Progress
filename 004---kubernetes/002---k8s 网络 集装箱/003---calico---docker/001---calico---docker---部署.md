calico docker
# 一、环境介绍   
1. 物理环境介绍  

    |主机名称|系统版本|IP|  
    |:---|:---|:---| 
    |master|centos7.5|172.16.91.205|  
    |slave1|centos7.5|172.16.91.206|  
    |slave2|centos7.5|172.16.91.207|  

2. 安装服务部署介绍  

    |服务名称|版本|
    |:---|:---|
    |docker|v17.03.2-ce|
    |etcd|v3.2.22|
    |calico|v2.6|

# 二、前提准备 
## 2.1 etcd的部署  
&ensp;&ensp;&ensp;&ensp;可以参考此文章  
&ensp;&ensp;&ensp;&ensp;https://www.jianshu.com/p/5ea027315285    

## 2.2 docker的部署(所有节点操作)  
1. 部署docker  
    请参考其他博客  
2. 链接etcd集群     
    vim /lib/systemd/system/docker.service  
    ![链接etcd](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/A7D5C71F78E94039816E4E7AE0B8D2AF/20688)  

3. 重新启动docker服务 
    ```
    systemctl daemon-reload  
    systemctl restart docker
    ```  
    ![重启docker服务](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/BEB3FBE07B3F4EE7AEEF39AD66B8BF80/20686)

# 三、部署calico(所有节点)
参考文献：  
https://docs.projectcalico.org/v2.6/getting-started/docker/installation/manual  
## 3.1 镜像准备  
|镜像名称|下载地址|
|:---|:---|
|quay.io/calico/node:v2.6.11|registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/calico-node:v2.6.11|  
- 方案一： 自己下载 
    - docker pull quay.io/calico/node:v2.6.11  
- 方案二： 使用下面的命令(阿里云)  
    - docker pull registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/calico-node:v2.6.11    

## 3.2 安装calicoctl工具  
0. 创建目录 
    ```
    mkdir -p /usr/local/bin
    ```
1. 下载calicoctl工具  
    官网地址:  
    https://docs.projectcalico.org/v2.6/getting-started/docker/installation/manual  
    ```
    wget -O /usr/local/bin/calicoctl https://github.com/projectcalico/calicoctl/releases/download/v1.6.4/calicoctl
    chmod +x /usr/local/bin/calicoctl
    ```
2. 配置calicoctl的配置文件   
    mkdir -p /etc/calico  
    官网地址：  
    https://docs.projectcalico.org/v2.6/reference/calicoctl/setup/etcdv2  
    vim /etc/calico/calicoctl.cfg  
    ```
    apiVersion: v1
    kind: calicoApiConfig
    metadata:
    spec:
      datastoreType: "etcdv2"
      etcdEndpoints: "http://172.16.91.205:2379,http://172.16.91.206:2379,http://172.16.91.207:2379"
    ```  
    ![主要操作步骤](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/EDED26DE296F49C18FB387A624F05BD6/20690)  


## 3.3 安装calico(所有节点操作)  
官网如下：  
https://docs.projectcalico.org/v2.6/usage/configuration/as-service  
生命周期是由linux系统的systemd来进行管理，以容器的形式运行  

1. 创建calico的环境变量  
    vim /etc/calico/calico.env
    ```
    ETCD_ENDPOINTS="http://172.16.91.205:2379,http://172.16.91.206:2379,http://172.16.91.207:2379"
    ETCD_CA_FILE=""
    ETCD_CERT_FILE=""
    ETCD_KEY_FILE=""
    CALICO_NODENAME="calico01"
    CALICO_NO_DEFAULT_POOLS=""
    CALICO_IP=""
    CALICO_IP6=""
    CALICO_AS=""
    CALICO_LIBNETWORK_ENABLED=true
    CALICO_NETWORKING_BACKEND=bird
    ```
    注意：  
    - CALICO_NODENAME 如果不设置的话，默认为主机名，要保证主机名唯一  
    - CALICO_NETWORKING_BACKEND：有效值如下：  
        - bird(默认值) 
        - gobgp(无ipip模式)
        - node 

2. 创建calico-node.service配置文件  
    vim /lib/systemd/system/calico-node.service
    ``` 
    [Unit]
    Description=calico-node
    After=docker.service
    Requires=docker.service

    [Service]
    EnvironmentFile=-/etc/calico/calico.env
    ExecStartPre=-/usr/bin/docker rm -f calico-node
    ExecStart=/usr/bin/docker run --net=host --privileged \
    --name=calico-node \
    -e NODENAME=${CALICO_NODENAME} \
    -e IP=${CALICO_IP} \
    -e IP6=${CALICO_IP6} \
    -e CALICO_NETWORKING_BACKEND=${CALICO_NETWORKING_BACKEND} \
    -e AS=${CALICO_AS} \
    -e NO_DEFAULT_POOLS=${CALICO_NO_DEFAULT_POOLS} \
    -e CALICO_LIBNETWORK_ENABLED=${CALICO_LIBNETWORK_ENABLED} \
    -e ETCD_ENDPOINTS=${ETCD_ENDPOINTS} \
    -e ETCD_CA_CERT_FILE=${ETCD_CA_CERT_FILE} \
    -e ETCD_CERT_FILE=${ETCD_CERT_FILE} \
    -e ETCD_KEY_FILE=${ETCD_KEY_FILE} \
    -v /var/log/calico:/var/log/calico \
    -v /run/docker/plugins:/run/docker/plugins \
    -v /lib/modules:/lib/modules \
    -v /var/run/calico:/var/run/calico \
    -v /var/run/docker.sock:/var/run/docker.sock \
    quay.io/calico/node:v2.6.11

    ExecStop=-/usr/bin/docker stop calico-node

    Restart=on-failure
    StartLimitBurst=3
    StartLimitInterval=60s

    [Install]
    WantedBy=multi-user.target
    ```
    注意，  
    - 更新镜像的地址， 更新成自己私有仓库的地方，这样比较方便，不然每个节点上还得从官网拉取  
    我的集群环境里，每个节点上，已经下载好了镜像quay.io/calico/node:v2.6.11  
    - 经测试：calico集群中至少要有一个节点要挂载 -v /var/run/docker.sock:/var/run/docker.sock   
    当然，为了提高可靠性，最好每个节点都要挂载，如果不挂的话，无法使用calico网络创建docker容器； 
        - 抛出的error日志如下:  https://www.jianshu.com/p/eaab33c49362  
        - libnetwork-plugin插件的官网明确要求了，必须要添加  
        https://github.com/projectcalico/libnetwork-plugin  
        ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/42226CC108614988B6F07A9D08FF2C2E/20762)    


3. 启动服务
    ```
    systemctl enable calico-node
    systemctl start calico-node
    systemctl status calico-node
    ```

# 四、测试calico服务  
1. 查看节点运行情况  
    ```
    [root@master ~]# calicoctl node status
    Calico process is running.

    IPv4 BGP status
    +---------------+-------------------+-------+----------+-------------+
    | PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
    +---------------+-------------------+-------+----------+-------------+
    | 172.16.91.206 | node-to-node mesh | up    | 07:49:38 | Established |
    | 172.16.91.207 | node-to-node mesh | up    | 07:49:37 | Established |
    +---------------+-------------------+-------+----------+-------------+

    IPv6 BGP status
    No IPv6 peers found.

    [root@master ~]# 

    ```  
2. 查看端口BGP 协议是通过TCP 连接来建立邻居的，因此可以用netstat 命令验证 BGP Peer  
     netstat -natp|grep ESTABLISHED|grep 179  
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/79696D9BF4094B64BB49190D65E1CFCA/20711)  


# 五、问题列表  
1. Calico node 'master' is already using the IPv4 address 172.16.91.205 
    ![事故现场](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/3B3006BA5CDF4624A45548D135BFA801/20692) ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/50415B0DA5FE40599D520AF8A83A2760/20694) 
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/814A8188495440B29CCE70D5182314FB/20697)  
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/0DBCD2C9B9864E498754072B88770126/20699)  
    删除了文件夹里的内容，再通过下面的命令，删除目录   
    ```
    [root@master ~]# etcdctl rmdir /calico/bgp/v1/host/master
    ```  

    &ensp;&ensp;&ensp;&ensp;如果etcd 使用的版本是v3的话，可以使用下面的形式 
    - 查看master的信息  
    etcdctl get / --prefix --keys-only | grep master 
    - 删除master信息  
    etcdctl get / --prefix --keys-only | grep localhost | xargs -I {} etcdctl del {}  
    - 再重启calico-node服务  

2. calicoctl node status Active Socket: Connection refused  INFO: BIRDv4 process: 'bird' is not running.
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/8A39CD586C014EF186E0043FFBFCD9D4/20701)  
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/69023E491DD84BCAA7A3D495B8597D17/20743)  
    当其他节点的bird进程挂掉后，就会出现Active Socket: Connection refused  
    如何解决呢？ 
    清理了些历史数据，然后再查看时，就好了。具体原因不知道为啥。   
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/90D0E866DBCB4E98ADA2DEAA3E9A1905/20706) 
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/58BBB1CEBA8F4C748BC8913D5A5935E0/20709)  

# 六、 说明  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/47047FBEA0B549278F746AEA4649D8C8/20716)  


# 七、注意：  
参考文献：  
http://www.mamicode.com/info-detail-2333592.html  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/FEFC12880A644EA0A8D9CF545A4FE287/20722)  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/8AC72DD3A02C4AA0BFDB417FF6C1B691/20724)   

# 八、特别说明：  
按照上面的步骤，创建后calico集群后， 
创建calico网络，然后再创建容器， 可能会有问题：
- 没有针对容器创建路由  
也就是主机不能ping通容器，容器也不能ping通主机。  
具体原因，始终没有查出来，可能是版本问题。   
这里有讨论： 
https://github.com/projectcalico/calicoctl/issues/1845  
目前，也没有给出具体的方案，大家的措施基本都是换docker的版本  
但是，docker 18.06.0-ce 他们没有成功，而我的集群就没有问题。 
可能跟docker的版本也没有关系。  

但是，下面的版本，我尝试了，可以成功：  
1. 物理环境介绍  

    |主机名称|系统版本|IP|  
    |:---|:---|:---| 
    |master|centos7.5|172.16.91.205|  
    |slave1|centos7.5|172.16.91.206|  
    |slave2|centos7.5|172.16.91.207|  

2. 安装服务部署介绍  

    |服务名称|版本|
    |:---|:---|
    |docker|18.06.0-ce|
    |etcd|v3.2.22|
    |calico|v2.6.2|   
3. 图形展示
    ![版本说明](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/361638F3E7F04EFCBD76B877A1CEC2F5/20866)      
    ![测试](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/325DCB285CC44789B36BF875969C5D9F/20869)  
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/93D3169A3D224EE68E8BC9A1261C87E5/20871)  
=====================================================
更新于2018年10月26日 
最后说明
上面 “八、特别说明：“  提到的问题，已经解决了。  
是由于配置文件里的CALICO_NODENAME 环境变量，赋值错误导致的。  
环境变量CALICO_NODENAME 的值？ 
- 方案一：不设置，
```
    ETCD_ENDPOINTS="http://172.16.91.205:2379,http://172.16.91.206:2379,http://172.16.91.207:2379"
    ETCD_CA_FILE=""
    ETCD_CERT_FILE=""
    ETCD_KEY_FILE=""
    CALICO_NODENAME=""
    CALICO_NO_DEFAULT_POOLS=""
    CALICO_IP=""
    CALICO_IP6=""
    CALICO_AS=""
    CALICO_LIBNETWORK_ENABLED=true
    CALICO_NETWORKING_BACKEND=bird
```
- 方案二：设置成主机名
```
    ETCD_ENDPOINTS="http://172.16.91.205:2379,http://172.16.91.206:2379,http://172.16.91.207:2379"
    ETCD_CA_FILE=""
    ETCD_CERT_FILE=""
    ETCD_KEY_FILE=""
    CALICO_NODENAME="master"
    CALICO_NO_DEFAULT_POOLS=""
    CALICO_IP=""
    CALICO_IP6=""
    CALICO_AS=""
    CALICO_LIBNETWORK_ENABLED=true
    CALICO_NETWORKING_BACKEND=bird
```  

