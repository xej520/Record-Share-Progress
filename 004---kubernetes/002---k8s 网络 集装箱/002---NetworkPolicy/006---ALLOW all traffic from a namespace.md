# kubernetes Network Policies 之 ALLOW all traffic from a namespace   
参考文献：  
https://github.com/xej520/kubernetes-network-policy-recipes/blob/master/06-allow-traffic-from-a-namespace.md  

## 1、规则描述：  
希望达到的效果：  
1. 只允许标签为purpose=production命名空间下的所有pod访问default命名空间下标签为app=web的pod里的nginx服务

如下图所示：  
![](https://note.youdao.com/yws/public/resource/a431131093aff717feb4fd8bc812c5f5/xmlnote/92105B523D394D2299957E772DC47E3B/20414)    

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
## 3、创建测试用的命名空间，prod, dev
    
    # 创建 命名空间
    kubectl create namespace prod 
    kubectl create namespace dev 

    # 创建测试用的标签
    kubectl label namespace/prod purpose=production
    kubectl label namespace/dev purpose=testing 

## 4、创建相应的Pod  
1. 在default命名空间下, 创建一个带有app=nginx标签的pod,名称是bookstore  
    ```
    kubectl run bookstore --image=mybusybox --labels name=bookstore,app=web --image-pull-policy=IfNotPresent
    ```
2. 在default命名空间下, 创建一个带有Nginx服务的pod,名称是gamestore  
    ```
    kubectl run gamestore --image=mybusybox --labels app=web,name=gamestore  --image-pull-policy=IfNotPresent
    ``` 
3. 在prod命名空间下, 创建一个带有Nginx服务的pod,名称是frontend   
    ```
    kubectl run frontend -n prod --image=mybusybox --labels app=frontend  --image-pull-policy=IfNotPresent
    ```  
4. 在dev命名空间下, 创建一个带有Nginx服务的pod,名称是coffeeshop   
    ```
    kubectl run coffeeshop -n dev --image=mybusybox --labels app=coffeeshop  --image-pull-policy=IfNotPresent
    ```     
5. 查看pod创建情况  
    ```
    [root@master ~]# kubectl get pod -owide
    NAME                         READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE
    bookstore-759cf5d788-kvr5z   1/1     Running   0          7m15s   192.168.1.40   slave1   <none>
    gamestore-8f4d4c576-w7klx    1/1     Running   0          10s     192.168.2.61   slave2   <none>
    [root@master ~]# kubectl get pod -owide -n prod
    NAME                        READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE
    frontend-86dcfb45d4-8b5xn   1/1     Running   0          5m55s   192.168.1.41   slave1   <none>
    [root@master ~]# kubectl get pod -owide -n dev
    NAME                         READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE
    coffeeshop-cf9b57f64-lhg7t   1/1     Running   0          5m52s   192.168.2.60   slave2   <none>
    [root@master ~]# 

    ```


## 5、开始测试  
### 5.1、未使用策略前的测试
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;默认未使用任何策略时，可以互相访问的，不再具体测试了
    
### 5.2、开始使用策略的测试  
1. 创建NetworkPolicy类型的yaml文件，如 web-allow-prod.yaml 
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata: 
      name: web-allow-prod
    spec: 
      podSelector: 
        matchLabels: 
          app: web
      ingress: 
      - from: 
        - namespaceSelector: 
            matchLabels: 
              purpose: production
    ```   

2. 使其生效  
    ```
    kubectl create -f web-allow-prod.yaml
    ```
3. 测试  

    3.1 测试点1： 命名空间为dev的pod是否可以访问default命名空间下pod的标签为app=web的pod里的Nginx服务？  
    ```
    [root@master day03]# kubectl exec -it coffeeshop-cf9b57f64-lhg7t sh -ndev
    # wget 192.168.1.40
    --2018-10-12 01:05:18--  http://192.168.1.40/
    Connecting to 192.168.1.40:80... ^C
    # 

    ```

    3.2 测试点2： 命名空间为prod的pod是否可以访问default命名空间下pod的标签为app=web的pod里的Nginx服务？  
    ```
    [root@master day03]# kubectl exec -it frontend-86dcfb45d4-8b5xn sh -n prod
    # wget 192.168.2.61
    --2018-10-12 01:06:22--  http://192.168.2.61/
    Connecting to 192.168.2.61:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html'

    index.html                                100%[==================================================================================>]     612  --.-KB/s    in 0.01s   

    2018-10-12 01:06:22 (46.3 KB/s) - 'index.html' saved [612/612]

    # 

    ```

    3.3 测试点3： gamestore的pod是否可以访问标签为app=web的pod里的Nginx服务呢？(也就是同一个命名空间内是否可以访问？)  
    ```
    [root@master day03]# kubectl exec -it gamestore-8f4d4c576-w7klx sh 
    # wget 192.168.1.40
    --2018-10-12 01:07:34--  http://192.168.1.40/
    Connecting to 192.168.1.40:80... ^C
    # 
    ```  
    说明：  
    网络策略web-allow-prod中pod的入站规则：  
    &ensp;&ensp;&ensp;&ensp;只接收命名空间带有purpose=production标签的pod  
    &ensp;&ensp;&ensp;&ensp;default命名空间没有标签purpose=production，因此default命名空间中的pod也不能访问此Nginx服务 

    3.4 测试点4： 网络策略web-allow-prod中没有显示的声明该策略的命名空间，那么这个web-allow-prod策略的使用范围是所有命名空间？还是default命名空间？  
    步骤：  
    &ensp;&ensp;&ensp;&ensp;在dev命名空间中，创建一个pod，带有标签app=web， 然后使用prod命名空间下的frontend 去访问刚创建的pod里的Nginx服务? 查看是否能访问？  如下图所示：  
    ![](https://note.youdao.com/yws/public/resource/a431131093aff717feb4fd8bc812c5f5/xmlnote/AE33A1A18B6643BA96172B760C35133D/20411)  
        
    ```
    # 第一步： 创建pod
    kubectl run appleshop --image=mybusybox --labels name=appleshop --labels  app=web --image-pull-policy=IfNotPresent -n dev  

    # 第二步： 查看pod的创建情况   
    [root@master day03]# kubectl get pod -n dev -owide
    NAME                         READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    appleshop-68c8674646-997dc   1/1     Running   0          12s   192.168.2.62   slave2   <none>
    coffeeshop-cf9b57f64-lhg7t   1/1     Running   0          28m   192.168.2.60   slave2   <none>
    
    # 第三步： 测试是否可以从prod命名空间中访问此服务  
    [root@master day03]# kubectl exec -it frontend-86dcfb45d4-8b5xn sh -n prod
    # wget 192.168.2.62
    --2018-10-12 01:27:55--  http://192.168.2.62/
    Connecting to 192.168.2.62:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html.1'

    index.html.1                              100%[==================================================================================>]     612  --.-KB/s    in 0s      

    2018-10-12 01:27:55 (82.1 MB/s) - 'index.html.1' saved [612/612]

    # 

    ```  
    总结：  
    &ensp;&ensp;&ensp;&ensp;可以访问Nginx服务，  
    &ensp;&ensp;&ensp;&ensp;说明：   
    &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;web-allow-prod.yaml文件中如果没有显示的声明此策略的命名空间的话，默认就是全局生效的  




## 5、清理工作  
    kubectl delete networkpolicy  web-allow-prod
    kubectl delete deployment bookstore gamestore
    kubectl delete deployment coffeeshop appleshop -n dev 
    kubectl delete deployment frontend -n prod






