# kubernetes Network Policies 之 ALLOW traffic from some pods in another namespace    

## 1、规则描述：  
希望达到的效果：  
&ensp;&ensp;&ensp;&ensp;只有命名空间里带有team=operations标签，pod里带有type=monitoring标签的pod才能访问制定的pod

如下图所示：  
![](https://note.youdao.com/yws/public/resource/a431131093aff717feb4fd8bc812c5f5/xmlnote/80A1A3210AD64771A9B5976EDB73D874/20419)    

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
    kubectl label namespace/prod team=operations
    kubectl label namespace/dev team=write

## 4、创建相应的Pod  
1. 在default命名空间下, 创建一个带有app=nginx标签的pod,名称是bookstore  
    ```
    kubectl run bookstore --image=mybusybox --labels name=bookstore,app=web --image-pull-policy=IfNotPresent
    ```
2. 在prod命名空间下, 创建一个带有Nginx服务的pod,名称是appleshop  
    ```
    kubectl run appleshop -n prod --image=mybusybox --labels app=appleshop,type=monitoring  --image-pull-policy=IfNotPresent
    ``` 
3. 在prod命名空间下, 创建一个带有Nginx服务的pod,名称是coffeeshop   
    ```
    kubectl run coffeeshop -n prod --image=mybusybox --labels app=coffeeshop,type=watching   --image-pull-policy=IfNotPresent
    ```  
4. 在dev命名空间下, 创建一个带有Nginx服务的pod,名称是gameshop   
    ```
    kubectl run gameshop -n dev --image=mybusybox --labels app=gameshop,type=monitoring  --image-pull-policy=IfNotPresent
    ```     
5. 查看pod创建情况  
    ```
    [root@master day03]# kubectl get pod -owide
    NAME                         READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE
    bookstore-759cf5d788-p8mvb   1/1     Running   0          6m26s   192.168.1.42   slave1   <none>
    [root@master day03]# kubectl get pod -owide -n dev
    NAME                       READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE
    gameshop-d64d5964f-q7mwh   1/1     Running   0          6m16s   192.168.1.43   slave1   <none>
    [root@master day03]# kubectl get pod -owide -n prod
    NAME                          READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE
    appleshop-58858fcb5c-bkg4b    1/1     Running   0          6m28s   192.168.2.63   slave2   <none>
    coffeeshop-74d86bc754-cnmzw   1/1     Running   0          13s     192.168.2.64   slave2   <none>
    [root@master day03]# 

    ```


## 5、开始测试  
### 5.1、未使用策略前的测试
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;默认未使用任何策略时，可以互相访问的，不再具体测试了
    
### 5.2、开始使用策略的测试  
1. 创建NetworkPolicy类型的yaml文件，如 web-allow-all-ns-monitoring.yaml 
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata: 
      name: web-allow-all-ns-monitoring
    spec: 
      podSelector: 
        matchLabels: 
          app: web
      ingress: 
      - from: 
        - namespaceSelector: 
            matchLabels: 
              team: operations  
          podSelector: 
            matchLabels:  
              type: monitoring     
    ```   
    ![](https://note.youdao.com/yws/public/resource/a431131093aff717feb4fd8bc812c5f5/xmlnote/950998324F894E14ACF2A5C056C25B5D/20417)  
2. 使其生效  
    ```
    kubectl create -f web-allow-all-ns-monitoring.yaml
    ```
3. 测试  

    3.1 测试点1： 命名空间为dev的pod是否可以访问default命名空间下pod的标签为app=web的pod里的Nginx服务？  
    ```
    [root@master day03]# kubectl exec -it gameshop-d64d5964f-q7mwh sh -ndev
    # wget 192.168.1.42
    --2018-10-12 02:27:50--  http://192.168.1.42/
    Connecting to 192.168.1.42:80... ^C
    # exit
    command terminated with exit code 130

    ```  
    说明，访问不了，因为，此pod所处的命名空间里的标签不符合要求，没有team=operations, 如果想访问的话，可以添加标签就可以了。 

    3.2 测试点2： 命名空间为prod的pod是否可以访问default命名空间下pod的标签为app=web的pod里的Nginx服务？  
    ```  
    # 第一步： 进入带有type=monitoring标签的pod
    [root@master day03]# kubectl exec -it appleshop-58858fcb5c-bkg4b sh -n prod
    # wget 192.168.1.42
    --2018-10-12 02:35:01--  http://192.168.1.42/
    Connecting to 192.168.1.42:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 612 [text/html]
    Saving to: 'index.html'

    index.html                                100%[==================================================================================>]     612  --.-KB/s    in 0.01s   

    2018-10-12 02:35:01 (50.6 KB/s) - 'index.html' saved [612/612]

    # 

    # 第二步： 进入没有type=monitoring标签的pod
    [root@master day03]# kubectl exec -it coffeeshop-74d86bc754-cnmzw sh -nprod
    # wget 192.168.1.42
    --2018-10-12 02:36:12--  http://192.168.1.42/
    Connecting to 192.168.1.42:80... ^C
    # 


    ```  
    说明，网络策略是生效的

## 5、清理工作  
    kubectl delete networkpolicy  web-allow-all-ns-monitoring
    kubectl delete deployment bookstore
    kubectl delete deployment gameshop -n dev
    kubectl delete deployment appleshop coffeeshop -n prod






