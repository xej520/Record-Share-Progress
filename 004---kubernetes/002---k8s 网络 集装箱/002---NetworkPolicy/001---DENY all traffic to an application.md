# kubernetes Network Policies 之 DENY all traffic to an application  
## 1、规则描述：  
1. 针对具有某些标签的Pod，并非所有pod
2. 这些pod的数据包允许出站 
3. 不允许访问这些pod  
如下图所示：  
![](https://note.youdao.com/yws/public/resource/a431131093aff717feb4fd8bc812c5f5/xmlnote/5619698C34964347A8794C79E26A0CDA/20392)  

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
docker build -t mybusybox .  

## 3、创建相应的Pod  
1. 在default命名空间下, 创建一个带有app=nginx标签的pod,名称是nginx1 
    ```
    kubectl run -n default nginx1 --replicas=1 --image=mybusybox --labels app=nginx --image-pull-policy=IfNotPresent
    ```    
2. 在default命名空间下, 创建一个带有Nginx服务的pod,名称是nginx2  
    ```
    kubectl run -n default nginx2 --replicas=1 --image=mybusybox --image-pull-policy=IfNotPresent
    ```
3. 在policy-demo命名空间下, 创建一个带有Nginx服务的pod,名称是nginx3  
    ```
    kubectl run -n policy-demo nginx3 --replicas=1 --image=mybusybox --image-pull-policy=IfNotPresent
    ```
4. 查看pod创建情况  
    ```
    [root@master ~]# kubectl get pod -owide
    NAME                      READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE
    nginx1-5f97c5d657-hjvtg   1/1     Running   0          30s     192.168.2.50   slave2   <none>
    nginx2-59cbdd9c66-7kht8   1/1     Running   0          8m19s   192.168.2.49   slave2   <none>
    [root@master ~]# kubectl get pod -npolicy-demo -owide 
    NAME                      READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE
    nginx3-864d548d66-c727s   1/1     Running   0          8m23s   192.168.1.31   slave1   <none>
    [root@master ~]# 
    ```

## 4、开始测试  
### 4.1、未使用策略前的测试  
    默认未使用任何策略时，可以互相访问的，不再具体测试了
### 4.2、开始使用策略的测试  
1. 创建NetworkPolicy类型的yaml文件，如 nginx-deny-all.yaml 
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata: 
    name: nginx-deny-all
    spec: 
    podSelector: 
        matchLabels: 
        app: nginx 
    ingress: []
    ```  

2. 使其生效  
    ```
    kubectl create -f nginx-deny-all.yaml
    ```
3. 测试nginx1的出站规则
    ```
    #进入nginx1，测试出站流量，是否可以访问nginx2, nginx3的nginx服务 
    [root@master day02]# kubectl exec -it nginx1-5f97c5d657-hjvtg sh
    # wget 192.168.1.31
    --2018-10-11 01:30:17--  http://192.168.1.31/
    Connecting to 192.168.1.31:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html'

    index.html                                100%[==================================================================================>]     612  --.-KB/s    in 0.01s   

    2018-10-11 01:30:17 (48.5 KB/s) - 'index.html' saved [612/612]

    # wget 192.168.2.49
    --2018-10-11 01:30:34--  http://192.168.2.49/
    Connecting to 192.168.2.49:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html.1'

    index.html.1                              100%[==================================================================================>]     612  --.-KB/s    in 0.01s   

    2018-10-11 01:30:34 (47.2 KB/s) - 'index.html.1' saved [612/612]

    # exit

    ```  
    说明：出站流量没有限制
4. 测试nginx1的入站规则
    ```
    #进入nginx2, 测试是否可以访问nginx1的ngninx服务  
    [root@master day02]# kubectl exec -it nginx2-59cbdd9c66-7kht8 sh
    # wget --timeout=1 192.168.2.50 -O -
    --2018-10-11 01:38:43--  http://192.168.2.50/
    Connecting to 192.168.2.50:80... failed: Connection timed out.
    Retrying.

    --2018-10-11 01:38:45--  (try: 2)  http://192.168.2.50/
    Connecting to 192.168.2.50:80... failed: Connection timed out.
    Retrying.

    #进入nginx3, 测试是否可以访问nginx1的ngninx服务 
    [root@master day02]# kubectl exec -it nginx3-864d548d66-c727s sh -npolicy-demo
    # wget --timeout=1 192.168.2.50 -O-      
    --2018-10-11 01:45:38--  http://192.168.2.50/
    Connecting to 192.168.2.50:80... failed: Connection timed out.
    Retrying.

    --2018-10-11 01:45:40--  (try: 2)  http://192.168.2.50/
    Connecting to 192.168.2.50:80... failed: Connection timed out.
    Retrying.

    ^C
    # 

    ```  
    说明，对入站流量进行了限制，不能访问nginx1

## 5、清理工作  
    kubectl delete networkpolicy nginx-deny-all
    kubectl delete deployment nginx1 nginx2 
    kubectl delete deployment nginx3 -n policy-demo 

## 6、注意：  
    需要总结policyTypes属性的作用，设置与不设置的区别 





