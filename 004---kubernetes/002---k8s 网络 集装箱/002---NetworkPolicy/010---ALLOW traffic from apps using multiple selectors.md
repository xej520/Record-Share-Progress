# kubernetes Network Policies 之  ALLOW traffic from apps using multiple selectors

## 1、规则描述：  
希望达到的效果：  
如下图所示：  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/51010B6D53694F178A677C092E61C467/20439)    

__使用场景:__  
&ensp;&ensp;&ensp;&ensp;比方说，有一个数据中心，可以提供给多个微服务使用。  


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
4. 下载redis镜像 
    ```
    docker pull reids:4
    ```  
## 3、创建测试用的命名空间，prod
    
    # 创建 命名空间
    kubectl create namespace prod 
    kubectl create namespace dev

## 4、创建相应的Pod、SVC  
1. 在default命名空间下, 创建一个deployment, 名称是redis  
    ```
    kubectl run redis --image=redis:4 --port 6379 --expose --labels="app=bookstore,role=db" --image-pull-policy=IfNotPresent   
    ```
2. 在default命名空间下, 创建一个deployment, 名称是harbor  
    ```
    kubectl run harbor --image=mybusybox --labels="app=inventory,role=watch"  --image-pull-policy=IfNotPresent
    ``` 
3. 在default命名空间下，创建两个deployment, 名称分别是spark，hadoop 
    ```
    kubectl run spark --image=mybusybox --labels="app=bookstore,role=api" --image-pull-policy=IfNotPresent
    kubectl run hadoop --image=mybusybox --labels="app=bookstore,role=search" --image-pull-policy=IfNotPresent
    ``` 
4. 在prod命名空间下, 创建一个deployment, 名称是zk   
    ```
    kubectl run zk -n prod --image=mybusybox --labels="app=inventory,role=web" --image-pull-policy=IfNotPresent
    ```    
5. 查看pod、svc创建情况  


## 5、开始测试  
### 5.1、未使用策略前的测试
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;默认未使用任何策略时，可以互相访问的，不再具体测试了
    
### 5.2、开始使用策略的测试  
 
1. 创建NetworkPolicy类型的yaml文件，如 redis-allow-services.yaml 
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata: 
      name: redis-allow-services
    spec: 
      podSelector: 
        matchLabels: 
          app: bookstore
          role: db
      ingress:         
      - from: 
        - podSelector: 
            matchLabels: 
              app: inventory
              role: web   
        - podSelector: 
            matchLabels: 
              app: bookstore
              role: api 
        - podSelector: 
            matchLabels: 
              app: bookstore
              role: search 

    ```   
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/4350210223164AD287E379C449C73DC0/20442)  
2. 使其生效  
    ``` 
    kubectl create -f redis-allow-services.yaml
    ```
3. 测试  

    3.1 测试点1： default命名空间下符合标签的pod是否可以访问redis pod？ 
    ``` 
    [root@master day03]# kubectl exec -it hadoop-58c7f5b7c9-zl9sn sh 
    # wget redis:6379
    --2018-10-13 10:59:02--  http://redis:6379/
    Resolving redis (redis)... 10.104.56.22
    Connecting to redis (redis)|10.104.56.22|:6379... connected.
    HTTP request sent, awaiting response... 200 No headers, assuming HTTP/0.9
    Length: unspecified
    Saving to: 'index.html'

    index.html                                    [ <=>                                                                               ]     290  --.-KB/s    in 0s      

    2018-10-13 10:59:02 (32.9 MB/s) - 'index.html' saved [290]

    # exit
    [root@master day03]# kubectl exec -it spark-5fd4758b67-xv6zn sh 
    # wget redis:6379
    --2018-10-13 10:59:30--  http://redis:6379/
    Resolving redis (redis)... 10.104.56.22
    Connecting to redis (redis)|10.104.56.22|:6379... connected.
    HTTP request sent, awaiting response... 200 No headers, assuming HTTP/0.9
    Length: unspecified
    Saving to: 'index.html'

    index.html                                    [ <=>                                                                               ]     290  --.-KB/s    in 0s      

    2018-10-13 10:59:30 (55.1 MB/s) - 'index.html' saved [290]

    # exit
    [root@master day03]# kubectl exec -it harbor-7d549c5bd9-mw74s  sh 
    # wget redis:6379
    --2018-10-13 11:00:34--  http://redis:6379/
    Resolving redis (redis)... 10.104.56.22
    Connecting to redis (redis)|10.104.56.22|:6379... failed: Connection timed out.
    Retrying.

    ```  
    同一个命名空间下，只有符合标签的才能访问redis   
    
    3.2 测试点2： prod命名空间下的zk是否可以访问redis pod？  
    ```
    [root@master day03]# kubectl exec -it zk-5bd48b7865-g8lkp sh -nprod
    # wget redis:6379
    --2018-10-13 11:06:52--  http://redis:6379/
    Resolving redis (redis)... ^C
    # wget 192.168.1.43:6379
    --2018-10-13 11:07:15--  http://192.168.1.43:6379/
    Connecting to 192.168.1.43:6379... ^C
    # wget 10.104.56.22:6379
    --2018-10-13 11:07:30--  http://10.104.56.22:6379/
    Connecting to 10.104.56.22:6379... ^C
    # 

    ```  
    因为，该网络策略只允许同一个命名空间下的pod访问，因此访问失败  

## 5、清理工作  
    kubectl delete networkpolicy redis-allow-services
    kubectl delete deployment redis harbor spark hadoop
    kubectl delete deployment zk -n prod 
    kubectl delete svc redis







