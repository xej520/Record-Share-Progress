# kubernetes Network Policies 之 ALLOW traffic from external clients    

## 1、规则描述：  
希望达到的效果：  
如下图所示：  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/7B28D5964F864316BC7989FA2B9B51E3/20465)    

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

## 4、创建相应的Pod  
1. 在default命名空间下, 创建一个带有Nginx服务的pod,名称是appleshop  
    ```
    kubectl run appleshop --image=mybusybox --labels role=appleshop,app=web --image-pull-policy=IfNotPresent --port 80 
    //将nginx服务映射到了物理机上
    kubectl expose deployment/appleshop --type=LoadBalancer
    ```
2. 在default命名空间下, 创建一个带有Nginx服务的pod,名称是gameshop  
    ```
    kubectl run gameshop --image=mybusybox --labels app=gameshop --image-pull-policy=IfNotPresent
    ``` 
3. 在prod命名空间下, 创建一个带有Nginx服务的pod,名称是bookstore   
    ```
    kubectl run bookstore -n prod --image=mybusybox  --labels app=web,role=bookstore   --image-pull-policy=IfNotPresent 
    ```  
    
4. 查看Pod, SVC创建情况  
    ```
    [root@master ~]# kubectl get pod -owide
    NAME                         READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    appleshop-5f9668b84c-6ltvs   1/1     Running   0          49s   192.168.1.70   slave1   <none>
    gameshop-5b95c4fc4d-6zdrt    1/1     Running   0          27s   192.168.2.92   slave2   <none>
    [root@master ~]# kubectl get pod -owide -nprod
    NAME                        READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    bookstore-88c9d8bf6-8bvlg   1/1     Running   0          29s   192.168.1.71   slave1   <none>
    [root@master ~]# kubectl get svc 
    NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
    appleshop    LoadBalancer   10.96.62.187   <pending>     80:30771/TCP   44s
    kubernetes   ClusterIP      10.96.0.1      <none>        443/TCP        5d7h
    [root@master ~]# 

    ```


## 5、开始测试  
### 5.1、未使用策略前的测试
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;默认未使用任何策略时，可以互相访问的，不再具体测试了
    
### 5.2、开始使用策略的测试  
0. 说明： 本次的测试是在网络策略default-deny-all.yaml基础之上进行的  
    ```
    kind: NetworkPolicy
    apiVersion: networking.k8s.io/v1
    metadata:
      name: default-deny-all
      namespace: default
    spec:
      podSelector: {}
      ingress: []
    ```
    这个策略实现的效果就是，default命名空间下的pod，出站规则允许，入站规则禁止了。  就是只能访问其他命名空间，本命名空间里的pod，由于入站规则禁止了，因此不能访问  
1. 创建NetworkPolicy类型的yaml文件，如 web-allow-external.yaml 
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata: 
      name: web-allow-external
    spec: 
      podSelector: 
        matchLabels: 
          app: web
      ingress: 
      - from: []  
    ```   
    ![](https://note.youdao.com/yws/public/resource/a431131093aff717feb4fd8bc812c5f5/xmlnote/0F368A274F0A4BE889443C49CFC4049A/20422)  
2. 使其生效  
    ``` 
    kubectl create -f default-deny-all.yaml
    kubectl create -f web-allow-external.yaml
    ```
3. 测试  

    3.1 测试点1： 测试命名空间default下pod的ingress规则？  
    ```
    在以前的文章中已经测试过了，这里就不再测试了
    ```

    3.2 测试点2： prod命名空间里的bookstroe 是否可以访问appleshop里的Nginx服务？  
    ```
    [root@master day03]# kubectl exec -it bookstore-88c9d8bf6-8bvlg sh -nprod
    # wget 192.168.1.70
    --2018-10-14 08:02:18--  http://192.168.1.70/
    Connecting to 192.168.1.70:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html.1'

    index.html.1                           100%[==========================================================================>]     612  --.-KB/s    in 0s      

    2018-10-14 08:02:18 (174 MB/s) - 'index.html.1' saved [612/612]

    # exit
    [root@master day03]# 
    ```

    3.3 测试点3： k8s集群外，如172.16.91.222节点，是否可以访问appleshop里的Nginx服务？
    ```
    [root@harbor ~]# wget 172.16.91.135:30771
    --2018-10-14 04:05:22--  http://172.16.91.135:30771/
    Connecting to 172.16.91.135:30771... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: ‘index.html.1’

    100%[==============================================================================================================================>] 612         --.-K/s   in 0s      

    2018-10-14 04:05:22 (134 MB/s) - ‘index.html.1’ saved [612/612]

    [root@harbor ~]# 
    ```

    说明：  
    &ensp;&ensp;&ensp;&ensp;网络策略配置生效 

## 5、清理工作  
    kubectl delete networkpolicy  web-allow-external
    kubectl delete deployment appleshop gameshop
    kubectl delete deployment bookstore -n prod 
    kubectl delete svc appleshop  -n prod






