# kubernetes 1.12.1 Ingress 部署使用  
参考文献：  
https://github.com/kubernetes/ingress-nginx/blob/nginx-0.20.0/docs/deploy/index.md
https://www.jianshu.com/p/0a58e61627c4  
https://www.jianshu.com/p/02845519b0e0  
https://blog.csdn.net/bbwangj/article/details/82940419  
https://www.jianshu.com/p/121b58782865  

测试环境：  
|组件|版本号|
|:---|:---|
|kubernetes|v1.12.1|
|ingress-nginx|0.20.0|
|centos|7.5|
|docker|17.03.2-ce|


# 一、Ingress 简介  
在Kubernetes中，服务和Pod的IP地址仅可以在集群网络内部使用，对于集群外的应用是不可见的。  
为了使外部的应用能够访问集群内的服务，  
在Kubernetes 目前 提供了以下几种方案：  
- NodePort 
- LoadBalancer  
- Ingress  

1. __Ingress 组成？__  
    - ingress controller  
        - 将新加入的Ingress转化成Nginx的配置文件并使之生效  
    - ingress服务  
        - 将Nginx的配置抽象成一个Ingress对象，每添加一个新的服务只需写一个新的Ingress的yaml文件即可  
2. __Ingress 工作原理?__  
    - ingress controller通过和kubernetes api交互，动态的去感知集群中ingress规则变化，  

    - 然后读取它，按照自定义的规则，规则就是写明了哪个域名对应哪个service，生成一段nginx配置，  
    - 再写到nginx-ingress-control的pod里，这个Ingress controller的pod里运行着一个Nginx服务，控制器会把生成的nginx配置写入/etc/nginx.conf文件中，  
    - 然后reload一下使配置生效。  
    以此达到域名分配置和动态更新的问题。

3. __Ingress 可以解决什么问题？__  
    - 动态配置服务  
    如果按照传统方式, 当新增加一个服务时, 我们可能需要在流量入口加一个反向代理指向我们新的k8s服务. 而如果用了Ingress, 只需要配置好这个服务, 当服务启动时, 会自动注册到Ingress的中, 不需要而外的操作.  
    - 减少不必要的端口暴露  
    配置过k8s的都清楚, 第一步是要关闭防火墙的, 主要原因是k8s的很多服务会以NodePort方式映射出去, 这样就相当于给宿主机打了很多孔, 既不安全也不优雅. 而Ingress可以避免这个问题, 除了Ingress自身服务可能需要映射出去, 其他服务都不要用NodePort方式

4. __Ingress当前的实现方式？__  
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/7A3667F3488848C9BCF4FC103348DA42/20501)  

本文使用的是基于nginx的ingress  


# 二、部署配置Ingress  
## 2.1 部署文件介绍、准备 
### 第一步： 获取配置文件位置   
https://github.com/kubernetes/ingress-nginx/tree/nginx-0.20.0/deploy  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/E604656883B5466BA3A23A4B92579210/20504)  

### 第二步： 下载部署文件 
提供了两种方式 ：  
- 默认下载最新的yaml 
- 指定版本号下载对应的yaml  
1. __默认下载最新的yaml__  
    ```
    wget  https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
    ```  
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/403A6D89D5CB4F32A7CC1BD0F61BE009/20511)  
2. __指定版本号下载对应的yaml__   
如下载ingress-nginx 0.17.0对应的yaml  
    ```
    wget  https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.17.0/deploy/mandatory.yaml
    ```  
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/A87BD58360ED4A2CBB6949F1A3E1EC09/20507)  

### 部署文件介绍  
1. __namespace.yaml__  
创建一个独立的命名空间 ingress-nginx  
2. __configmap.yaml__   
ConfigMap是存储通用的配置变量的，类似于配置文件，使用户可以将分布式系统中用于不同模块的环境变量统一到一个对象中管理；而它与配置文件的区别在于它是存在集群的“环境”中的，并且支持K8S集群中所有通用的操作调用方式。    
从数据角度来看，ConfigMap的类型只是键值组，用于存储被Pod或者其他资源对象（如RC）访问的信息。这与secret的设计理念有异曲同工之妙，主要区别在于ConfigMap通常不用于存储敏感信息，而只存储简单的文本信息。  
ConfigMap可以保存环境变量的属性，也可以保存配置文件。  
创建pod时，对configmap进行绑定，pod内的应用可以直接引用ConfigMap的配置。相当于configmap为应用/运行环境封装配置。  

pod使用ConfigMap，通常用于：设置环境变量的值、设置命令行参数、创建配置文件。  

3. __default-backend.yaml__   
如果外界访问的域名不存在的话，则默认转发到default-http-backend这个Service，其会直接返回404：  
4. __rbac.yaml__    
负责Ingress的RBAC授权的控制，其创建了Ingress用到的ServiceAccount、ClusterRole、Role、RoleBinding、ClusterRoleBinding  

5. __with-rbac.yaml__  
是Ingress的核心，用于创建ingress-controller。前面提到过，ingress-controller的作用是将新加入的Ingress进行转化为Nginx的配置  

## 2.2 部署ingress  
### 第一步： 准备镜像，从这里mandatory.yaml查看需要哪些镜像  
已经准备好， 可以直接点击下载  
|镜像名称|版本|下载地址|
|:---|:---|:---|
|k8s.gcr.io/defaultbackend-amd64|1.5|registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/defaultbackend-amd64|
|quay.io/kubernetes-ingress-controller/nginx-ingress-controller|0.20.0|registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/nginx-ingress-controller|
如：  
docker pull registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/nginx-ingress-controller:0.20.0  
将镜像上传到自己的私有仓库，以供下面的步骤使用。  
### 第二步： 更新mandatory.yaml中的镜像地址 
替换成自己的镜像地址：   
1. __替换defaultbackend-amd64镜像地址__
    ```
    sed -i 's#k8s.gcr.io/defaultbackend-amd64#registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/defaultbackend-amd64#g' mandatory.yaml
    ```  
2. __替换nginx-ingress-controller镜像地址__ 
    ```
    sed -i 's#quay.io/kubernetes-ingress-controller/nginx-ingress-controller#registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/nginx-ingress-controller#g' mandatory.yaml
    ```   

### 第三步： 部署nginx-ingress-controller  

```
kubectl apply -f mandatory.yaml
``` 
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/B28A07F799F247B594F361BBAF0FDFAF/20518)  


### 第四步： 查看ingress-nginx组件状态？  
1. __查看相关pod状态__ 
    ```
    kubectl get pods -n ingress-nginx -owide
    ```  
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/BCC57ECA0D79416B9EC6785E59C4CDB9/20521)  
    ```
    [root@master ingress-nginx]# kubectl get pods -n ingress-nginx -owide
    NAME                                       READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE
    default-http-backend-7f594549df-nzthj      1/1     Running   0          3m59s   192.168.1.90    slave1   <none>
    nginx-ingress-controller-9fc7f4c5f-dr722   1/1     Running   0          3m59s   192.168.2.110   slave2   <none>
    [root@master ingress-nginx]# 
    ```  

2. __查看service状态__  
    ```
    [root@master ingress-nginx]# kubectl get service -n ingress-nginx
    NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
    default-http-backend   ClusterIP   10.104.146.218   <none>        80/TCP    5m37s
    [root@master ingress-nginx]# 

    ```

# 三、测试   
## 3.1 测试default-http-backend 是否起作用？      
系统自动安装了一个default-http-backend pod， 这是一个缺省的http后端服务， 用于返回404结果，如下所示：  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/5F3ADAEC42FB4B0BB6C16971D8811F1E/20523)  

接下会做一些关于ingress的详细测试  














