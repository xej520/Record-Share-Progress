
# service catelog 是用来解决什么问题的
- 对k8s上的服务或者app进行生命周期的管理， 
- 开发者的应用程序 与  基础服务(mysql，nginx，tomcat等)进行隔离


# service catelog的使用场景？  


# service catelog的优点？



# 阿里云是如何使用Service Catelog的？
https://yq.aliyun.com/articles/592156  



# Service Catelog 实现了什么东西？  
实现了由open Service Broker API(OSBAPI) 所定义的一组接口

# Service Catelog 可以运行在什么平台？
可以运行在
- k8s平台  
- Cloud Foundry  

# Service Catalog提供了什么？  



# 为什么要使用Service Catalog  
是为了系统架构的角度类分析的，将有状态的服务和无状态的服务分开。 
中间使用service catalog进行链接。

# 准备环境 
需要安装  Helm  
通过Helm来安装部署Service Catalog  

- 安装Helm  
- 安装Helm Charts的仓库地址 

- 安装Service Catalog   
    - 注册一个对象 servicecatalog
- 安装Service Broker 


# Open Service Broker API(OSBAPI)    
- Originated with Cloud Foundry as Services API 
- API with five resources 
    - List Catalog
    - Provision Instance 
    - Bind Instance 
    - Unbind Instance 
    - Deprovision Instance 

# 先了解一下什么是Service Catalog 以及 Service Catalog提供了什么？ 
Service Catalog 在k8s平台上实现了Open Service Broker API也就是OSBAPI，所定义的一组API接口。Open Service Broker API 来源于Cloud Foundry项目作为Service API接口。   
OSBAPI已经成为应用程序平台需要实现的与各种Service Broker交互的开放API接口。   
所以，对于用户来说，所需要做的事情就是根据上面列出的5种资源接口实现一个ServiceBroker接入到kubernetes中， 这样一来Cloud Foundry 就知道如何与用户所写的Service Broker交互，并通过操控Service Broker将其所提供的服务，在CloudFoundry的Service Marketplace中展示出来；  
我们(即Kubernetes或者Cloud Foundry) 作为应用程序平台并不关心一个Service Broker能够提供什么具体的服务；  
我们只关心Service Broker是否能够实现这个5个标准接口，并能够返回正确的返回值和返回格式；   
Service Broker最主要的返回值，就是告诉用户如何访问到其所提供或者创建的服务。  
比如服务的DNS Entry记录或者IP地址以及用于访问服务时所需要的认证信息，如用户名密码。  

# 现在，来看看这5个Service Broker的标准接口：  
## List Catalog  
- 用于列举出一个Service Broker所能提供的所有Service Class  
- Service Class 代表了一个Service Broker 所能够提供的不同种类的服务  

## Provision
- 通过这个接口，应用程序平台可以命令Service Broker创建一个服务实例Service Instance  
- 这一步意味着Service Broker需要作出一些必须的动作，以保证服务被正确创建并处于就绪状态以便用户连接   
- 对于某些服务而言，创建出服务实例已经足够使用并不再需要后续的其他Service Broker操作了。  
- 但在某些情况下，我们还需要调用第3个接口，Bind

## Bind
- Bind操作将会产生一个Binding对象  
- 在Cloud Foundry中，Bind操作将服务绑定到某一个应用程序上，这也是Cloud Foundry所特有的一个概念  
- 但绑定操作背后的 __通用含义__  在于用户需要通过bind操作得到访问服务所需要的认证信息，  
- 所以，Service Broker通过实现bind接口将认证等信息提供给用户

## Unbind
- 这一操作可以撤销binding对象， 清理或者注销认证信息  
- 总之，unbind中所作的操作与bind接口中所作的操作相反  
## Deprovision  
- 清理或者销毁服务实例  
- 例如，如果我们以数据库服务为例，Provision就代表创建一个数据库实例，可以通过数据库应用程序创建一个数据库实例，也可以通过命令创建一个数据库实例  
- 而用户可以通过后续的bind操作得到刚刚创建的数据库实例的用户名和密码  

在很多情况下，Provision仅仅创建了包含默认用户名密码的数据库实例， 而我们需要使用bind来修改密码或者创建新的密码   

# Service Plan  
与服务实例相关的还有一个新的概念，叫做服务计划(Service Plan)  
服务实例是某一种服务类型(Service Class)的实例，一个Service Broker的Catalog可以提供多种类型的服务   
而每一种类型的服务(Service Class) 都可以包含若干个不同的服务计划(Service Plan)  
所以，用户可以根据某一服务计划(Service Plan) 创建一种服务类型(Service Class)的实例（Service Instance）然后绑定这一服务实例。  

我们的数据库Service Broker可以提供多种不同的SQL数据库服务，不同种类Service Class的数据库服务，可能包括Mysql数据库和Postgress SQL 数据库， 每种不同的服务可以提供不同能力的服务实例， 
而服务计划(Service Plan)，就是在不同服务种类内(Service Class);  例如mysql或者postgress sql, 所能提供的不同的服务能力，这些能力可能包括数据库IOPS速率，存储空间大小，服务是否免费等等；  
通常而言，我们可以创建一些免费或者默认的服务计划， 来便于我们测试Service Broker的可用性以及各种服务的可用性。  
上述接口规范完整的描述了应用程序平台是如何与ServiceBroker交互的接口规范中并没有说明Service Broker和 应用平台需要做的具体动作。  
另一方面，通过这些接口返回的数据应用平台也需要保存服务与服务实例的相关信息，从而在后续过程中引用这些服务。   
到目前为止，已经有许多Service Broker的实现，也有一些基于OSBAPI实现的一些应用程序平台实例。   
当然，目前我们最感兴趣的能够接受Service Broker的平台还是Cloud Foundry 和 Kubernetes

OSBAPI 来源于Cloud Foundry, 而Kubernetes可以通过Service Catalog接入更多的服务。  


# 为什么 我们 需要使用 Service Catalog?  
主要是为何我们需要使用Open Service Broker API 在kubernetes中创建服务平台， 让我们先从理论和思维过程角度先看一下， 在Cloud Foundry环境中，通常我们只需要cf push即可将代码推送到云端，随后Cloud Foundry将编译源代码，运行程序并帮助用户维护应用程序的部署环境，如果代码更新了push到Cloud Foundry后，会自动部署新版本的应用程序， 但应用程序的状态数据丢失，因为现在环境中运行的是全新版本的应用程序，因此就程序所保存的数据就丢失了，这一点并不符合我们大部分时候的需求，为了改善这一点，通常我们会将这些无状态的应用程序链接到持久化存储上，这种情况下，场景被分成了两个独立的部分，一个是有状态的用于提供持久化存储的服务，另一个是专注于业务逻辑、无状态的应用程序，应用程序将通过输入输出在持久化存储服务中读取或者写入自己的状态，所以，在这里，我们将持久化存储以服务的形式提供出来，  
存储服务独立应用程序而存在，应用程序将通过外部链接的方式将状态存储这一特点的优势在于，服务将由更具有专业性的提供者来提供，比如，数据库的服务，监控以及扩展将由专门的DBA来提供。 所以应用程序开发者仅仅需要关注如何使自己所开发的应用程序链接上这些服务即可，那么，如何才能让我的应用程序链接上这些服务呢？   
__Service Broker__  
# Service Broker  
首先，应用程序需要确定依赖哪些外部服务，   
然后，应用程序平台需要知道应用程序所依赖的服务  
从而，应用程序平台能够通过给应用程序注入环境变量的方式，告知应用程序相关服务的相关细节信息  
这些细节信息就包括服务的位置以及访问这些服务所需要的认证信息等   
认证信息可以是Http basic auth, TLS, oauth, bearer token或者OIDC， OpenId Connect等等   

# kubernetes  
作为容器编排平台，一切都围绕容器为核心，容器，从设计上来说并非持久化的，拥有有限的生命周期，通常容器并不会持续运行很长时间，也不必运行很长时间，当一个容器销毁之后，总是可以用一个新容器来替换旧容器。  
那么一个应用程序的生命周期又如何呢？   
相比较于Cloud Foundry，在k8s中我们推送的是容器而不是应用程序猿代码，这是因为k8s并不直接编译运行应用程序源代码，而是运行应用程序所在的容器，所以在k8s环境中，我们推送的是应用程序容器，而不是应用程序源代码，用户一旦告诉k8s将要做的事情后，k8s就会下载容器镜像，然后在集群中运行容器，但这些容器在运行时，仍然需要后台持久化存储服务的支持，所以，在这里，Service Catalog 就会提供跟OSBAPPI相同的API来在k8s平台上提供类似持久化存储等服务，
## kubernetes service 与 Service Broker Service的区别？  
kubernetes Service是一种引用(reference point)作为后台pod的代理(proxy)  
而Service Catalog Service 被称为Cluster Service或者Cluster Service Class、Cluster Service Instances  
与k8s Service最大的不同在于他们没有对应需要代理请求的实际的后台pod

由于kubernetes只有容器的概念而没有应用程序的概念，所以在kubernetes中，我们需要将运行在容器中的无状态的应用程序链接到外部有状态的服务， 这需要在应用程序平台上实现 也即通过Service Catalog来实现的  

Service Catalog为k8s而设计，只要Service Broker实现了我们刚才的讨论的5个接口，就能够与其交互， Service Broker API 设计的目标之一就是保证兼容性。  

# 具体到Service Catalog原理?  
它提供了一组kubernetes原生对象以及一组API Server上的接口，因此，我们可以用与其他kubernetes对象同样格式的YAML或者JSON格式文件，通过HTTP请求操作这些Service Catalog所定义的对象  
与其他k8s原生对象一样，这些新定义的对象也拥有API Group, API Version, Spec以及Status等数据，为了能够做到这一点，Service Catalog重用了许多k8s项目代码，包括Go client Generator, API Macinery等等  
API Macinery提供了一个标准配置的API Server,包括TLS,证书，认证与授权，Open api, endpoint, Health Check Endpoint等等  
所以，Service Catalog所扩展的新的API Server与现有的k8s api server一样，提供同样的对象管理服务并在etcd中存储相应的数据，并且提供Watch服务让Contronller能够在Service Catalog所定义的新对象发生变化时，被及时通知。  
在Service Catalog所定义的Controller中，也实现了Desired State同步逻辑，Controller通过监控API Server所提供的各种对象的变化，并作出相应的动作实现Open Service Broker api 所定义的各种功能。  
这些动作包括更新API Server中各种对象的状态和数据以及根据变化调用Service Borker实现Open Service Broker api 调用， 所以在这里，Controller是实际调用Service Broker中所实现的List Catalog, Provision, bind, deprovision等操作的地方。  
基于上述设计和实现，Service Catalog与其他k8s组件一样将原生地出现在k8s集群中，并且kubectl也会像与原生API Server一样与Service Catalog所扩展的新的API Server交互。  

# 部署
这里以部署mariadb broker为例   
- 准备一个k8s集群 
- 安装Service catalog  
- 安装mariadb broker  
- 创建实例  
- 进行Bind操作   
- 使用binding进行服务的访问  
- 清理  

|服务名称|部署方式|  
|:---|:---|  
|Service catalog|Helm|  
|mariadb broker|Helm|  

# 镜像准备 
|镜像名字|版本号|
|quay.io/coreos/etcd|latest|





0. 部署k8s环境  
可以参考这篇文章：  
https://www.jianshu.com/p/48695bd6401f  


1. 当前k8s集群信息如下：  
```
kubectl get nodes  
```  
2. 安装部署Helm(版本至少是2.7以上)  
- 部署Helm  
可以参考下面文章进行部署安装：  
https://www.jianshu.com/p/200020e7a843  

- 为tiller设置权限  
https://github.com/kubernetes-incubator/service-catalog/blob/master/docs/install.md  
```
kubectl create clusterrolebinding tiller-cluster-admin \
    --clusterrole=cluster-admin \
    --serviceaccount=kube-system:default
```

- 添加Service Catalog的Helm Charts仓库地址  
```
helm repo add svc-cat https://svc-catalog-charts.storage.googleapis.com
```  
![repo add svc-cat](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/BD7E677B39274E45AA49C49EBE0030B0/21833)  
- 查看Service Catalog的Helm仓库是否已经在系统中注册了  
```
helm repo list    
```  
![helm repo list](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/F35FE70DCF3B4962BC1F4170D6559B33/21835)  
- 查看service-catalog 仓库中给我们提供了什么内容
```
helm search service-catalog  
```  
![helm search service-catalog](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/D3C16D45426C431293F5F0FCC4AAC5E1/21838)  

3. 安装service catalog  
可以使用许多可选的helm options来安装Service Catalog  
这里，我们仅仅使用标准的options将Service Catalog安装到catalog名字空间中。  
```
helm install svc-cat/catalog --name catalog --namespace catalog
kubectl get pods -n catalog  
```  
![install svc-catalog](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/9570BCBC6B374FD8B89CC18DDEFF1AF8/21840)  
![kubectl get pods -n catalog](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/ABB0877AB29748FDB9BB422600F49D8C/21842)    
4. 准备mariadb-broker  
https://github.com/prydonius/mariadb-broker  

```
git clone https://github.com/prydonius/mariadb-broker.git
cd mariadb-broker 
```
![git clone mariadb-broker](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/810D228A3C4148EF98C9895443EE520E/21844)   
![list mariadb-broker](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/45CAE7A4F5C24A81A013F2AF8E9547D6/21846)   


重新开启一个终端，安装mariadb-broker(选做)  

5. 通过helm安装mariadb-broker  
```
helm install --name mariadb-broker --namespace default charts/mariadb-broker
```  
![install mariadb-broker](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/06BE3071588D4FABAFB60319616F17D5/21848)  
6. 查看catalog安装进度    
```
kubectl get pods -n catalog
```
7. 查看broker的安装进度   
```
kubectl get pods -n default
```  
![kubectl get pods -n default](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/F424F9D98028409D8305DE8C5AE759C7/21850)    
  
# 如何使用  
1. 查看新生成的api 
```
kubectl api-versions   
```  
![api-versions](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/F85C0FDE38734850952770885C5A30D9/21852)    
2. 查看是否能获取到新的类型   
```
kubectl get clusterservicebrokers,clusterserviceclasses,clusterserviceplans,serviceinstances,servicebindings
```  
![get resources](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/D8AF9090C5814C848F0391F5C18050E3/21854)    


3. 如何获取mariadb-broker的用户名和密码   
```
cd /root/mariadb-broker
cat examples/mariadb-secret.yaml
```  
![mariadb-secret.yaml](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/0DCD6EDEF7154C7EACD93E194186CF42/21858)  
4. 根据mariadb-secret.yaml生成secret对象  
```
kubectl create -f examples/mariadb-secret.yaml
```  

5. 查看是否创建成功secret对象？  
```
kubectl get secrets  
```  
![create examples/mariadb-secret.yaml](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/B30D94E3BF324674A5E45DAD24973E6A/21856)

接下来，需要在Service Catalog中创建这个Service Broker  
6. 查看一下Service Broker的YAML文件  
```
cat examples/mariadb-broker.yaml  
```  
![more examples/mariadb-broker.yaml](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/06B9E1C885DA442289B0ACCA9A9E8622/21864)  
7. 创建Service Broker  
```
kubectl create -f examples/mariadb-broker.yaml
```  
![create -f examples/mariadb-broker.yaml](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/BF6A2AF34D4F4E60968F267DA268088D/21860)  
8. 查看一个所有Service Broker相关资源的列表   
```
kubectl get clusterservicebrokers,clusterserviceclasses,clusterserviceplans,serviceinstances,servicebindings  
```   
![资源列表](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/623294AF98D84CD1A2F494472E7B7C27/21867)  

9. 查看一个所有Service Broker相关资源的列表的详细信息   
```
kubectl get clusterservicebrokers, clusterserviceclasses, clusterserviceplans, serviceinstances, servicebindings -oyaml  
```
![资源列表](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/3B41C81E63F94BDBB90DCB28AA4487AE/21869)    
![资源列表](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/4D8E969B21414D87BB2279B18D018930/21871)   
![资源列表](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/45E3E4101FA94B14BA7A5A6A3AD73BBA/21873)   


10. 请求Broker创建一个mariadb数据库的实例(相当于创建了一个数据库)   
- 查看所需要的yaml    
```
cat examples/mariadb-instance.yaml    
```
![more mariadb-instance](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/AD08241DA7554AE6BCAF2AED81A88DB1/21875)  

- 进行创建  
```
kubectl create -f examples/mariadb-instance.yaml
```  
![create mariadb-instance.yaml](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/0C6117ABD13243A694E96990111F9001/21877)    

- 查看进度  
```
kubectl get clusterservicebrokers,clusterserviceclasses,clusterserviceplans,serviceinstances,servicebindings  
```  
![资源列表](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/52F610CD3F844312AD455A86B89B8DD8/21879)  
- 查看详细信息  
```
kubectl get serviceinstances mariadb-instance -o yaml  
```  
![mariadb-instance](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/EF946868E9F6460ABFACD4A54D8297C0/21881)   
![mariadb-instance](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/2B99DD6FD82B4A5CBA3D3F030C930678/21883)    

11. 创建binding对象
- 目的：是为了获取访问数据库服务的认证信息   
- 查看binding yaml文件  
```
cat examples/mariadb-binding.yaml   
```  
![more mariadb-binding.yaml](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/4F8101BD8E9241C18A78C482ADECE179/21885)   
- 进行创建 
```
kubectl create -f examples/mariadb-binding.yaml 
```  
![create mariadb-binding.yaml]()  

- 这时，我们应该创建了一些binding action 运行在后台，如果我们浏览binding对象的时候，我们将看到bind结果已经注入了，bind结果是True  
- 查看binding对象  
```
kubectl get servicebindings mariadb-binding -o yaml   
```  
![more mariadb-binding]()  
- 应该有一个secret被创建了出来  
- 进行获取secret  
```
kubectl get secret mariadb-instance-credentials   
```  
![get secret mariadb-instance-credentials]()     
- kubectl get secret mariadb-instance-credentials -oyaml  
```

```
- 


=====================================
# 主要步骤 :  
1. 安装helm  
2. 添加svc-catalog的charts的仓库地址 
3. 通过helm 安装service catalog  


# 安装service broker  
1. 

