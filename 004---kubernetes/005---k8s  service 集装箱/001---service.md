# kubernetes Service  
参考文献：  
https://blog.csdn.net/watermelonbig/article/details/79693962  

A、k8s的Service定义了一个服务的访问入口地址，前端的应用通过这个入口地址访问其背 后的一组由Pod副本组成的集群实例，来自外部的访问请求被负载均衡到后端的各个容器应用上。  
B、Service与其后端Pod副本集群之间则是通过Label Selector来实现对接的；  
C、而RC的作用相当于是保证Service的服务能力和服务质量始终处于预期的标准。Service 定义可以基于 POST 方式，请求 apiserver 创建新的实例。  
D、一个 Service 在 Kubernetes 中是一个 REST 对象。  
本文对Service的使用进行详细说明，包括Service的负载均衡、外网访问、DNS服务的搭建、Ingress7层路由机制等。  

# Service 定义详解  
## yaml格式的Service定义文件的完整内容  
```
apiVersion: v1
kind: Service
matadata:
  name: string
  namespace: string
  labels:
  - name: string
  annotations:
  - name: string
spec:
  selector: []
  type: string
  clusterIP: string
  sessionAffinity: string
  ports:
  - name: string
    protocol: string
    port: int
    targetPort: int
    nodePort: int
  status:
    loadBalancer:
      ingress:
        ip: string
        hostname: string
```  
## 对Service定义文件中各属性的说明表  
|属性名称|取值类型|是否必须|取值说明| 
|:---|:---|:---|:---|
|version|string|Required|v1|
|kind|string|Required|Service|
|metadata|object|Required|元数据|
|metadata.name|string|Required|Service名称| 
|metadata.namespace|string|Required|命名空间，默认为default|
|metadata.labels[]|list||自定义标签属性列表| 
|metadata.annotation[]|list||自定义注解属性列表| 
|spec|object|Required|详细描述|
|spec.selector[]|list|Required|Label Selector配置，将选择具有指定Label标签的Pod作为管理范围|
|spec.type|string|Required|Service的类型，指定Service的访问方式，默认值为ClusterIP。取值范围如下:ClusterIP: 虚拟服务的IP，用于k8s集群内部的pod访问，在Node上kube-proxy通过设置的Iptables规则进行转发。NodePort：使用宿主机的端口，使用能够访问各Node的外部客户端通过Node的IP地址和端口就能访问服务。LoadBalancer: 使用外接负载均衡器完成到服务的负载分发，需要在spec.status.loadBalancer字段指定外部负载均衡器的IP地址，并同时定义nodePort和clusterIP，用于公有云环境。|
|spec.clusterIP|string||虚拟服务的IP地址，当type=clusterIP时，如果不指定，则系统进行自动分配。也可以手工指定。当type=LoadBalancer时，则需要指定。|
|spec.sessionAffinity|string||是否支持Session，可选值为ClientIP，表示将同一个源IP地址的客户端访问请求都转发到同一个后端Pod。默认值为空。|
|spec.ports[]|list||Service需要暴露的端口列表| 
|spec.ports[].name|string||端口名称|
|spec.ports[].protocol|string||端口协议，支持TCP和UDP，默认值为TCP|
|spec.ports[].port|int||服务监听的端口号|
|spec.ports[].targetPort|int||需要转发到后端Pod的端口号|
|spec.ports[].nodePort|int||当spec.type=NodePort时，指定映射到物理机的端口号|
|status|object||当spec.type=LoadBalancer时，设置外部负载均衡器的地址，用于公有云环境|
|status.loadBalancer|object||外部负载均衡器|
|status.loadBalancer.ingress|object||外部负载均衡器|
|status.loadBalancer.ingress.ip|string||外部负载均衡器的IP地址|
|status.loadBalancer.ingress.hostname|string||外部负载均衡器的主机名|  

# Service, RC, Pod 架构层次关系  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/7C344CB537C94E69928EFED25F3470E9/20472)  

# VIP和Service 代理  
## kube-proxy
运行在每个Node上的kube-proxy进程其实就是一个智能的软件负载均衡器，它会负责把对Service的请求转发到后端的某个Pod实例上并在内部实现服务的负载均衡与会话保持机制。Service不是共用一个负载均衡器的IP，而是被分配了一个全局唯一的虚拟IP地址，称为Cluster IP。   
在Service的整个生命周期内，它的Cluster IP不会改变。 
kube-proxy 负责为 Service 实现了一种 VIP（虚拟 IP）的形式，而不是 ExternalName 的形式。  
在k8s v1.2版本之前默认使用userspace提供vip代理服务，从 Kubernetes v1.2 起，默认是使用 iptables 代理。

## iptables 代理模式
这种模式，kube-proxy 会监视 Kubernetes master 对 Service 对象和 Endpoints 对象的添加和移除。 对每个 Service，它会创建相关 iptables 规则，从而捕获到达该 Service 的 clusterIP（虚拟 IP）和端口的请求，进而将请求重定向到 Service 的一组 backend 中的某个上面。 对于每个 Endpoints 对象，它也会创建 iptables 规则，这个规则会选择一个 backend Pod。  
默认的策略是，随机选择一个 backend。 实现基于客户端 IP 的会话亲和性，可以将 service.spec.sessionAffinity 的值设置为 "ClientIP" （默认值为 "None"）。
和 userspace 代理类似，网络返回的结果是，任何到达 Service 的 IP:Port 的请求，都会被代理到一个合适的 backend，不需要客户端知道关于 Kubernetes、Service、或 Pod 的任何信息。 这应该比 userspace 代理更快、更可靠。然而，不像 userspace 代理，如果初始选择的 Pod 没有响应，iptables 代理能够自动地重试另一个 Pod，所以它需要依赖 readiness probes。  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/5E8F1056307949DC8C833321CCF21F81/20475)  

# 发布服务---type类型  
对一些应用希望通过外部（Kubernetes 集群外部）IP 地址暴露 Service。   
Kubernetes ServiceTypes 允许指定一个需要的类型的 Service，默认是 ClusterIP 类型。  
Type 的取值以及行为如下：  
- ClusterIP：通过集群的内部 IP 暴露服务，选择该值，服务只能够在集群内部可以访问，这也是默认的 ServiceType。  
- NodePort：通过每个 Node 上的 IP 和静态端口（NodePort）暴露服务。NodePort 服务会路由到 ClusterIP 服务，这个 ClusterIP 服务会自动创建。通过请求<NodeIP>:<NodePort>，可以从集群的外部访问一个 NodePort 服务。  
- LoadBalancer：使用云提供商的负载局衡器，可以向外部暴露服务。外部的负载均衡器可以路由到 NodePort 服务和 ClusterIP 服务。  
- ExternalName：通过返回 CNAME 和它的值，可以将服务映射到 externalName 字段的内容（例如， foo.bar.example.com）。 没有任何类型代理被创建，这只有 Kubernetes 1.7 或更高版本的 kube-dns 才支持。  

k8s中有3种IP地址：
- Node IP： Node节点的IP地址，这是集群中每个节点的物理网卡的IP地址；
- Pod IP： Pod的IP地址，这是Docker Engine根据docker0网桥的IP地址段进行分配的，通常是一个虚拟的二层网络；
- Cluster IP：Service 的IP地址，这也是一个虚拟的IP，但它更像是一个“伪造”的IP地址，因为它没有一个实体网络对象，所以无法响应ping命令。它只能结合Service Port组成一个具体的通信服务端口，单独的Cluster IP不具备TCP/IP通信的基础。  

在k8s集群之内，Node IP网、Pod IP网与Cluster IP网之间的通信采用的是k8s自己设计的一种编程实现的特殊的路由规则，不同于常见的IP路由实现


# 服务发现 
Kubernetes 支持2种基本的服务发现模式 —— 环境变量和 DNS。
__环境变量__  
当 Pod 运行在 Node 上，kubelet 会为每个活跃的 Service 添加一组环境变量。 它同时支持 Docker links兼容 变量、简单的 {SVCNAME}_SERVICE_HOST 和 {SVCNAME}_SERVICE_PORT 变量，这里 Service 的名称需大写，横线被转换成下划线。
举个例子，一个名称为 "redis-master" 的 Service 暴露了 TCP 端口 6379，同时给它分配了 Cluster IP 地址 10.0.0.11，这个 Service 生成了如下环境变量：  
    
    REDIS_MASTER_SERVICE_HOST=10.0.0.11
    REDIS_MASTER_SERVICE_PORT=6379
    REDIS_MASTER_PORT=tcp://10.0.0.11:6379
    REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
    REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
    REDIS_MASTER_PORT_6379_TCP_PORT=6379
    REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11
这意味着需要有顺序的要求 —— Pod 想要访问的任何 Service 必须在 Pod 自己之前被创建，否则这些环境变量就不会被赋值。DNS 并没有这个限制。  
__DNS__  
一个强烈推荐的集群插件 是 DNS 服务器。 DNS 服务器监视着创建新 Service 的 Kubernetes API，从而为每一个 Service 创建一组 DNS 记录。 如果整个集群的 DNS 一直被启用，那么所有的 Pod 应该能够自动对 Service 进行名称解析。  
例如，有一个名称为 "my-service" 的 Service，它在 Kubernetes 集群中名为 "my-ns" 的 Namespace 中，为 "my-service.my-ns" 创建了一条 DNS 记录。 在名称为 "my-ns" 的 Namespace 中的 Pod 应该能够简单地通过名称查询找到 "my-service"。 在另一个 Namespace 中的 Pod 必须限定名称为 "my-service.my-ns"。 这些名称查询的结果是 Cluster IP。  
Kubernetes 也支持对端口名称的 DNS SRV（Service）记录。 如果名称为 "my-service.my-ns" 的 Service 有一个名为 "http" 的 TCP 端口，可以对 "_http._tcp.my-service.my-ns" 执行 DNS SRV 查询，得到 "http" 的端口号。  
Kubernetes DNS 服务器是唯一的一种能够访问 ExternalName 类型的 Service 的方式。 更多信息可以查看DNS Pod 和 Service。  
Kubernetes 从 1.3 版本起， DNS 是内置的服务，通过插件管理器 集群插件 自动被启动。Kubernetes DNS 在集群中调度 DNS Pod 和 Service ，配置 kubelet 以通知个别容器使用 DNS Service 的 IP 解析 DNS 名字。

# Service的基本用法  
&ensp;&ensp;&ensp;&ensp;一般来说，对外提供服务的应用程序需要通过某种机制来实现，对于容器应用最简便的方式就是通过TCP/IP机制及监听IP和端口号来实现。     
创建一个基本功能的Service  
(1) 例如，我们定义一个提供web服务的RC，由两个tomcat容器副本组成，每个容器通过containerPort设置提供服务号为8080： 
webapp-rc.yaml  
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: webapp
spec:
  replicas: 2
  template:
    metadata:
      name: webapp
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: tomcat
        ports:
        - containerPort: 8080
```
创建该RC  
```
kubectl create -f webapp-rc.yaml 
```  
获取Pod的IP地址：  
```
[root@master service]# kubectl get pods -l app=webapp -o yaml| grep podIP | grep -v cni
    podIP: 192.168.2.94
    podIP: 192.168.1.73
[root@master service]# 
```  
直接通过这两个Pod的IP地址和端口号访问Tomcat服务：  
```
[root@master service]# curl 192.168.2.94:8080



<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8" />
        <title>Apache Tomcat/8.5.34</title>
        <link href="favicon.ico" rel="icon" type="image/x-icon" />
        <link href="favicon.ico" rel="shortcut icon" type="image/x-icon" />
        <link href="tomcat.css" rel="stylesheet" type="text/css" />
    </head>
........
```  
说明可以直接通过podIP进行访问  
直接通过Pod的IP地址和端口号可以访问容器内的应用服务，但是Pod的IP地址是不可靠的，例如Pod所在的Node发生故障，Pod将被k8s重新调度到另一台Node。Pod的IP地址将发生变化，更重要的是，如果容器应用本身是分布式的部署方式，通过多个实例共同提供服务，就需要在这些实例的前端设置一个负载均衡器来实现请求的分发。kubernetes中的Service就是设计出来用于解决这些问题的核心组件。   

(2)  为了让客户端应用能够访问到两个Tomcat Pod 实例，需要创建一个Service来提供服务  
k8s提供了一种快速的方法，即通过kubectl expose命令来创建：  
```
[root@master service]# kubectl expose deployment webapp
service/webapp exposed
``` 
查看新创建的Service可以看到系统为它分配了一个虚拟的IP地址（clusterIP），而Service所需的端口号则从Pod中的containerPort复制而来：  
```
[root@master service]# kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP    6d10h
webapp       ClusterIP   10.103.4.220   <none>        8080/TCP   5s
[root@master service]# 
```  
接下来，我们就可以通过Service的IP地址和Service的端口号访问该Service了：  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/121D974A91B04A68B3D0A9BD80EFC116/20477)  

这里，对Service地址10.103.4.220:8080的访问被自动负载分发到了后端两个Pod之一。  

(3) 除了使用kubectl expose命令创建Service，我们也可以通过配置文件定义Service，再通过kubectl create命令进行创建  
例如前面的webapp就用，我们可以设置一个Service：  
webapp-svc.yaml  
```
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  ports:
  - port: 8081
    targetPort: 8080
  selector:
    app: webapp
```
本例中ports定义部分指定了Service所需的虚拟端口号为8081，由于与Pod容器端口号8080不一样，所以需要在通过targetPort来指定后端Pod的端口  
selector定义部分设置的是后端Pod所拥有的label: app=webapp  
(4) 目前kubernetes提供了两种负载分发策略：RoundRobin和SessionAffinity  
- RoundRobin：轮询模式，即轮询将请求转发到后端的各个Pod上
- SessionAffinity：基于客户端IP地址进行会话保持的模式，第一次客户端访问后端某个Pod，之后的请求都转发到这个Pod上   
默认是RoundRobin模式。  
# 多端口Service  
有时候，一个容器应用提供多个端口服务，可以按下面这样定义：  
```
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  ports:
  - name: web
    port: 8080
    targetPort: 8080
  - name: management
    port: 8005
    targetPort: 8005
  selector:
    app: webapp 
```  
另一个例子是两个端口使用了不同的4层协议，即TCP和UDP  
```
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.10.10.100
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```  
# Headless Service  
有时不需要或不想要负载均衡，以及单独的 Service IP。 遇到这种情况，可以通过指定 Cluster IP（spec.clusterIP）的值为 "None" 来创建 Headless Service。
这个选项允许开发人员自由寻找他们自己的方式，从而降低与 Kubernetes 系统的耦合性。 应用仍然可以使用一种自注册的模式和适配器，对其它需要发现机制的系统能够很容易地基于这个 API 来构建。
对这类 Service 并不会分配 Cluster IP，kube-proxy 不会处理它们，而且平台也不会为它们进行负载均衡和路由。仅依赖于Label Selector将后端的Pod列表返回给调用的客户端。

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
  clusterIP: None
  selector:
    app: nginx
```
这样，Service就不再具有一个特定的ClusterIP地址，对其进行访问将获得包含Label"app=nginx"的全部Pod列表，然后客户端程序自行决定如何处理这个Pod列表。  
例如, StatefulSet就是使用Headless Service为客户端返回多个服务地址。  
__Lable Secector：__  
配置 Selector：对定义了 selector 的 Headless Service，Endpoint 控制器在 API 中创建了 Endpoints 记录，并且修改 DNS 配置返回 A 记录（地址），通过这个地址直接到达 Service 的后端 Pod上。  
不配置 Selector：对没有定义 selector 的 Headless Service，Endpoint 控制器不会创建 Endpoints 记录。   

# 外部服务Service——没有 selector 的 Service    
在某些环境中，应用系统需要将一个外部数据库用为后端服务进行连接，或将另一个集群或Namespace中的服务作为服务的后端，这时可以通过创建一个无Label Selector的Service实现：  
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```  
Servcie 抽象了该如何访问 Kubernetes Pod，但也能够抽象其它类型的 backend，例如：   
希望在生产环境中使用外部的数据库集群。   
希望服务指向另一个 Namespace 中或其它集群中的服务。    
正在将工作负载同时转移到 Kubernetes 集群和运行在 Kubernetes 集群之外的 backend。   
由于这个 Service 没有 selector，就不会创建相关的 Endpoints 对象。可以手动将 Service 映射到指定的 Endpoints：  
```
kind: Endpoints
apiVersion: v1
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 1.2.3.4
    ports:
      - port: 9376
```  
注意：Endpoint IP 地址不能是 loopback（127.0.0.0/8）、 link-local（169.254.0.0/16）、或者 link-local 多播（224.0.0.0/24）。   
访问没有 selector 的 Service，与有 selector 的 Service 的原理相同。请求将被路由到用户定义的 Endpoint（该示例中为 1.2.3.4:9376）。   
ExternalName Service 是 Service 的特例，它没有 selector，也没有定义任何的端口和 Endpoint。 相反地，对于运行在集群外部的服务，它通过返回该外部服务的别名这种方式来提供服务。   
```
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```  
当查询主机 my-service.prod.svc.CLUSTER时，集群的 DNS 服务将返回一个值为 my.database.example.com 的 CNAME 记录。 访问这个服务的工作方式与其它的相同，唯一不同的是重定向发生在 DNS 层，而且不会进行代理或转发。 如果后续决定要将数据库迁移到 Kubernetes 集群中，可以启动对应的 Pod，增加合适的 Selector 或 Endpoint，修改 Service 的 type。   

# 10、集群外部访问Pod或Service的方法  
由于Pod和Service是k8s集群范围内的虚拟概念，所以集群外的客户端系统无法通过Pod的IP地址或者Service的虚拟IP地址和虚拟端口号访问到它们。
为了让外部客户端可以访问这些服务，可以将Pod或Service的端口号映射到宿主机，以使得客户端应用能够通过物理机访问容器应用。
## 10.1、将容器应用的端口号映射到物理机
(1) 通过设置容器级别的hostPort，将容器应用的端口号映射到物理机上  
文件pod-hostport.yaml   
```
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  containers:
  - name: webapp
    image: tomcat
    ports:
    - containerPort: 8080
      hostPort: 8081

```  
hostPort的值可以跟containerPort的值一样  
通过kubectl create创建这个Pod：  
```
kubectl create -f pod-hostport.yaml
```  
通过物理机的IP地址和8081端口号访问Pod内的容器服务：  
```
curl 172.16.91.137:8081
```
(2) 通过设置Pod级别的hostNetwork=true，该Pod中所有容器的端口号都将被直接映射到物理机上
设置hostWork=true时需要注意，在容器的ports定义部分如果不指定hostPort，则默认hostPort等于containerPort，如果指定了hostPort，则hostPort必须等于containerPort的值。
文件pod-hostnetwork.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: webapp-hostnetwork
  labels:
    app: webapp-hostnetwork
spec:
  hostNetwork: true
  containers:
  - name: webapp-hostnetwork
    image: tomcat
    imagePullPolicy: Never
    ports:
    - containerPort: 8080

```  
创建这个Pod：  
```
kubectl create -f pod-hostnetwork.yaml
```
```
[root@master pod]# kubectl get pod -owide
NAME                      READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE
webapp-77df6fff64-dhczl   1/1     Running   1          14h   192.168.2.95    slave2   <none>
webapp-77df6fff64-kpl6w   1/1     Running   1          14h   192.168.1.74    slave1   <none>
webapp-hostnetwork        1/1     Running   0          7s    172.16.91.136   slave1   <none>

```  
通过物理机的IP地址和8080端口访问Pod的容器服务  
```
curl slave1:8080  
```  
## 将Service的端口号映射到物理机  
(1) 通过设置nodePort映射到物理机，同时设置Service的类型为NodePort  
文件webapp-svc-nodeport.yaml  
```
apiVersion: v1
kind: Service
metadata:
  name: webapp-nodeport
spec:
  type: NodePort
  ports:
  - port: 8090
    targetPort: 8080
    nodePort: 8090
  selector:
    app: webapp
```
创建这个Service：  
```
kubectl create -f webapp-svc-nodeport.yaml
```
```
[root@master service]# kubectl get svc
NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes        ClusterIP   10.96.0.1      <none>        443/TCP          7d1h
webapp            ClusterIP   10.103.4.220   <none>        8080/TCP         14h
webapp-nodeport   NodePort    10.99.248.88   <none>        8090:31090/TCP   6s

```  
通过物理机的IP和端口访问：(4种方式访问) 
```
#Service的IP，即ClusterIP
curl 10.99.248.88:8090 
#master的IP，
curl master:31090 
curl slave1:31090 
curl slave2:31090 
```  
也就是说，只要是nodePort端口，就会在物理机上开始监听端口号  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/DC77341C95D2434294488378985BAB3E/20480)  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/E976C39C85E04C9596CC5129C1A3A28B/20483)  
如果访问不通，查看下物理机的防火墙设置  
同样，对该Service的访问也将被负载分发到后端多个Pod上  

(2) 通过设置LoadBalancer映射到云服务商提供的LoadBalancer地址    
这种用法仅用于在公有云服务提供商的云平台上设置Service的场景。  
status.loadBalancer.ingress.ip设置的146.148.47.155为云服务商提供的负载均衡器的IP地址。对该Service的访问请求将会通过LoadBalancer转发到后端的Pod上，负载分发的实现方式依赖云服务商提供的LoadBalancer的实现机制。  
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: Myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
    nodePort: 30061
  clusterIP: 10.0.171.239
  loadBalancerIP: 78.11.24.19
  type: LoadBalancer
status:
  loadBalancer:
    ingree:
    - ip: 146.148.47.155
```  

# Ingress: HTTP 7层 路由机制  根据前面的
根据前面对Service的使用说明，我们知道Service的表现形式为IP:Port，即工作在TCP/IP层，  
而对于基于HTTP的服务来说，不同的URL地址经常对应到不同的后端服务或者虚拟服务器，这些应用层的转发机制仅通过kubernetes的Service机制是无法实现的。  
kubernetes v1.1版本中新增的Ingress资源对象将不同URL的访问请求转发到后端不同的Service，以实现HTTP层的业务路由机制。

使用 Ingress 进行负载分发时，Ingress Controller 将基于 Ingress 规则将客户端请求直接转发到 Service 对应的后端 Endpoint（即 Pod）上，这样会跳过 kube-proxy 的转发功能，kuber-proxy 不在起作用。  
如果 Ingress Controller 提供的是对外服务，则实际上实现的是边缘路由器的功能。

















