# kubernetes Network Policies 之 只允许设有某些标签的Pod可以访问指定的pod  
    
## 1、规则描述：  
如一个apiserver服务，部署在带有标签app=bookstore, role=api的pod里 
希望达到的效果：  
1. 只能带有app=bookstore标签的pod访问bookstorePod
2. 其他pod不允许访问此pod

如下图所示：  
![](https://note.youdao.com/yws/public/resource/a431131093aff717feb4fd8bc812c5f5/xmlnote/FCEE75C269E64EFCB68BB5B8A78FE904/20398)   

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
    kubectl run bookstore --image=mybusybox --labels app=bookstore,role=api --expose --port 80 --image-pull-policy=IfNotPresent
    ```  
2. 在default命名空间下, 创建一个带有Nginx服务的pod,名称是frontend  
    ```
    kubectl run frontend --image=mybusybox --labels app=bookstore,role=frontend  --image-pull-policy=IfNotPresent
    ```  
3. 在policy-demo命名空间下, 创建一个带有Nginx服务的pod,名称是coffeeshop   
    ```
    kubectl run coffeeshop -n policy-demo --image=mybusybox --labels app=coffeeshop,role=api  --image-pull-policy=IfNotPresent
    ```  
4. 查看pod创建情况  
    ```
    [root@master day02]# kubectl get pod  -owide
    NAME                         READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    bookstore-6dc84486c7-wljn8   1/1     Running   0          65s   192.168.1.32   slave1   <none>
    frontend-6b855d4b8d-c2nft    1/1     Running   0          59s   192.168.2.51   slave2   <none>
    [root@master day02]# kubectl get pod  -owide -npolicy-demo
    NAME                         READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    coffeeshop-95cd499f8-d2dcn   1/1     Running   0          12s   192.168.2.52   slave2   <none>
    [root@master day02]# 

    ```


## 4、开始测试  
### 4.1、未使用策略前的测试  
    默认未使用任何策略时，可以互相访问的，不再具体测试了
### 4.2、开始使用策略的测试  
1. 创建NetworkPolicy类型的yaml文件，如 api-allow.yaml 
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata: 
      name: api-allow
    spec: 
      podSelector: 
        matchLabels: 
          app:  bookstore
          role: api
      ingress:  
      - from: 
        - podSelector: 
            matchLabels: 
              app: bookstore
    ```  
    这个policy只针对标签是app=bookstore，role=api的pod生效 ，限制这个pod的入站流量  
    这个策略，并未涉足egress, 也就是说，出站流量是允许的  

2. 使其生效  
    ```
    kubectl create -f api-allow.yaml 
    ```
3. 测试bookstore的入站规则(只允许带有指定标签的pod进行访问？)
    ```
    # 第一步： 进入frontend pod，查看是否可以访问bookstore里的Nginx服务？ 
    [root@master day02]# kubectl exec -it frontend-6b855d4b8d-c2nft sh
    # wget --timeout=1 http://bookstore -O-
    --2018-10-11 02:52:30--  http://bookstore/
    Resolving bookstore (bookstore)... 10.96.172.80
    Connecting to bookstore (bookstore)|10.96.172.80|:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'STDOUT'

    -                                        0%[                                                                           ]       0  --.-KB/s               <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
        body {
            width: 35em;
            margin: 0 auto;
            font-family: Tahoma, Verdana, Arial, sans-serif;
        }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
    -                                      100%[==========================================================================>]     612  --.-KB/s    in 0s      

    2018-10-11 02:52:30 (98.5 MB/s) - written to stdout [612/612]

    # 第二步： 进入coffeeshop pod，查看是否可以访问bookstore里的服务  
    [root@master day02]# kubectl exec -it coffeeshop-95cd499f8-d2dcn  sh -npolicy-demo
    # wget -O- --timeout=1 http://bookstore 
    --2018-10-11 02:55:04--  http://bookstore/
    Resolving bookstore (bookstore)... failed: Name or service not known.
    wget: unable to resolve host address 'bookstore'
    # wget 192.168.1.32
    --2018-10-11 02:55:51--  http://192.168.1.32/
    Connecting to 192.168.1.32:80... failed: Connection timed out.
    Retrying.

    --2018-10-11 02:57:59--  (try: 2)  http://192.168.1.32/
    Connecting to 192.168.1.32:80... ^C
    # 

    ```
    说明，api-allow策略起作用了。 只允许某些pod进行访问  

## 5、清理工作  
    kubectl delete networkpolicy api-allow
    kubectl delete svc bookstore
    kubectl delete deployment bookstore frontend
    kubectl delete deployment coffeeshop -n policy-demo 








