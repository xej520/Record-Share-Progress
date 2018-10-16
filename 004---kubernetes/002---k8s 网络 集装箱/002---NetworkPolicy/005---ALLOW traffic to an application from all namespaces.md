# kubernetes Network Policies 之  ALLOW traffic to an application from all namespaces   

## 1、规则描述：  
希望达到的效果：  
1. 命名空间default下的标签不是role=web的pod，不能互相访问
2. 非default命名空间下的pod可以访问default命名空间下的标签为role=web的pod  

如下图所示：  
![](https://note.youdao.com/yws/public/resource/a431131093aff717feb4fd8bc812c5f5/xmlnote/69DC04E8E79946408D679D5EF02406D2/20408)    

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
0. 创建一个命名空间policy2  
    ```
    kubectl create namespace policy2
    ```
1. 在default命名空间下, 创建一个带有app=nginx标签的pod,名称是bookstore  
    ```
    kubectl run bookstore --image=mybusybox --labels app=bookstore,role=web --image-pull-policy=IfNotPresent
    ```
2. 在default命名空间下, 创建一个带有Nginx服务的pod,名称是frontend  
    ```
    kubectl run frontend --image=mybusybox --labels app=frontend  --image-pull-policy=IfNotPresent
    ``` 
3. 在policy-demo命名空间下, 创建一个带有Nginx服务的pod,名称是coffeeshop   
    ```
    kubectl run coffeeshop -n policy-demo --image=mybusybox --labels app=coffeeshop  --image-pull-policy=IfNotPresent
    ```  
4. 在policy2命名空间下, 创建一个带有Nginx服务的pod,名称是appleshop   
    ```
    kubectl run appleshop -n policy2 --image=mybusybox --labels app=appleshop  --image-pull-policy=IfNotPresent
    ```     
5. 查看pod创建情况  
    ```
    [root@master day02]# kubectl get pod -owide
    NAME                         READY   STATUS    RESTARTS   AGE    IP             NODE     NOMINATED NODE
    bookstore-7466f96497-7zt55   1/1     Running   0          16s    192.168.2.58   slave2   <none>
    frontend-86dcfb45d4-cv29q    1/1     Running   0          109s   192.168.1.38   slave1   <none>
    [root@master day02]# kubectl get pod -owide -n policy-demo
    NAME                         READY   STATUS    RESTARTS   AGE    IP             NODE     NOMINATED NODE
    coffeeshop-cf9b57f64-ppm79   1/1     Running   0          116s   192.168.2.57   slave2   <none>
    [root@master day02]# kubectl get pod -owide -n policy2
    NAME                         READY   STATUS    RESTARTS   AGE    IP             NODE     NOMINATED NODE
    appleshop-5f9c7bf6cb-9fzx5   1/1     Running   0          113s   192.168.1.39   slave1   <none>
    [root@master day02]# 

    ```  


## 4、开始测试  
### 4.1、未使用策略前的测试
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;默认未使用任何策略时，可以互相访问的，不再具体测试了
    
### 4.2、开始使用策略的测试  
此次策略有两个策略对象组成，其中以前的文章中已经有了，这里直接拿过来使用 

0.  创建策略对象default-deny-all， default-deny-all.yaml
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata: 
      name: default-deny-all
      namespace: default
    spec: 
      podSelector: {}
      ingress: []
    ```
    这个策略实现的效果就是：命名空间default下的所有pod，不可以互相访问

1. 创建NetworkPolicy类型的yaml文件，如 web-allow-all-namespaces.yaml 
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata: 
      name: web-allow-all-namespaces
      namespace: default
    spec: 
      podSelector: 
        matchLabels: 
          role: web
      ingress: 
      - from: 
        - namespaceSelector: {}
    ```   

2. 使其生效  
    ```
    kubectl create -f default-deny-all.yaml
    kubectl create -f web-allow-all-namespaces.yaml
    ```
3. 测试  

    3.1 测试点1： 测试default命名空间下pod之间是否可以互相访问(针对的是pod的标签不是role=web的pod)？    
    ```
    以前的文章已经测试过，这里不再测试了
    ```
    3.2 测试点2： 当有2个策略同时应用到一个pod时的情况？(针对的是pod的标签是role=web) 
    ```
    # 第一步： 测试bookstore是否可以访问frontend里的Nginx服务？ 
    [root@master day02]# kubectl exec -it bookstore-7466f96497-7zt55 sh 
    # wget 192.168.1.38
    --2018-10-11 09:38:07--  http://192.168.1.38/
    Connecting to 192.168.1.38:80... ^C
    # exit
    command terminated with exit code 130
    -------------------------------------------------------

    # 第二步： 测试frontend是否可以访问bookstore里的Nginx服务? 
    [root@master day02]# kubectl exec -it frontend-86dcfb45d4-cv29q sh
    # wget 192.168.2.58
    --2018-10-11 09:39:00--  http://192.168.2.58/
    Connecting to 192.168.2.58:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html'

    index.html                                100%[==================================================================================>]     612  --.-KB/s    in

    2018-10-11 09:39:00 (89.7 MB/s) - 'index.html' saved [612/612]

    # 
    ```

    3.3 测试点3： 测试其他命名空间里的pod是否可以访问default命名空间里的bookstore里的Nginx服务?   
    ```
    # 第一步： 测试coffeeshop是否可以访问bookstore里的Nginx服务？ 
    [root@master ~]# kubectl exec -it coffeeshop-cf9b57f64-ppm79  sh -n policy-demo
    # wget 192.168.2.58
    --2018-10-11 11:08:34--  http://192.168.2.58/
    Connecting to 192.168.2.58:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html'

    index.html                                100%[==================================================================================>]     612  --.-KB/s    in 0s      

    2018-10-11 11:08:34 (126 MB/s) - 'index.html' saved [612/612]

    # 
    -------------------------------------------------------

    # 第二步： 测试appleshop是否可以访问bookstore里的Nginx服务? 
    [root@master ~]# kubectl exec -it appleshop-5f9c7bf6cb-9fzx5 sh -n policy2
    # wget 192.168.2.58
    --2018-10-11 11:07:08--  http://192.168.2.58/
    Connecting to 192.168.2.58:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html'

    index.html                                100%[==================================================================================>]     612  --.-KB/s    in 0s      

    2018-10-11 11:07:08 (79.3 MB/s) - 'index.html' saved [612/612]

    # exit

    -------------------------------------------------------

    # 第三步： 测试appleshop是否可以访问frontend里的Nginx服务? 
    [root@master ~]# kubectl exec -it appleshop-5f9c7bf6cb-9fzx5 sh -n policy2
    # wget 192.168.1.38
    --2018-10-11 11:09:59--  http://192.168.1.38/
    Connecting to 192.168.1.38:80... 

    ```  



## 5、清理工作  
    kubectl delete networkpolicy web-allow-all-namespaces default-deny-all
    kubectl delete deployment bookstore frontend
    kubectl delete deployment coffeeshop -n policy-demo 
    kubectl delete deployment appleshop -n policy2






