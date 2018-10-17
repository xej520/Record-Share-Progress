# 部署kubernetes在AWS上  
主要参考文献：  
https://github.com/kubernetes/kops/blob/master/docs/aws.md  
https://kubernetes.io/docs/setup/custom-cloud/kops/  
https://blog.csdn.net/cloudvtech/article/details/80539086  
https://blog.csdn.net/qq_36348557/article/details/79795288
https://my.oschina.net/geekidentity/blog/1635540  

说明：  
在AWS上部署kuberntes 可以使用下面的两个工具？  
- kubeadm 
- kops(是在建立在kubeadm的基础之上的) 

两种方式对比：  
- kubeadm需要预安装kubectl和kubelet，kops需要预安装kubectl;
- kubeadm只负责在部署Kubenetes在computers running Linux，不负责infrastructure orchestration，kops负责infrastructure orchestration和Kubenetes orchestration;
- kubeadm以容器化的方式运行Kubernetes服务，kops以普通服务的方式运行Kubernetes;
- Integration with kubeadm kops now uses kubeadm for some RBAC related functionality;





服务器选择？  
- 国内服务器的话，选择北京区域，不要选择 宁夏区域，因为宁夏区域目前没有kops 
- 

本文使用kops命令进行安装部署    
官网文档：  
https://kubernetes.io/docs/setup/custom-cloud/kops/  

# 准备工作
## 如何切换成root用户？  
登陆时，可能是以centos这个用户进行登录，如何切换到root用户呢？  
```
通过命令 sudo -s 可以实现从普通用户转为root用户
通过 exit 可以从root用户退回到普通用户
``` 
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/E7F0AAA1293D42019C65AC1E1138A61E/20498)  
相关参考文献：   
https://blog.csdn.net/coder__cs/article/details/80917886  
https://www.cnblogs.com/520wife/p/7744015.html  
https://blog.csdn.net/wz947324/article/details/80313315  
 
## 更新配置环境 
```
yum install wget
wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum install ./epel-release-latest-*.noarch.rpm
yum -y update
yum -y install python-pip
```

## 安装aws 命令行工具 
```
pip install --upgrade pip
pip install awscli --upgrade —user
export PATH=~/.local/bin:$PATH
```

## centos上安装 kops工具  
```
wget https://github.com/kubernetes/kops/releases/download/1.10.0/kops-linux-amd64
chmod +x kops-linux-amd64
mv kops-linux-amd64 /usr/local/bin/kops
```  

## 配置AWS资源 
我们需要一个AWS账号 用于运行kops， 该用户需要具有以下的权限:  
```
AmazonEC2FullAccess
AmazonRoute53FullAccess
AmazonS3FullAccess
IAMFullAccess
AmazonVPCFullAccess
```
使用aws cli 来创建用户组、用户 以及  access key
### 创建一个新的AWS Group 、给AWS Group 赋予权限 
1. 创建用户组  
```
aws iam create-group --group-name kops
``` 
2. 给用户组设置权限 
```
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
```
### 创建用户  
1. 创建用户  
```
aws iam create-user --user-name kops
```    
2. 将用户添加到用户组里  
```
aws iam add-user-to-group --user-name kops --group-name kops  
```
### 创建access key  
```
aws iam create-access-key --user-name kops
```
请记录下这里生成的SecretAccessKey 和 AccessKeyID，  需要配置到客户端，   
并且手动导入环境变量  

### 使用aws-cli登陆这个新用户 


### 



# 创建集群  
## 创建route53域  


## 创建一个S3存储  
Kops把K8s集群的配置存储在AWS的S3中，每一个集群的配置对应于一个S3文件，  
所有我们创建一个S3的bucket用于存储集群的配置  
替换下面的变量bucket-name, 如设置成:  
```
export BUCKET=cluster.k8s.local
aws s3api create-bucket \
    --bucket $BUCKET \
    --region us-west-2 \
    --create-bucket-configuration  LocationConstraint=us-west-2

```  
注意：  
可以将cluster.k8s.local改成自己的值  
```
aws s3api put-bucket-versioning --bucket $BUCKET  --versioning-configuration Status=Enabled
```  
下面开始创建kubernetes 集群
## 创建集群  
### 配置环境变量，定义集群的名字和配置的URL 
```
export NAME= test.k8s.local
export KOPS_STATE_STORE=s3://$BUCKET
```
注意根据自己的需求，重新设置环境变量NAME的值  

### 创建集群配置文件  
1. 获取可用的zones    
```
aws ec2 describe-availability-zones --region us-west-2
```  
当前在使用 us-west-2的region可用的az是这三个  
```

```

因此，从上面的几个选项中，选择一个可用的zones即可   
2. 创建集群时用到的配置文件 
```
kops create cluster \
    --zones us-west-2a \
    ${NAME}
```  
注意
- 这一步只是生成了集群的配置文件，并存储在S3中。  
- 需要替换成自己的zones值

3. 查看刚才生成的配置  
```
kops edit cluster ${NAME}
```

4. 若配置不满足自己的需求，可以使用下面的命令修改配置
```
kops edit cluster ${NAME}
```  

### 开始真正部署kubernetes集群    
1. 创建kubernetes集群  
```
kops update cluster ${NAME} --yes  
```  
缺省的情况下，kops会创建所有对应的AWS资源，包含VPC，子网，EC2，Auto Scaling Group，ELB，安全组等等。  
如果需要安装在特定的子网，在创建集群时可以指定子网的id。另外，也支持跨AZ的HA配置    

2. 查看集群状态  
集群安装好之后，需要几分钟时间启动，我们可以用kubectl来查看一下状态(Kops会自动把cluster的配置写到~/.kube/config 文件中作为缺省配置)：
```
kubectl cluster-info  
```  




### 如何扩展集群(新增节点到集群)    
在云上的K8s集群可以很方便的扩展，如果你的集群的计算资源都用的差不多了，你希望扩展你的集群的时候，有两种办法。  
1. 方法一： 
直接修改AWS的auto scaling group。  
KOps会在AWS上创建两个auto scaling group:  
一个用于Master, 
一个用于Node 
一般情况下，我们只要修改Node所在的Auto Scaling Group的number就好了  
Kops的缺省设置是2，你可以把对应的数值设置成自己需要的数字。

2. 方法二：  
通过kops来修改  
```
kops edit ig nodes  
```  
把maxSize和minSize都设置成需要的值，然后更新  
```
kops update cluster --yes
kops rolling-update cluster
```
使用rolling-update可以保证在更新的时候业务不会中断。  

### 如何暂停集群  
有人可能会问，我希望不用的时候能把集群暂停，这样就不会使用很多的AWS系统资源了，这要怎么办。  
因为Auto Scaling Group的存在，如果直接stop对应的EC2实例，Auto Scaling Group会创建新的实例的取代，所以这个方法是不管用的。   
其实办法很简单，只要把对应的Auto Scaling Group的数值设置为0就好了  

同样可以在AWS中直接修改Master和Node所在的Auto Scaling Group，或者在Kops中修改  

注意在Kops中修改，需要调用如下的命令来获得Master所在group的名字  

```
$ kops get ig
Using cluster from kubectl context: staging.cluster-name.com

NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
master-us-west-2a	Master	m3.medium	0	0	us-west-2a
nodes			Node	t2.medium	0	0	us-west-2a
```


### 如何删除集群 
在集群不需要的时候，可以用kops删除集群：  
```
kops delete cluster --name ${NAME}
```  


# 注意：  
- 认证时长？ 
- 是否必须按照S3， 不安装可以不？  
- 修改配置文件 

# 如何使用kops 来操作kubernetes 集群  











