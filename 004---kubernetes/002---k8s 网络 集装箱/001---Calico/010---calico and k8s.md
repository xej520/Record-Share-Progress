# <center>Simple Policy Demo for Calico on Kubernetes </center>  
1. k8s版本：  
kuberctl version  
    ```
    Client Version: version.Info{Major:"1", Minor:"9",     
    GitVersion:"v1.9.0",  
    GitCommit:"925c127ec6b946659ad0fd596fa959be43f0cc05",   
    GitTreeState:"clean", BuildDate:"2017-12-15T21:07:38Z",   
    GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
    
    Server Version: version.Info{Major:"1", Minor:"9",   
    GitVersion:"v1.9.0",   
    GitCommit:"925c127ec6b946659ad0fd596fa959be43f0cc05",   
    GitTreeState:"clean", BuildDate:"2017-12-15T20:55:30Z",   
    GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}  
    ```  
2. calico 版本 ：  
calicoctl version  
    ```
    Client Version:    v1.6.1   
    Build date:        2017-09-28T01:12:35+0000   
    Git commit:        1724e011    
    Cluster Version:   v2.6.2   
    Cluster Type:      unknown  
    ```  
3. 当前环境   

    |主机名|IP|系统|当前服务| 
    |:---|:---|:---|:---|
    |master|172.16.91.185|centos7|kube-apiserver, kube-controller-manager, kube-scheduler, calico|
    |slave1|172.16.91.186|centos7|kube-let,kube-proxy, calico|
    |slave1|172.16.91.187|centos7|kube-let,kube-proxy, calico|
    |harbor|172.16.91.222|centos7|etcd|



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
进入从节点如slave1  
3. 查看路由信息  
   ```
    default via 172.16.91.254 dev ens33 proto static metric 100 
    172.16.0.0/16 dev ens33 proto kernel scope link src 172.16.91.186 metric 100 
    172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
    172.20.140.64/26 via 172.16.91.187 dev ens33 proto bird 
    blackhole 172.20.140.192/26 proto bird 
    172.20.140.206 dev cali3b76f8148f9 scope link 
    172.20.140.207 dev cali7907ffa73ab scope link 
    172.20.140.208 dev cali6448c93f838 scope link
   ```  
   ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/CE28021C126248FB9F7D16FF53199D9A/20264)  



# Enable isolation  
```
kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
  namespace: policy-demo
spec:
  podSelector:
    matchLabels: {}
EOF
```
## Test Isolation  
```
# Run a Pod and try to access the `nginx` Service.
$ kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
Waiting for pod policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false

If you don't see a command prompt, try pressing enter.

/ # wget -q --timeout=5 nginx -O -
wget: download timed out
/ #
```  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/123F68573F2F4D4BA6D65CFEB9C866E3/20372)  



# Allow Access using a NetworkPolicy  








