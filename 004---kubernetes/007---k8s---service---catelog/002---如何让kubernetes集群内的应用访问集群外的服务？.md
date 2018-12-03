__关键词：__  
- kubernetes  
- docker 
- Service Catalog
- Service Broker 
- Helm

# 一、背景介绍  
&ensp;&ensp;&ensp;&ensp;本文主要介绍如何让Kubernetes集群内的应用访问集群外的服务；从而实现应用与服务的分离，解耦，使得应用开发人员只关注自己的核心业务逻辑；而不用关心服务的生命周期。     
&ensp;&ensp;&ensp;&ensp;采取的主要方案是使用Service Catalog与Service Broker技术。   
&ensp;&ensp;&ensp;&ensp;至于Service Catalog与Service Broker的原理，可以参考一下的链接：  
&ensp;&ensp;&ensp;&ensp;https://blog.csdn.net/dkfajsldfsdfsd/article/details/81062077    
&ensp;&ensp;&ensp;&ensp;https://github.com/kubernetes-incubator/service-catalog     
&ensp;&ensp;&ensp;&ensp;https://github.com/openservicebrokerapi/servicebroker   


# 二、测试环境介绍   
## 2.1 虚拟机环境介绍  
|**`系统类型`**|**`IP`**|**`role`**|**`cpu`**|**`memory`**|**`hostname`**|
|:---|:---|:---|:---|:---|:---|
|CentOS 7.4.1708|172.16.91.215|master|4|4G|master|
|CentOS 7.4.1708|172.16.91.216|worker|4|4G|slave1|
|CentOS 7.4.1708|172.16.91.217|worker|2|1G|slave2|  
`注意:`  
最好将内存和cpu分配的多些，不然，很有可能导致catalog-apiserver中的etcd服务不能正常提供服务；  
或者部署catalog时，修改etcd中的资源属性配置  

## 2.2 现有服务介绍  
1. Kubernetes  服务  
    - 部署方式  
        kubernetes使用kubeadm方式部署的，
    - 版本  
        1.9.0  
    - 可以参考下面的链接，进行部署  
        https://www.jianshu.com/p/48695bd6401f   

2. docker 服务  
    - docker版本  
    ![docker版本](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/906253C78F2345CCBB3FBAB7409FD344/22251)   
3. helm 服务  
    helm的部署，可以参考下面的链接  
    https://www.jianshu.com/p/200020e7a843  
    `注意:`  
    Tiller的权限设置，不要参考上面链接的，可以使用下面的方式：  
    ![Tiller权限设置](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/CC8845983F0C430DAE7F8F83A0F621E7/22253)   
    https://github.com/kubernetes-incubator/service-catalog/blob/master/docs/install.md    

# 三、主要步骤说明   
## 3.1 部署方式说明   
使用Helm来部署Service Catalog 和 Service Borker 

## 3.2 主要分为以下步骤：  
- 部署Service Catalog  
    - 部署版本号:0.1.27   
- 部署Service Borker   
    - https://github.com/xej520/shared-mysql-service-broker
- 测试

# 四、部署Service Catalog   
## 4.1 下载catalog-0.1.27.tgz  
```
helm fetch svc-cat/catalog --version 0.1.27 
```
## 4.2 部署Service Catalog  
- 方式一：可以直接安装部署，    
- 方式二：根据自己的需求，做一些修改  
### 4.2.1 本文采用方式二   
1. 解压catalog-0.1.27.tgz  
    tar -zxvf catalog-0.1.27.tgz  
    cd catalog  
    ![解压catalog](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/242BCE2D68194150B74FEAEBF4E47CA1/22255)   
2. 编辑values.yaml文件  
    - 修改镜像的拉取策略:  
        - 将imagePullPolicy: Always修改为imagePullPolicy: IfNotPresent
    -  将健康校验禁止掉(可能是docker版本的问题，导致健康校验总是报错，从而导致pod服务异常)   
        - 将enabled: true修改为enabled: false      

    - 或者直接使用下面的命令来替换(`下面的命令有问题，还在排查中`)  
        sed -i 's#imagePullPolicy:&ensp;Always#imagePullPolicy:&ensp; IfNotPresent#g'&ensp;values.yaml   
        sed -i 's#enabled:&ensp;true#enabled:&ensp;false#g'&ensp;values.yaml   
        sed -i 's#image:&ensp;quay.io/kubernetes-service-catalog/service-catalog:v0.1.27#image:&ensp;registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/catalog:v0.1.27#g'&ensp;values.yaml   
    ![values.yaml](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/7395AB1594E646CB9910850599085118/22258)         
3. cd templates   
    `下面的命令，有问题，建议手动修改`  
    sed -i 's#imagePullPolicy: Always#imagePullPolicy: IfNotPresent#g'&ensp;apiserver-deployment.yaml  
    sed -i 's#image:&ensp;quay.io/coreos/etcd:latest#image:&ensp;registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/etcd:3.2.24#g'&ensp;apiserver-deployment.yaml      
    `注意`   
    etcd版本，不一定非得是3.2.24


4. 部署Service Catalog  
    - helm install /root/catalog --name catalog --namespace catalog   
    - 如何安装失败的话，就删除
        - helm del --purge catalog  
    ![install catalog](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/61385EBB1C0A43BF93B0D6AF1D70D09E/22260)       
5. 查看部署情况  
    kubectl get pod -n catalog  
    ![查看catalog部署情况](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/D07EF46FDC3C4AE9AEE90CED856314AF/22262)
# 五、部署Service broker  
## 5.1 在k8s集群外创建mysql服务   
1. 下载mysql:5.6镜像(`不建议使用8.0或者最新的版本，可能会存在验证问题`)  
    docker pull mysql:5.6

2. 编写启动脚本   
    mkdir mysql   
    cd mysql

    tee start-mysql.sh <<-'EOF'  
    #!/bin/bash  
    cur_dir=\`pwd\`  
    docker  stop  test-mysql  
    docker  rm  test-mysql   
    docker run --name test-mysql -v ${cur_dir}/data:/var/lib/mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.6  
    EOF

3. chmod +x start-mysql.sh && ./start-mysql.sh    
    ![mysql服务容器化](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/742649D0549B466AB0D9CFC6303E30EC/22264)   
4. docker exec -it test-mysql bash    

5. 创建数据库   
    mysql -uroot -p123456   
    create database mysql_service_broker;
    show databases;      
    ![创建数据库](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/40D3E7F830094F60A79527069ADFD0A6/22266)   
## 5.2 部署mysql-broker   
1. 下载mysql-broker  
    git clone https://github.com/xej520/shared-mysql-service-broker.git  
2. cd shared-mysql-service-broker

3. kubectl apply -f k8s/namespace.yml  

4. cp k8s/secret.yml.sample k8s/secret.yml  

5. 需要根据自己的实际情况，来修改密钥   
    文件中，注释的地方说明了如何得到具体的值。  
    - 修改mysql的url地址  
    - 修改用户名和密码  
    ![修改secret.yaml](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/E8C1A8DAA9084E9C910A303AC5063BE2/22269)   
    - kubectl create -f k8s/secret.yaml   
    ![create secret](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/FDA99C0208F149C2A6A4737B84E3AF44/22271)    
6. 修改k8s/deployment.yml文件中的镜像拉取策略  
    将imagePullPolicy: Always修改为imagePullPolicy: IfNotPresent 
7. 修改k8s/deployment.yml文件中的镜像   
    将image: making/shared-mysql-service-broker:0.0.3 修改为image: registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/shared-mysql-service-broker:0.0.3  

8. kubectl apply -f k8s/deployment.yml
    - kubectl get pod -nosb #查看部署是否成功？   
    ![create shared-mysql-service-broker](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/B56C6D90687344F19FDF17D503C3F9C0/22273)  
9. kubectl apply -f k8s/cluster-service-broker.yml   
    - 查看部署情况  
    kubectl get clusterservicebrokers,clusterserviceclasses,clusterserviceplans,serviceinstances,servicebindings -oyaml  

10. 创建实例  
    - kubectl apply -f k8s/sample/service-instance.yml  
    - 查看实例对象是否创建成功？  
        - kubectl get clusterservicebrokers,clusterserviceclasses,clusterserviceplans,serviceinstances,servicebindings -oyaml  
    - `如果存在问题，就不要往下进行了，先解决问题。`   
    ![create instance](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/6C1CC6D11FA54410B887702A81A5475A/22275)     
11.  binding  
        -  kubectl apply -f k8s/sample/service-binding.yml      
        - 查看bingding对象是否创建成功？  
            - kubectl get clusterservicebrokers,clusterserviceclasses,clusterserviceplans,serviceinstances,servicebindings -oyaml  
        - 如果存在问题，就不要往下进行了，先解决问题。 
        - 如果创建成功的话，会在默认命名空间下，会创建一个secret对象，名称是wp-db-secret    
        - kubectl get secret wp-db-secret -oyaml  
        ![查看是否绑定成功](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/5FD5D533FF70490A8627C7726ABC94B0/22279)   
        ![查看wp-db-secret对象](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/5195640CEDD945CB80ED57B9F132F35A/22277)      
k8s集群内的应用就是通过wp-db-secret这个密钥，访问外面的mysql服务的。   
# 六、测试：如何使得k8s集群内的应用访问集群外的服务？  
1. 使用k8s/sample/wordpress.yaml   
    `如果报挂载问题的话，可以做以下修改`   
    mkdir /usr/local/wd  
    ![删除PVC](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/48261E326B6A4D7FA805BFF5BD48CD4D/22281)  
    ![更新hostPath](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/227DF90350B741888A5E85E13923D61B/22283)     
    

2. kubectl create -f k8s/sample/wordpress.yaml    

3. kubectl get pod -owide
    ![查看wordpress](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/4586D521EFD84FBF94F292491ACA7336/22285)  
4. 通过curl命令来获取catalog  
    随便创建一个pod,然后执行下面的命令即可

    - curl http://admin:123456@shared-mysql-service-broker-service.osb.svc.cluster.local/v2/catalog    
    - curl -u admin:123456 http://shared-mysql-service-broker-service.osb.svc.cluster.local/v2/catalog   
    - curl -u admin:123456 http://shared-mysql-service-broker-service.osb.svc.cluster.local/v2/catalog -H "X-Broker-API-Version: 2.12"
    ![GET /v2/catalog](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/462BFF932D8D498DBE44C4CC8F52E8C8/22287)  

