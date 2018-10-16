# kubernetes Network Policies 之  DENY external egress traffic
__实现效果图:__  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/88DD15174AC8496FB1C5DF59F37DE301/20458)  


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
1. 在default命名空间下, 创建一个deployment, 名称是hadoop  
    ```
    kubectl run hadoop --image=mybusybox --labels app=foo --image-pull-policy=IfNotPresent   
    ```
2. 在default命名空间下, 创建一个deployment, 名称是spark  
    ```
    kubectl run spark --image=mybusybox --labels app=gameshop  --image-pull-policy=IfNotPresent
    ``` 
3. 在prod命名空间下, 创建一个deployment, 名称是zk   
    ```
    kubectl run zk -n prod --image=mybusybox --labels app=foo --image-pull-policy=IfNotPresent
    ```    
4. 查看pod、svc创建情况  
    ```
    [root@master day04]# kubectl get pod -owide
    NAME                      READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    hadoop-6d854b84ff-9h9fn   1/1     Running   0          28s   192.168.2.90   slave2   <none>
    spark-6c68fcb8c7-2bxcb    1/1     Running   0          23s   192.168.1.67   slave1   <none>
    [root@master day04]# kubectl get pod -owide -nprod
    NAME                  READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    zk-7dd5f8cb69-bpt65   1/1     Running   0          20s   192.168.1.68   slave1   <none>
    [root@master day04]# 
    ```

## 5、开始测试  
### 5.1、未使用策略前的测试
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;默认未使用任何策略时，可以互相访问的，不再具体测试了
    
### 5.2、开始使用策略的测试  
 
1. 创建NetworkPolicy类型的yaml文件，如 foo-deny-external-egress.yaml 
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
    name: foo-deny-external-egress
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
    - to: 
        - namespaceSelector: {}
    ``` 
    
2. 使其生效  
    ``` 
    kubectl create -f foo-deny-external-egressyaml
    ```  
    
3. 测试  

    3.1 测试点1： 测试网络策略中定义的egress规则？是否允许出站流量  
    思路1：hadoop Pod是否可以访问spark Pod里的Nginx服务呢？ 
    ```
    [root@master day04]# kubectl exec -it hadoop-6d854b84ff-9h9fn sh 
    # wget 192.168.1.67
    --2018-10-14 03:03:35--  http://192.168.1.67/
    Connecting to 192.168.1.67:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html'

    index.html                                100%[==================================================================================>]     612  --.-KB/s    in 0s      

    2018-10-14 03:03:35 (105 MB/s) - 'index.html' saved [612/612]

    # 

    ```  
    说明可以访问本命名空间下的pod里提供的nginx服务
    思路2：hadoop Pod是否可以访问 prod命名空间下的zk pod里的nginx服务呢？  
    ```
    [root@master day04]# kubectl exec -it hadoop-6d854b84ff-9h9fn sh 
    # wget 192.168.1.68
    --2018-10-14 03:04:40--  http://192.168.1.68/
    Connecting to 192.168.1.68:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html.1'

    index.html.1                              100%[==================================================================================>]     612  --.-KB/s    in 0s      

    2018-10-14 03:04:40 (83.7 MB/s) - 'index.html.1' saved [612/612]

    # 

    ```  
    说明可以访问其他命名空间下的pod里提供的nginx服务
    思路3：hadoop Pod是否可以访问k8s集群外的服务？ 
    ```
    [root@master day04]# kubectl exec -it hadoop-6d854b84ff-9h9fn sh 
    # wget 172.16.91.222:6379
    --2018-10-14 03:06:41--  http://172.16.91.222:6379/
    Connecting to 172.16.91.222:6379... ^C
    # 

    ```

    说明不可以访问k8s集群外的redis服务

    3.2 测试点2：如果想访问k8s集群外的服务呢？   
    更新网络策略，添加出站规则，ip段, 最新的配置文件如下：  
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
    name: foo-deny-external-egress
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
    - to:
        - namespaceSelector: {}
        - ipBlock:
        cidr: 172.16.0.0/16

    ```    
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/AFB725BE8F2C42DB976660570AEC556A/20463)  
    测试如下：   
    ```
    [root@master day04]# kubectl exec -it hadoop-6d854b84ff-9h9fn sh 
    # wget 172.16.91.222:6379
    --2018-10-14 03:06:41--  http://172.16.91.222:6379/
    Connecting to 172.16.91.222:6379... ^C
    # ^[[A : not found
    # ^C2: 
    # wget 172.16.91.222:6379 
    --2018-10-14 03:11:12--  http://172.16.91.222:6379/
    Connecting to 172.16.91.222:6379... connected.
    HTTP request sent, awaiting response... 200 No headers, assuming HTTP/0.9
    Length: unspecified
    Saving to: 'index.html.2'

    index.html.2                                  [                <=>                                                                ]     225  --.-KB/s     
    ```  
    
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/E3EEF21BE4594BED8AE5A7F175995347/20455)  
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/3828C9D73A304CF68AAAD1BF52BC1806/20460)  


## 5、清理工作  
    kubectl delete networkpolicy foo-deny-external-egress
    kubectl delete deployment spark hadoop
    kubectl delete deployment zk -n prod 







