# 一、在AWS上部署kubernetes  
主要参考文献：  
https://github.com/kubernetes/kops/blob/master/docs/aws.md  
https://kubernetes.io/docs/setup/custom-cloud/kops/  
https://blog.csdn.net/cloudvtech/article/details/80539086  
https://blog.csdn.net/qq_36348557/article/details/79795288
https://my.oschina.net/geekidentity/blog/1635540  
https://jackiechen.org/2018/04/23/install-kubernetes-on-aws-with-kops/  

说明：  
在AWS上部署kuberntes 可以使用下面的两个工具？  
- kubeadm 
- kops(是在建立在kubeadm的基础之上的) 

两种方式对比：  
- kubeadm需要预安装kubectl和kubelet，kops需要预安装kubectl;
- kubeadm只负责在部署Kubenetes在computers running Linux，不负责infrastructure orchestration，kops负责infrastructure orchestration和Kubenetes orchestration;
- kubeadm以容器化的方式运行Kubernetes服务，kops以普通服务的方式运行Kubernetes;
- Integration with kubeadm kops now uses kubeadm for some RBAC related functionality;

------------------



服务器选择？  
- 国内服务器的话，选择北京区域，不要选择 宁夏区域，因为宁夏区域目前没有kops 
- 

本文使用kops命令进行安装部署    
官网文档：  
https://kubernetes.io/docs/setup/custom-cloud/kops/  

# 二、准备工作
## 2.1 如何切换成root用户？  
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
 
## 2.2 更新配置环境 
```
yum install wget
wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum install -y ./epel-release-latest-*.noarch.rpm
yum -y update
yum -y install python-pip
```
## 2.3 安装kubectl
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
```

## 2.4 安装aws 命令行工具 
```
pip install --upgrade pip
pip install awscli --upgrade --user
export PATH=/usr/local/bin:$PATH
```  
注意，我这里是将kubetl, kops命令部署到了/usr/local/bin目录下， 需要添加到环境变量PATH里； 
根据自己的实际情况，更新export PATH的值  

## 2.5 centos上安装 kops工具  
```
wget https://github.com/kubernetes/kops/releases/download/1.10.0/kops-linux-amd64
chmod +x kops-linux-amd64
mv kops-linux-amd64 /usr/local/bin/kops
```  

## 2.6 配置AWS资源 
我们需要一个AWS账号 用于运行kops， 该用户需要具有以下的权限:  
```
AmazonEC2FullAccess
AmazonRoute53FullAccess
AmazonS3FullAccess
IAMFullAccess
AmazonVPCFullAccess
```
使用aws cli 来创建用户组、用户 以及  access key
### 2.6.1 创建一个新的AWS Group 、给AWS Group 赋予权限 

0. 登陆原来的AWS账号  
  替换下面的xxxxxx, yyyyyyy, zzzz  
  具体方法，可以参考"遇到的问题"
  ```
  [root@ip-172-31-36-99 ~]# aws configure
  AWS Access Key ID [****************-k8s]: xxxxxxxxxxxxxxxxxxx
  AWS Secret Access Key [****************-k8s]: yyyyyyyyyyyyyyyyyyyy
  Default region name [us-west-2]: zzzzz
  Default output format [json]: json
  ```

1. 创建用户组  
  ```
  aws iam create-group --group-name kops
  ``` 
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/5C34D79558B140CDB6DB26CE5FFE3771/20542)  
2. 给用户组设置权限 
  ```
  aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
  aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
  aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
  aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
  aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
  ```  
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/BCFAD32D5B5D45EFB986FBB9ED6C852B/20545)  


### 2.6.2 创建用户  
1. 创建用户  
  ```
  aws iam create-user --user-name kops
  ```    
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/9433F7A2BCDC49AD80FE8B84DBE76118/20548)  

2. 将用户添加到用户组里  
  ```
  aws iam add-user-to-group --user-name kops --group-name kops  
  ```
### 2.6.3 创建access key  
```
[root@ip-172-31-36-99 ~]# aws iam create-access-key --user-name kops
{
    "AccessKey": {
        "UserName": "kops", 
        "Status": "Active", 
        "CreateDate": "2018-10-18T01:42:34Z", 
        "SecretAccessKey": "l7SWH7nFW94arHc+9WbmIJ7XzxbitALSN1zzzAyt", 
        "AccessKeyId": "AKIAI2E4ZKXRZDQGJYJA"
    }
}

```  
请记录下这里生成的SecretAccessKey 和 AccessKeyID，  需要配置到客户端，   
并且手动导入环境变量  

### 2.6.4 使用aws-cli登陆这个新用户 
1. 更新awscli的配置, 让它使用新创建的kops用户的密钥：
  ```
  # configure the aws client to use your new IAM user
  aws configure           # Use your new access and secret key here
  aws iam list-users      # you should see a list of all your IAM users here
  ```  
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/06D4882514DD4641AB8A9667960E7C06/20553)  


2. 将kops用户的密钥导出到命令行的环境变量： 
  ```
  # Because "aws configure" doesn't export these vars for kops to use, we export them now
  export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
  export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
  export AWS_REGION=$(aws configure get region)
  ```  
3. 生成SSH密钥： 
  ```
  ssh-keygen
  ``` 
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/B0AA134EC2F54367AE6AD1032D2186F5/20557)  

# 三、创建集群  
## 3.1 创建一个S3存储  
Kops把K8s集群的配置存储在AWS的S3中，每一个集群的配置对应于一个S3文件，  
所有我们创建一个S3的bucket用于存储集群的配置  
1. 设置BUCKET环境变量
  注意：  
  export BUCKET=XXX.k8s.local 修改成自己的值，替换XXX， 其他建议不要动。
  ```
  export BUCKET=cluster.k8s.local

  ```  
2. 创建一个bucket,  根据自己的实际情况，替换us-west-2
  ```
  aws s3api create-bucket \
      --bucket $BUCKET \
      --region us-west-2 \
      --create-bucket-configuration  LocationConstraint=us-west-2
  ```

3. 使其生效  
  ```
  aws s3api put-bucket-versioning --bucket $BUCKET  --versioning-configuration Status=Enabled
  ```  
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/51729D01A2FF41EFB57F9AAE593122EF/20559)  


下面开始创建kubernetes 集群
## 3.2 创建集群  
### 3.2.1 配置环境变量，定义集群的名字和配置的URL 
```
export NAME=test.k8s.local
export KOPS_STATE_STORE=s3://$BUCKET
```
注意根据自己的需求，重新设置环境变量NAME的值  

### 3.2.2 创建集群配置文件  
1. 获取可用的zones    
  ```
  aws ec2 describe-availability-zones --region us-west-2
  ```  
  当前在使用 us-west-2的region可用的az是这三个  
  ```
  [root@ip-172-31-36-99 ~]# aws ec2 describe-availability-zones --region us-west-2
  {
      "AvailabilityZones": [
          {
              "State": "available", 
              "ZoneName": "us-west-2a", 
              "Messages": [], 
              "RegionName": "us-west-2"
          }, 
          {
              "State": "available", 
              "ZoneName": "us-west-2b", 
              "Messages": [], 
              "RegionName": "us-west-2"
          }, 
          {
              "State": "available", 
              "ZoneName": "us-west-2c", 
              "Messages": [], 
              "RegionName": "us-west-2"
          }
      ]
  }

  ```

因此，从上面的几个选项中，选择一个可用的zones即可   
我这里选择us-west-2a  

2. 创建集群时用到的配置文件 

  ```
  kops create cluster \
      --zones us-west-2a \
      ${NAME}
  ```  
注意
- 这一步只是生成了集群的配置文件，并存储在S3中。  
- 需要替换成自己的zones值  
  如果此时报下面的错误：  
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/C3BB65F42FD34FB895814322F9A1FD91/20562)  
  可以直接使用下面的命令来替换 ：(写成绝对路径)  
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/65A7E639D16140F2818DC7AC1A15B895/20565)  

  执行成功后，最后会打印下面的信息：  
  ```
  Must specify --yes to apply changes

  Cluster configuration has been created.

  Suggestions:
  * list clusters with: kops get cluster
  * edit this cluster with: kops edit cluster test.k8s.local
  * edit your node instance group: kops edit ig --name=test.k8s.local nodes
  * edit your master instance group: kops edit ig --name=test.k8s.local master-us-west-2a

  Finally configure your cluster with: kops update cluster test.k8s.local --yes

  ```


3. 查看刚才生成的配置  
  ```
  kops get cluster ${NAME}  
  ```  
4. 查看节点信息(选做)  
  ```
  [root@ip-172-31-36-99 bin]#/usr/local/bin/kops edit ig --name=test.k8s.local nodes

  apiVersion: kops/v1alpha2
  kind: InstanceGroup
  metadata:
    creationTimestamp: 2018-10-18T02:06:54Z
    labels:
      kops.k8s.io/cluster: test.k8s.local
    name: nodes
  spec:
    image: kope.io/k8s-1.10-debian-jessie-amd64-hvm-ebs-2018-08-17
    machineType: t2.medium
    maxSize: 2
    minSize: 2
    nodeLabels:
      kops.k8s.io/instancegroup: nodes
    role: Node
    subnets:
    - us-west-2a

  ```
5. 查看master信息
  ```
  [root@ip-172-31-36-99 bin]# /usr/local/bin/kops edit ig --name=test.k8s.local master-us-west-2a

  apiVersion: kops/v1alpha2
  kind: InstanceGroup
  metadata:
    creationTimestamp: 2018-10-18T02:06:54Z
    labels:
      kops.k8s.io/cluster: test.k8s.local
    name: master-us-west-2a
  spec:
    image: kope.io/k8s-1.10-debian-jessie-amd64-hvm-ebs-2018-08-17
    machineType: m3.medium
    maxSize: 1
    minSize: 1
    nodeLabels:
      kops.k8s.io/instancegroup: master-us-west-2a
    role: Master
    subnets:
    - us-west-2a
  ```


6. 将网络插件设置成calico模式
  ```
  /usr/local/bin/kops edit cluster ${NAME}
  ``` 
  更新前：   
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/CCAB851CAE2D4CA0A218F4C35F4AED0A/20568)  
  更新后：   
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/76CC11A8D24C4E66AEBDFC2E77E2956B/20570)  

### 3.2.3 开始真正部署kubernetes集群    
1. 创建kubernetes集群  
  ```
  kops update cluster ${NAME} --yes  
  ```  
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/EEC1567CD204416780A57F441904D133/20579) 
  认证过程，自动生成  
  参数：  
  --yes， 表示，显示具体的执行过程，都做了哪些工作  
  缺省的情况下，kops会创建所有对应的AWS资源，包含VPC，子网，EC2，Auto Scaling Group，ELB，安全组等等。  
  如果需要安装在特定的子网，在创建集群时可以指定子网的id。另外，也支持跨AZ的HA配置    

2. 查看集群状态  
  集群安装好之后，需要几分钟时间启动，我们可以用kubectl来查看一下状态(Kops会自动把cluster的配置写到~/.kube/config 文件中作为缺省配置)：
  ```
  kubectl cluster-info  
  ```  
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/77C5253D9F834F099AF195D9A8AFAB43/20574)  
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/77C5253D9F834F099AF195D9A8AFAB43/20574)  


### 3.2.4 如何扩展集群(新增节点到集群)    
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
  /usr/local/bin/kops edit ig nodes  
  ```  
  把maxSize和minSize都设置成需要的值，然后更新  
  ```
  /usr/local/bin/kops update cluster --yes
  /usr/local/bin/kops rolling-update cluster
  ```
  使用rolling-update可以保证在更新的时候业务不会中断。  

### 3.2.5 如何暂停集群  
有人可能会问，我希望不用的时候能把集群暂停，这样就不会使用很多的AWS系统资源了，这要怎么办。  
因为Auto Scaling Group的存在，如果直接stop对应的EC2实例，Auto Scaling Group会创建新的实例的取代，所以这个方法是不管用的。   
其实办法很简单，只要把对应的Auto Scaling Group的数值设置为0就好了  

同样可以在AWS中直接修改Master和Node所在的Auto Scaling Group，或者在Kops中修改  

注意在Kops中修改，需要调用如下的命令来获得Master所在group的名字  

```
$ /usr/local/bin/kops get ig
Using cluster from kubectl context: staging.cluster-name.com

NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
master-us-west-2a	Master	m3.medium	0	0	us-west-2a
nodes			Node	t2.medium	0	0	us-west-2a
```


### 3.2.6 如何删除集群 
在集群不需要的时候，可以用kops删除集群：  
```
/usr/local/bin/kops delete cluster --name ${NAME}  --yes
```  
![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/D33403F9116041599351F3CC28B0EF1A/20677)  

# 四、遇到的问题
1. An error occurred (InvalidClientTokenId) when calling the CreateUser operation: The security token included in the request is invalid  
  是因为使用aws，kops命令时，需要授权。  
  解决措施：  
  创建访问密钥: 步骤如下：  
  在用户管理页面--->我的安全凭证--->访问密钥(访问密钥ID和秘密访问密钥)  
  这里面有。  
  如果没有的话，点击下面的"创建新的访问密钥" , 可以下载到本地，rootkey.csv文件  
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/7F9B676803B943E295C753406AECDAEB/20537)  
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/03708842EA134A60B16C42D9B061E11E/20535)  
  ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/F3A0AD8FD6324F96BD4B4E80B9823380/20540)  


# 五、关于“认证“  
## 5.1 什么是IAM？  
    权限让您能够指定对 AWS 资源的访问权限。权限授予 IAM 实体 (用户、组和角色)，这些实体在开始时默认没有任何权限。也就是说，除非您授予需要的权限，否则 IAM 实体在 AWS 中就无法进行任何操作。要为实体提供权限，您可以附加一条指定访问类型、可以执行的操作以及可以操作的资源的策略。此外，您还可以针对允许访问或拒绝访问指定必须设置的条件。
https://amazonaws-china.com/cn/iam/details/manage-permissions/  


# 六、Authentication

Kops has support for configuring authentication systems.  This should not be used with kubernetes versions
before 1.8.5 because of a serious bug with apimachinery [#55022](https://github.com/kubernetes/kubernetes/issues/55022).

## 6.1 kopeio authentication

If you want to experiment with kopeio authentication, you can use
`--authentication kopeio`.  However please be aware that kopeio authentication
has not yet been formally released, and thus there is not a lot of upstream
documentation.

Alternatively, you can add this block to your cluster:

```
authentication:
  kopeio: {}
```

For example:

```
apiVersion: kops/v1alpha2
kind: Cluster
metadata:
  name: cluster.example.com
spec:
  authentication:
    kopeio: {}
  authorization:
    rbac: {}
```

## 6.2 AWS IAM Authenticator

If you want to turn on AWS IAM Authenticator, you can add this block 
to your cluster running Kubernetes 1.10 or newer:

```
authentication:
  aws: {}
```

For example:

```
apiVersion: kops/v1alpha2
kind: Cluster
metadata:
  name: cluster.example.com
spec:
  authentication:
    aws: {}
  authorization:
    rbac: {}
```

Once the cluster is up, or after you've performed a rolling update to an existing cluster with `kops rolling-update cluster ${CLUSTER_NAME} --instance-group-roles=Master --force --yes`, you will need to create the AWS IAM authenticator
config as a config map. (This can also be done when boostrapping a cluster using addons)
For more details on AWS IAM authenticator please visit [kubernetes-sigs/aws-iam-authenticator](https://github.com/kubernetes-sigs/aws-iam-authenticator)

Example config:

```
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: kube-system
  name: aws-iam-authenticator
  labels:
    k8s-app: aws-iam-authenticator
data:
  config.yaml: |
    # a unique-per-cluster identifier to prevent replay attacks
    # (good choices are a random token or a domain name that will be unique to your cluster)
    clusterID: my-dev-cluster.example.com
    server:
      # each mapRoles entry maps an IAM role to a username and set of groups
      # Each username and group can optionally contain template parameters:
      #  1) "{{AccountID}}" is the 12 digit AWS ID.
      #  2) "{{SessionName}}" is the role session name.
      mapRoles:
      # statically map arn:aws:iam::000000000000:role/KubernetesAdmin to a cluster admin
      - roleARN: arn:aws:iam::000000000000:role/KubernetesAdmin
        username: kubernetes-admin
        groups:
        - system:masters
      # map EC2 instances in my "KubernetesNode" role to users like
      # "aws:000000000000:instance:i-0123456789abcdef0". Only use this if you
      # trust that the role can only be assumed by EC2 instances. If an IAM user
      # can assume this role directly (with sts:AssumeRole) they can control
      # SessionName.
      - roleARN: arn:aws:iam::000000000000:role/KubernetesNode
        username: aws:{{AccountID}}:instance:{{SessionName}}
        groups:
        - system:bootstrappers
        - aws:instances
      # map federated users in my "KubernetesAdmin" role to users like
      # "admin:alice-example.com". The SessionName is an arbitrary role name
      # like an e-mail address passed by the identity provider. Note that if this
      # role is assumed directly by an IAM User (not via federation), the user
      # can control the SessionName.
      - roleARN: arn:aws:iam::000000000000:role/KubernetesAdmin
        username: admin:{{SessionName}}
        groups:
        - system:masters
      # each mapUsers entry maps an IAM role to a static username and set of groups
      mapUsers:
      # map user IAM user Alice in 000000000000 to user "alice" in "system:masters"
      - userARN: arn:aws:iam::000000000000:user/Alice
        username: alice
        groups:
        - system:masters
```




