# kubernetes Network Policies 之 限制其他命名空间的pod访问本命名空间的pod    

## 1、规则描述：  
希望达到的效果：  
1. 命名空间default下的所有pod，可以互相访问
2. 禁止非default命名空间下的pod访问default命名空间下的pod  

如下图所示：  
![](https://note.youdao.com/yws/public/resource/a431131093aff717feb4fd8bc812c5f5/xmlnote/B0DAA9FB210B4230AD32F67CDAEFE1DD/20405)   

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
## 3、创建相应的Pod  
1. 在default命名空间下, 创建一个带有app=nginx标签的pod,名称是bookstore  
     ```
    kubectl run bookstore --image=mybusybox --labels app=bookstore --image-pull-policy=IfNotPresent
    ```
2. 在default命名空间下, 创建一个带有Nginx服务的pod,名称是frontend  
    ```
    kubectl run frontend --image=mybusybox --labels app=frontend  --image-pull-policy=IfNotPresent
    ``` 
3. 在policy-demo命名空间下, 创建一个带有Nginx服务的pod,名称是coffeeshop   
    ```
    kubectl run coffeeshop -n policy-demo --image=mybusybox --labels app=coffeeshop  --image-pull-policy=IfNotPresent
    ```  
4. 查看pod创建情况  
    ```
    [root@master day02]# kubectl get pod  -owide
    NAME                         READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    bookstore-6d5b4dc66d-qvphv   1/1     Running   0          78s   192.168.1.36   slave1   <none>
    frontend-86dcfb45d4-npsx5    1/1     Running   0          32s   192.168.2.55   slave2   <none>
    [root@master day02]# kubectl get pod  -owide -n policy-demo
    NAME                         READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    coffeeshop-cf9b57f64-x9bgp   1/1     Running   0          31s   192.168.1.37   slave1   <none>
    [root@master day02]# 
    ```  
## 4、开始测试  
### 4.1、未使用策略前的测试
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;默认未使用任何策略时，可以互相访问的，不再具体测试了
    
### 4.2、开始使用策略的测试  
1. 创建NetworkPolicy类型的yaml文件，如 deny-from-other-namespaces.yaml 
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata: 
      name: deny-from-other-namespaces
      namespace: default
    spec: 
      podSelector: 
        matchLables: 
      ingress: 
      - from: 
        - podSelector: {}
    ```  
    namespace: default, 表示，该policy的使用范围是default命名空间下的pod  
    ingerss: rule 定义的规则，表示入站规则 匹配default命名下所有的pod  

2. 使其生效  
    ```
    kubectl create -f deny-from-other-namespaces.yaml
    ```
3. 测试  

    3.1 测试点1： 测试default命名空间下的bookstore pod是否可以访问frontend pod呢？    
    ```
    # 第一步：进入bookstore pod， 去访问frontend 里的nginx服务 
    [root@master day02]# kubectl exec -it bookstore-6d5b4dc66d-qvphv sh 
    # wget 192.168.2.55
    --2018-10-11 07:46:47--  http://192.168.2.55/
    Connecting to 192.168.2.55:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html'

    index.html                                100%[==================================================================================>]     612  --.-KB/s    in 0s      

    2018-10-11 07:46:47 (110 MB/s) - 'index.html' saved [612/612]

    # exit

    ----------------------------------------------  

    # 第二步：进入frontend pod， 去访问bookstore里的Nginx服务  
    [root@master day02]# kubectl exec -it frontend-86dcfb45d4-npsx5 sh
    # wget 192.168.1.36
    --2018-10-11 07:48:37--  http://192.168.1.36/
    Connecting to 192.168.1.36:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html'

    index.html                                100%[==================================================================================>]     612  --.-KB/s    in 0s      

    2018-10-11 07:48:37 (91.5 MB/s) - 'index.html' saved [612/612]

    # 

    ```  
    说明：  
    default命名空间下的pod，可以互相访问彼此的服务  

    3.2 测试点2： 测试policy-demo命名空间下的coffeeshop pod 是否可以访问default命名空间下的bookstore pod?   
    ```
    [root@master day02]# kubectl exec -it coffeeshop-cf9b57f64-x9bgp sh -npolicy-demo
    # wget 192.168.1.36
    --2018-10-11 07:50:34--  http://192.168.1.36/
    Connecting to 192.168.1.36:80... failed: Connection timed out.
    Retrying.

    --2018-10-11 07:52:42--  (try: 2)  http://192.168.1.36/
    Connecting to 192.168.1.36:80... ^C
    ```


## 5、清理工作  
    kubectl delete networkpolicy deny-from-other-namespaces
    kubectl delete deployment bookstore frontend
    kubectl delete deployment coffeeshop -n policy-demo 






