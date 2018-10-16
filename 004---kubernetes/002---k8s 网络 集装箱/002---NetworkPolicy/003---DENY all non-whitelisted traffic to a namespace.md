# kubernetes Network Policies 之 DENY all non-whitelisted traffic to a namespace    
参考文献：  
https://github.com/xej520/kubernetes-network-policy-recipes/blob/master/03-deny-all-non-whitelisted-traffic-in-the-namespace.md  

## 1、规则描述：  
希望达到的效果：  
1. 命名空间default下的所有pod，入站流量全部关闭
2. 命名空间default下的所有pod，只能访问其他命名空间下的pod

如下图所示：  
![](https://note.youdao.com/yws/public/resource/a431131093aff717feb4fd8bc812c5f5/xmlnote/E7D69F06BA3841CB8E9B057B14E7EBE9/20400)   

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
4. 在policy-demo命名空间下, 创建一个带有Nginx服务的pod,名称是appleshop   
    ```
    kubectl run appleshop -n policy-demo --image=mybusybox --labels app=appleshop  --image-pull-policy=IfNotPresent
    ``` 

4. 查看pod创建情况  
    ```
    [root@master ~]# kubectl get pod -owide
    NAME                         READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    bookstore-6d5b4dc66d-gc8dk   1/1     Running   0          26s   192.168.1.34   slave1   <none>
    frontend-86dcfb45d4-lj9h7    1/1     Running   0          20s   192.168.2.53   slave2   <none>
    [root@master ~]# kubectl get pod -owide -npolicy-demo
    NAME                         READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    appleshop-5f9c7bf6cb-bpp8g   1/1     Running   0          15s   192.168.2.54   slave2   <none>
    coffeeshop-cf9b57f64-s6l7l   1/1     Running   0          20s   192.168.1.35   slave1   <none>
    [root@master ~]# 

    ```


## 4、开始测试  
### 4.1、未使用策略前的测试  
    默认未使用任何策略时，可以互相访问的，不再具体测试了
### 4.2、开始使用策略的测试  
1. 创建NetworkPolicy类型的yaml文件，如 default-deny-all.yaml 
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
    namespace: default，表示该策略使用的范围是default的命名空间  
    podSelector: {}, 表示是所有命名空间下的pod  
    ingress: [], 表示，入站规则为空，就是将所有进入的流量，全部丢掉  
    意思是说：  
    default命名空间下的pod的入站流量，全部关闭  
    或者说，所有命名空间下的pod都不能访问default命名空间下的pod

2. 使其生效  
    ```
    kubectl create -f default-deny-all.yaml
    ```
3. 测试  

    3.1 测试点1： 测试default命名空间下的bookstore pod的入站流量禁止访问  
    ```
    #第一步：进入frontend pod，访问bookstore服务
    [root@master day02]# kubectl exec -it frontend-86dcfb45d4-lj9h7 sh
    # wget 192.168.1.34
    --2018-10-11 06:26:59--  http://192.168.1.34/
    Connecting to 192.168.1.34:80... 
    
    #第二步：进入coffeeshop pod，访问bookstore服务
    [root@master day02]# kubectl exec -it coffeeshop-cf9b57f64-s6l7l sh -npolicy-demo
    # wget 192.168.1.34
    --2018-10-11 06:28:56--  http://192.168.1.34/
    Connecting to 192.168.1.34:80...failed: Connection timed out.
    Retrying. 

    ```  
    说明，入站流量已经禁止，所有命名空间下的pod都不能访问  

    3.2 测试点2： 测试default命名空间下的bookstore pod的出站流量，只能访问其他命名空间下的pod  
    ```
    [root@master day02]# kubectl exec -it bookstore-6d5b4dc66d-gc8dk sh
    # wget 192.168.2.53
    --2018-10-11 06:32:39--  http://192.168.2.53/
    Connecting to 192.168.2.53:80... ^C
    # wget 192.168.1.35
    --2018-10-11 06:33:02--  http://192.168.1.35/
    Connecting to 192.168.1.35:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html'

    index.html                                100%[==================================================================================>]     612  --.-KB/s    in 0s      

    2018-10-11 06:33:02 (57.0 MB/s) - 'index.html' saved [612/612]

    # 
    ```  
    很明显，同命名空间下pod是拒绝的， 可以访问其他命名空间，egress允许出站  

    3.3 测试点3： 测试policy-demo命名空间下的appleshop pod的入站流量是允许的，即不受default-deny-all网络策略的影响   
    ```
    [root@master day02]# kubectl exec -it coffeeshop-cf9b57f64-s6l7l sh -npolicy-demo
    # wget 192.168.2.54
    --2018-10-11 06:37:17--  http://192.168.2.54/
    Connecting to 192.168.2.54:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html'

    index.html                                100%[==================================================================================>]     612  --.-KB/s    in 0s      

    2018-10-11 06:37:17 (63.1 MB/s) - 'index.html' saved [612/612]

    # 

    ```
    说明，policy-demo命名空间下的pod的入站规则没有受到default-deny-all策略的影响，  
    其实3.2测试点已经测试过了，bookstore可以访问coffeeshop里的Nginx服务，就说明此问题了。 


## 5、清理工作  
    kubectl delete networkpolicy default-deny-all
    kubectl delete deployment bookstore frontend
    kubectl delete deployment coffeeshop appleshop -n policy-demo 

## 6、注意： 
![](https://note.youdao.com/yws/public/resource/a431131093aff717feb4fd8bc812c5f5/xmlnote/C5AAE2B18E0E4C848D38307B259D9B34/20402)  
如果没有声明namespace的话，默认是default命名空间  





