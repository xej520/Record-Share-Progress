# kubernetes Network Policies 之  DENY egress traffic from an application
__实现效果图:__  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/07AAED60DE304BB5963CEDD141FD19B1/20453)


## 1、__使用场景__  
- 阻止应用程序建立到Pod外部的任何连接；
- 用于限制单实例数据库和数据存储的出站流量。

## 2、使用镜像准备：  
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
    ```  
    docker build -t mybusybox .  
    ```    
## 3、创建测试用的命名空间，prod
    
    # 创建 命名空间
    kubectl create namespace prod 

## 4、创建相应的Pod、SVC  
1. 在default命名空间下, 创建一个deployment, 名称是nginx  
    ```
    kubectl run nginx --image=mybusybox --port 80 --expose --labels="app=web,role=nginx" --image-pull-policy=IfNotPresent   
    ```
2. 在default命名空间下, 创建一个deployment, 名称是spark  
    ```
    kubectl run spark --image=mybusybox --labels app=foo  --image-pull-policy=IfNotPresent
    ``` 
3. 在default命名空间下，创建一个deployment, 名称分别是hadoop 
    ```
    kubectl run hadoop --image=mybusybox --labels app=query --image-pull-policy=IfNotPresent
    ``` 
4. 在prod命名空间下, 创建一个deployment, 名称是zk   
    ```
    kubectl run zk -n prod --image=mybusybox --labels="role=inventory,app=foo" --image-pull-policy=IfNotPresent
    ```    
5. 查看pod、svc创建情况  
    ```
    [root@master test]# kubectl get pod  -owide
    NAME                      READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    hadoop-5c986dfc55-vpxqn   1/1     Running   0          15s   192.168.1.64   slave1   <none>
    nginx-74cc9c969f-fc895    1/1     Running   0          26s   192.168.1.63   slave1   <none>
    spark-5bf9868c5-cn9gg     1/1     Running   0          21s   192.168.2.86   slave2   <none>
    [root@master test]# kubectl get pod  -owide -nprod
    NAME                  READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    zk-745f8c9795-vxdg8   1/1     Running   0          13s   192.168.2.87   slave2   <none>
    [root@master test]# kubectl get svc 
    NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
    kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   5d
    nginx        ClusterIP   10.100.233.200   <none>        80/TCP    43s
    [root@master test]# 

    ```  

## 5、开始测试  
### 5.1、未使用策略前的测试
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;默认未使用任何策略时，可以互相访问的，不再具体测试了
    
### 5.2、开始使用策略的测试  
 
1. 创建NetworkPolicy类型的yaml文件，如 foo-deny-egress.yaml 
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata: 
      name: foo-deny-egress
    spec: 
      podSelector: 
        matchLabels: 
          app: foo
      policyTypes: 
      - Egress 
      egress: []  
    ```   
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/3E47ECD49F9D4D018FE9E4497AB3BB35/20448)   
    
2. 使其生效  
    ``` 
    kubectl create -f foo-deny-egress.yaml
    ```  
    
3. 测试  

    3.1 测试点1： 测试网络策略中定义的egress规则？是否允许出站流量  
    思路：spark Pod是否可以访问Nginx Pod里的Nginx服务呢？ 
    ```
    [root@master test]# kubectl exec -it spark-5bf9868c5-cn9gg sh
    # wget 192.168.1.63
    --2018-10-14 01:04:30--  http://192.168.1.63/
    Connecting to 192.168.1.63:80... ^C
    # 
    ```  
    不能访问， 网络策略作用于这个spark Pod生效，禁止出站流量  

    3.2 测试点2： 测试网络策略中默认的igress规则？ 是否允许进入  
    思路：hadoop Pod是否可以访问 spark pod里的nginx服务呢？  
    ```
    [root@master test]# kubectl exec -it hadoop-5c986dfc55-vpxqn sh
    # wget 192.168.2.86
    --2018-10-14 01:06:24--  http://192.168.2.86/
    Connecting to 192.168.2.86:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html'

    index.html                                100%[==================================================================================>]     612  --.-KB/s    in 0.009s  

    2018-10-14 01:06:24 (65.6 KB/s) - 'index.html' saved [612/612]

    # 

    ```  
    说明，网络策略中， ingress策略默认是允许的，可以进行访问，

    3.2 测试点2： 测试网络策略的作用域？  
    思路：使用prod命名空间下的zk pod, 是否可以查询default命名空间下的hadoop Pod里的Nginx服务  
    ```
    [root@master test]# kubectl exec -it zk-745f8c9795-vxdg8 sh -nprod
    # wget 192.168.1.64
    --2018-10-14 01:10:02--  http://192.168.1.64/
    Connecting to 192.168.1.64:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html'

    index.html                                100%[==================================================================================>]     612  --.-KB/s    in 0s      

    2018-10-14 01:10:02 (33.0 MB/s) - 'index.html' saved [612/612]

    # 

    ```  
    说明，这个网络策略的作用域就是default命名空间里，符合要求的pod

    3.3 测试点3： 更新网络策略的Egress规则，允许DNS服务可以出站流量  
    - 更新策略前，测试dns服务  
    进入spark pod,ping www.baidu.com  
        ```
        [root@master test]# kubectl exec -it spark-5bf9868c5-cn9gg sh
        # ping www.baidu.com
        ping: www.baidu.com: Temporary failure in name resolution
        # 
        ```  
    - 更新网络策略配置文件，如下:  
        ```
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
        name: foo-deny-egress
        spec:
        podSelector:
            matchLabels:
            app: foo
        policyTypes:
        - Egress
        egress: 
        - ports:
            - port: 53
            protocol: UDP
            - port: 53
            protocol: TCP
        ```    
        测试，如下图：  
        ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/0565075C374B436B9211637D73FFA70E/20450)  



## 5、清理工作  
    kubectl delete networkpolicy foo-deny-egress
    kubectl delete deployment spark hadoop nginx 
    kubectl delete deployment zk -n prod 
    kubectl delete svc nginx  







