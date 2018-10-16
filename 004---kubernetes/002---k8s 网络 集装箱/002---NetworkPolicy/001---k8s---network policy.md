# kubernetes Network Policies 之 禁止访问所有命名空间下的pod  
换句话说，就是pod之间彼此隔离，互相不能访问  
主要参考文献：  
https://kubernetes.io/docs/concepts/services-networking/network-policies/  
__建议：__  
在进行下次network policy练习之间，先将本次练习用的policy等资源删除， 不然很有可能会影响到后面的测试效果  

__NetworkPolicy__  
```
1. 通过label标签来选择pod 
2. 通过定义rules规则来制定哪些流量可以访问制定的pod的，或者说，rule是用来表明流量的方向的，可以到哪里，不可以到哪里  
```  
镜像准备：  

1. 下载Nginx镜像 
    ```
    docker pull nginx
    ```  
2. 编写Dockerfile(添加wget) 
    ```
    FROM nginx
    RUN apt-get update 
    RUN apt-get install -y iputils-ping iproute2 wget
    ```  
    主要是想安装上ip等网络命令     
3. 重新构建镜像    
docker build -t mybusybox .  


# 1、Defualt policies
如果不配置policy的话，默认策略：  
出站和入站都是允许的 
## 1.1 禁止所有入站规则  
1. 创建yaml(default-ingress-deny.yaml)
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
    name: default-deny
    spec:
    podSelector: {}
    policyTypes:
    - Ingress  
    ```   
规则说明：  
podSelector: {}， 表示：针对的是所有命名空间下的所有pod 
policyTypes，设置的类型仅对是Ingress， 
但是，针对Ingress类型，没有设置具体的规则，默认就是不允许
因此，这个策略的效果就是，所有的pod都不能互相访问彼此的应用
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/25F9A2DCD341457EAE03D587696C2094/20379)  
ICMP，TCP协议不支持

## 1.2 针对这种策略进行测试 
测试目的:  
测试所有pod不允许其他pod对其进行访问  
如下如所示：  

### 1.2.1 创建测试pod  
1. 在default命名空间下，创建一个Nginx deployment, 名字是d-nginx1 
    ``` 
    kubectl run -n default d-nginx1 --replicas=1 --image=mybusybox --image-pull-policy=IfNotPresent
    ```
2. 在default命名空间下，创建一个deployment, 名称是:d-nginx2  
    ```
    kubectl run -n default d-nginx2 --replicas=1 --image=mybusybox --image-pull-policy=IfNotPresent
    ```
3. 在policy-demo命名空间下，创建一个deployment, 名称是:p-busybox1   
    ```
    kubectl run -n policy-demo p-nginx1 --replicas=1 --image=mybusybox  --image-pull-policy=IfNotPresent
    ```
### 1.2.2 具体测试  
0. 查看pod 
    ```
    1. 查看default命名空间下的pod  

    [root@master ~]# kubectl get pod -n default -owide
    NAME                        READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    bb                          1/1     Running   20         31h   192.168.1.13   slave1   <none>
    d-nginx1-779cffd574-9wgjl   1/1     Running   0          30s   192.168.1.25   slave1   <none>
    d-nginx2-788b974d8b-7j9q2   1/1     Running   0          26s   192.168.1.26   slave1   <none>

    2. 查看policy-demo命名空间下的pod
    [root@master ~]# kubectl get pod -n policy-demo  -owide
    NAME                        READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    p-nginx1-54f786698b-7l46p   1/1     Running   0          66m   192.168.2.35   slave2   <none>

    ```
1. 测试podA，是否可以访问podB里的Nginx服务  
    ```
    [root@master ~]# kubectl exec -it d-nginx1-779cffd574-9wgjl sh
    # wget 192.168.1.26
    --2018-10-10 11:11:57--  http://192.168.1.26/
    Connecting to 192.168.1.26:80... 

    ``` 
    无法访问  
2. 测试podA, 是否可以访问podC里的Nginx服务

    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/61A30970F6FC463790CD0234CB9C7055/20386)  
    无法访问  
3. 测试podB,是否可以访问podA里的Nginx服务  
    ```
    [root@master ~]# kubectl exec -it d-nginx2-788b974d8b-7j9q2 sh
    # wget 192.168.1.25
    --2018-10-10 11:15:55--  http://192.168.1.25/
    Connecting to 192.168.1.25:80... 
    ```
    无法访问  
4. 测试podC,是否可以访问podA里的Nginx服务  
    ```
    [root@master ~]# kubectl exec -it  p-nginx1-54f786698b-7l46p  sh -npolicy-demo
    # wget 192.168.1.25
    --2018-10-10 11:17:11--  http://192.168.1.25/
    Connecting to 192.168.1.25:80... failed: Connection timed out.
    Retrying.
    

    ```  
    无法访问  
5. 测试podA,是否可以ping通podB,podC(测试ICMP协议是否支持) 
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/DEE44BCD12864577894034468D844A22/20388)  
    ping不通，不支持  

## 2、 Clean up  
```
kubectl delete networkpolicy default-deny 
kubectl delete deployment d-nginx1 d-nginx2 
kubectl delete deployment p-nginx1 -npolicy-demo 
```



