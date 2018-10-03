# <center>Simple Policy Demo for Calico on Kubernetes </center>  
k8s版本：  
kuberctl version  
```
Client Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.0", GitCommit:"925c127ec6b946659ad0fd596fa959be43f0cc05", GitTreeState:"clean", BuildDate:"2017-12-15T21:07:38Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.0", GitCommit:"925c127ec6b946659ad0fd596fa959be43f0cc05", GitTreeState:"clean", BuildDate:"2017-12-15T20:55:30Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}  
```
# Configure Namespaces  
```
kubectl create ns policy-demo 
```
# Create demo Pods  
## 创建Nginx Pod  
1. Run the Pods.  
    ```
    kubectl run --namespace=policy-demo nginx --replicas=2 --image=nginx 
    ```
2. Create the Service.
    ```
    kubectl expose --namespace=policy-demo deployment nginx --port 80 

    NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
    nginx     ClusterIP   10.68.246.67   <none>        80/TCP    22s
    ```  
## Ensure the nginx service is accessible.
创建一个pod，来测试Nginx 服务，  
busybox 是一个工具镜像  
1. 创建一个pod，专门用来进行测试连通性, 并进入pod  
kubectl run -n policy-demo access --rm -it --image busybox /bin/sh  
2. 访问Nginx服务   
    wget -q nginx -O -  
    ```
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
    ```  
关闭测试Pod access  ，
进入从节点 

# Enable isolation  

## Test Isolation  



# Allow Access using a NetworkPolicy  








