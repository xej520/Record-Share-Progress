# kubernetes Network Policies 之  ALLOW traffic only to a port of an application

## 1、规则描述：  
希望达到的效果：  
如下图所示：  
![](https://note.youdao.com/yws/public/resource/a431131093aff717feb4fd8bc812c5f5/xmlnote/71F48A969128478F9DA15E29309E9C7E/20433)    

__使用场景:__  
&ensp;&ensp;&ensp;&ensp;允许监视系统通过查询应用程序的诊断端口来收集度量标准，而无需授予其访问应用程序其余部分的权限


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
4. 下载ahmet/app-on-two-ports镜像 
    ```
    docker pull ahmet/app-on-two-ports
    ```    
 
## 3、创建测试用的命名空间，prod
    
    # 创建 命名空间
    kubectl create namespace prod 

## 4、创建相应的Pod、SVC  
1. 在default命名空间下, 创建一个deployment, 名称是apiserver  
    ```
    kubectl run apiserver --image=ahmet/app-on-two-ports --labels app=apiserver --image-pull-policy=IfNotPresent   

    kubectl create service clusterip apiserver --tcp 8001:8000 --tcp 5001:5000 
    ```
2. 在default命名空间下, 创建一个deployment, 名称是gameshop  
    ```
    kubectl run gameshop --image=mybusybox --labels app=gameshop,role=monitoring  --image-pull-policy=IfNotPresent
    ``` 
3. 在prod命名空间下, 创建一个deployment, 名称是appleshop   
    ```
    kubectl run appleshop -n prod --image=mybusybox --labels app=appleshop,role=monitoring --image-pull-policy=IfNotPresent
    ```  
    
4. 查看pod、svc创建情况  
    ```
    [root@master day03]# kubectl get pod -owide
    NAME                         READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE
    apiserver-58c754bc85-x97nt   1/1     Running   0          6m10s   192.168.2.66   slave2   <none>
    gameshop-7c5cb65b6f-5ntxx    1/1     Running   0          54s     192.168.1.47   slave1   <none>
    [root@master day03]# kubectl get pod -owide -nprod
    NAME                        READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE
    appleshop-bbc7c8769-68khc   1/1     Running   0          11s   192.168.1.48   slave1   <none>
    [root@master day03]# kubectl get svc
    NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
    apiserver    ClusterIP   10.96.20.30   <none>        8001/TCP,5001/TCP   6m5s
    kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP             3d7h
    [root@master day03]# 

    ```


## 5、开始测试  
### 5.1、未使用策略前的测试
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;默认未使用任何策略时，可以互相访问的，不再具体测试了
    
### 5.2、开始使用策略的测试  
 
1. 创建NetworkPolicy类型的yaml文件，如 api-allow-5000.yaml 
    ```
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata: 
      name: api-allow-5000
    spec: 
      podSelector: 
        matchLabels: 
          app: apiserver
      ingress: 
      - ports: 
        - port: 5000 
        from: 
        - podSelector: 
            matchLabels: 
              role: monitoring   
    ```   
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/7649F8FF6D1141CEAA6AFE1F0BEC3062/20467)  
2. 使其生效  
    ``` 
    kubectl create -f api-allow-5000.yaml
    ```
3. 测试  

    3.1 测试点1： gameshop 是否可以访问5000端口，8000端口？  
    ```
    [root@master day03]# kubectl exec -it gameshop-7c5cb65b6f-5ntxx sh 
    # wget -O- --timeout=1 http://apiserver:8001
    --2018-10-12 07:51:23--  http://apiserver:8001/
    Resolving apiserver (apiserver)... 10.96.20.30
    Connecting to apiserver (apiserver)|10.96.20.30|:8001... failed: Connection timed out.
    Retrying.

    --2018-10-12 07:51:25--  (try: 2)  http://apiserver:8001/
    Connecting to apiserver (apiserver)|10.96.20.30|:8001... failed: Connection timed out.
    Retrying.

    --2018-10-12 07:51:28--  (try: 3)  http://apiserver:8001/
    Connecting to apiserver (apiserver)|10.96.20.30|:8001... failed: Connection timed out.
    Retrying.

    --2018-10-12 07:51:32--  (try: 4)  http://apiserver:8001/
    Connecting to apiserver (apiserver)|10.96.20.30|:8001... failed: Connection timed out.
    Retrying.

    ^C
    # wget -O- --timeout=1 http://apiserver:5001/metrics
    --2018-10-12 07:52:06--  http://apiserver:5001/metrics
    Resolving apiserver (apiserver)... 10.96.20.30
    Connecting to apiserver (apiserver)|10.96.20.30|:5001... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 42 [text/plain]
    Saving to: 'STDOUT'

    -                                           0%[                                                                                   ]       0  --.-KB/s               http.requests=1
    go.goroutines=5
    go.cpus=2
    -                                         100%[==================================================================================>]      42  --.-KB/s    in 0s      

    2018-10-12 07:52:06 (10.1 MB/s) - written to stdout [42/42]

    # exit
    [root@master day03]# 

    ```  
    说明，同一个命名空间下，带有指定标签role=monitoring的pod，可以访问5000端口，8000端口禁止
    3.2 测试点2： appleshop 是否可以访问5000端口，8000端口？  
    ```
    [root@master day03]# kubectl exec -it appleshop-bbc7c8769-68khc sh -nprod
    # wget -O- --timeout=1 -t 0 http://apiserver:8001  
    --2018-10-12 07:56:22--  http://apiserver:8001/
    Resolving apiserver (apiserver)... failed: Connection timed out.
    wget: unable to resolve host address 'apiserver'
    # wget -O- --timeout=1 -t 0 http://apiserver:5001/metrics
    --2018-10-12 07:56:59--  http://apiserver:5001/metrics
    Resolving apiserver (apiserver)... failed: Connection timed out.
    wget: unable to resolve host address 'apiserver'
    # wget 192.168.2.66
    --2018-10-12 07:57:24--  http://192.168.2.66/
    Connecting to 192.168.2.66:80... ^C
    # wget http://192.168.2.66:5001/metrics
    --2018-10-12 07:58:12--  http://192.168.2.66:5001/metrics
    Connecting to 192.168.2.66:5001... ^C
    # wget http://10.96.20.30:5001/metrics
    --2018-10-12 07:58:57--  http://10.96.20.30:5001/metrics
    Connecting to 10.96.20.30:5001... failed: Connection timed out.
    Retrying.

    --2018-10-12 08:01:05--  (try: 2)  http://10.96.20.30:5001/metrics
    Connecting to 10.96.20.30:5001... ^C
    # 

    ```  
    现象是：  
    pod的标签是app=apisever， 只能接收这个pod所在的命名空间下的pod的访问，其他命名空间不接收

## 5、清理工作  
    kubectl delete networkpolicy api-allow-5000
    kubectl delete deployment gameshop apiserver
    kubectl delete deployment appleshop -n prod 
    kubectl delete svc apiserver

## 6、网络策略规则说明
- 如果没有显示的声明端口的话，默认是全部端口开放的  
- 端口可以是数值 
- 端口也可以是名称  
- 




