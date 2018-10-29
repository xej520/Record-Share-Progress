本文以DaemonSet方式启动nginx-ingress-controller   

上一篇文章  
https://www.jianshu.com/p/e30b06906b77  
是用Deployment方式启动 nginx-ingress-controller 

参考文献：  
http://blog.51cto.com/superpcm/2095395  
http://blog.51cto.com/devingeng/2149377  
https://blog.csdn.net/ygqygq2/article/details/82017617  

# 一、准备工作  
## 1.1 下载部署的yaml文件  
```
wget  https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```

## 1.2 更新yaml文件 
1. 更新镜像地址  
&ensp;&ensp;替换成自己的镜像地址：   
    ```
    sed -i 's#k8s.gcr.io/defaultbackend-amd64#registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/defaultbackend-amd64#g' mandatory.yaml
    sed -i 's#quay.io/kubernetes-ingress-controller/nginx-ingress-controller#registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/nginx-ingress-controller#g' mandatory.yaml
    ```   
2. 更kind类型  
 DaemonSet：官方原始文件使用的是deployment，replicate 为 1，这样将会在某一台节点上启动对应的nginx-ingress-controller pod。外部流量访问至该节点，由该节点负载分担至内部的service。测试环境考虑防止单点故障，改为DaemonSet然后删掉replicate ，配合亲和性部署在制定节点上启动nginx-ingress-controller pod，确保有多个节点启动nginx-ingress-controller pod，后续将这些节点加入到外部硬件负载均衡组实现高可用性

3. 添加hostNetwork  
true：添加该字段，暴露nginx-ingress-controller pod的服务端口（80）

4. 添加亲和性属性  
增加亲和性部署，有custom/ingress-controller-ready 标签的节点才会部署该DaemonSet

## 1.3 设置节点的label  
```
kubectl label nodes slave1 custom/ingress-controller-ready=true
kubectl label nodes slave2 custom/ingress-controller-ready=true
```

# 二、开始部署nginx-ingress-controller  
## 2.1 部署nginx-ingress-controller  
```
kubectl create -f mandatory.yaml
```
## 2.2 查看pod状态  
![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/90D833C5A38841E1846E90AD4D0AA611/20581)  


# 三、测试  
## 3.1 创建被测试的服务  
### 3.1.1 创建my-apache服务  
1. 准备所需要的yaml： my-apache.yaml   
    ```
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:  
    name: my-apache
    spec:
    replicas: 2
    template:
        metadata:
        labels:        
            run: my-apache
        spec:
        containers:
        - name: my-apache
            image: httpd:2.4
            ports:
            - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:  
    name: my-apache
    labels:   
        run: my-apache
    spec:
    type: NodePort
    ports:
    - port: 80
        targetPort: 80
        nodePort: 30002
    selector:    
        run: my-apache
    ```
2. 创建服务
    ```
    kubectl create -f my-apache.yaml
    ```

### 3.1.2 创建my-nginx服务  
1. 准备所需要的yaml: my-nginx.yaml   
    ```
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:  
    name: my-nginx
    spec:
    replicas: 2
    template:
        metadata:
        labels:        
            run: my-nginx
        spec:
        containers:
        - name: my-nginx
            image: nginx:1.7.9
            ports:
            - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:  
    name: my-nginx
    labels:    
        run: my-nginx
    spec:
    type: NodePort
    ports:
    - port: 80
        targetPort: 80
        nodePort: 30001
    selector:    
        run: my-nginx
    ```
2. 创建服务
    ```
    kubectl create -f my-nginx.yaml
    ```

### 3.1.3 查看服务创建情况  
```
[root@master ingress-nginx]# kubectl get pod  -owide
NAME                            READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE
my-apache-57874fd49c-jqpwr      1/1     Running   0          2m50s   192.168.2.116   slave2   <none>
my-apache-57874fd49c-tq999      1/1     Running   0          2m50s   192.168.1.100   slave1   <none>
my-nginx-79c95d84d4-jp5nz       1/1     Running   0          4m17s   192.168.2.115   slave2   <none>
my-nginx-79c95d84d4-t98sq       1/1     Running   0          4m17s   192.168.1.99    slave1   <none>
test-ingress-78fc77444f-kwtww   1/1     Running   2          38h     192.168.1.96    slave1   <none>
[root@master ingress-nginx]# kubectl get svc 
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes     ClusterIP   10.96.0.1        <none>        443/TCP        10d
my-apache      NodePort    10.104.46.97     <none>        80:30002/TCP   3m8s
my-nginx       NodePort    10.104.55.249    <none>        80:30001/TCP   4m35s
test-ingress   ClusterIP   10.102.193.247   <none>        80/TCP         38h
[root@master ingress-nginx]# 

```

### 3.1.4 测试服务是否可以正常方法  
1. 测试my-apache服务是否可以正常访问  
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/5F28850541E24AD29406B2232B999273/20584)  

2. 测试my-nginx服务是否可以正常访问  
    ```
    [root@master ingress-nginx]# curl 10.104.55.249
    <!DOCTYPE html>
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
    [root@master ingress-nginx]# 
    ```

    说明，两个服务都可以正常对外提供服务  
## 3.2 创建Ingress服务  
### 3.2.1 创建所需的yaml文件test-ingress-2.yaml   
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress-2
  namespace: default
spec:
  rules:
  - host: test.apache.ingress
    http:
      paths:
      - path: /
        backend:
          serviceName: my-apache
          servicePort: 80
  - host: test.nginx.ingress
    http:
      paths:
      - path: /
        backend:
          serviceName: my-nginx
          servicePort: 80
```
### 3.2.2 使其生效  
```
kubectl create -f test-ingress-2.yaml
```

## 3.3 域名映射说明  
1. 方案一：  
如果你的网络中存在dns服务器的话，就在dns中把这两个域名映射到nginx-ingress-controller运行的任意一个节点上   
2. 方案二：  
如果没有dns服务器的话，就修改host文件  
3. 方案三:(据说是正规的做法)  
    - 在部署nginx-ingress-controller的节点上，分别安装部署keepalive, 生成一个vip  
    - 在dns上把域名和vip做映射  
    https://github.com/kubernetes/contrib/tree/master/keepalived-vip  
    很长时间没有更新了， 不敢用  
4. 方案四： 
使用curl命令式，添加-H参数  


## 3.4 测试  
### 3.4.1 使用host方式  
0. 在同一个局域网内找一个节点，更新hosts文件
    ```
    echo "172.16.91.136  test.nginx.ingress" >> /etc/hosts
    echo "172.16.91.137  test.apache.ingress" >> /etc/hosts
    ```
1. 测试my-apache服务  
    ```
    [root@master ingress-nginx]#  curl test.apache.ingress
    <html><body><h1>It works!</h1></body></html>
    ```
2. 测试my-nginx服务 
    ```
    [root@master ingress-nginx]# curl test.nginx.ingress
    <!DOCTYPE html>
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
    [root@master ingress-nginx]# 
    ```
### 3.4.2 使用curl方式测试 
1. 测试my-apache服务  
    ```
    curl -H 'host: test.apache.ingress' http://172.16.91.136
    curl -H 'host: test.apache.ingress' http://172.16.91.137
    ```

2. 测试my-nginx服务 
    ```
    curl -H 'host: test.nginx.ingress' http://172.16.91.136
    curl -H 'host: test.nginx.ingress' http://172.16.91.137
    ```
