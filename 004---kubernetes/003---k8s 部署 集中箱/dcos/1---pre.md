# 一、预先准备环境  
## 1. 准备vm服务器  
|**系统类型**|**IP**|**role**|**cpu**|**memory**|**hostname**|
|:---|:---|:---|:---|:---|:---|
|CentOS 7.4.1708|172.16.91.185|master|2|2G|master|
|CentOS 7.4.1708|172.16.91.186|worker|1|1G|slave1|
|CentOS 7.4.1708|172.16.91.187|worker|1|1G|slave2|  

## 2. 部署docker(所有节点) (针对的是ubuntu 16.04的docker部署，非centos的) 
- ubuntu 16.04 系统安装docker  
- centos 7.4 系统安装docker
### 2.1 ubuntu 版本的docker部署
#### 2.1.1 卸载旧版本(若存在) 
>apt-get remove docker docker-engine docker.io

#### 2.1.2 更新apt-get源  
>add-apt-repository  "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"  

>apt-get update

#### 2.1.3 安装apt的https支持包并添加gpg密钥  
    apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common  

>curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -  

#### 2.1.4 安装docker-ce  
##### 2.1.4.1 安装最新的稳定版  
>apt-get install -y docker-ce

##### 2.1.4.2 安装指定版本  
#获取版本列表
>apt-cache madison docker-ce
 
#指定版本安装(比如版本是17.09.1~ce-0~ubuntu)
>apt-get install -y docker-ce=17.09.1~ce-0~ubuntu
### 2.1.5 更新配置文件docker.sevice--->接收所有IP的数据包转发  
>    vi /lib/systemd/system/docker.service     
    #找到ExecStart=xxx，在这行上面加入一行，内容如下：(k8s的网络需要)  
    ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT

### 2.1.6 启动docker服务  
>systemctl daemon-reload  
service docker start
### 2.1.7 设计开机启动 docker服务  
>systemctl enable docker


### 2.2 centos 版本的docker部署   
#### 2.2.1 安装yum工具链  
>yum install -y yum-utils

#### 2.2.2 添加yum源 
两种方式：
-  国内yum源  
    yum-config-manager \\  
    --add-repo \\  
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo  
- 国外官方yum源  
     yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

#### 2.2.3 更新yum缓存  
>yum makecache fast
#### 2.2.4 安装docker-ce  
>yum install -y docker-ce
#### 2.2.5 更新配置文件docker.sevice--->接收所有IP的数据包转发
>vi /lib/systemd/system/docker.service     
    #找到ExecStart=xxx，在这行上面加入一行，内容如下：(k8s的网络需要)  
    ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
#### 2.2.6 设置开机启动docker服务  
>systemctl enable docker

#### 2.2.7 启动docker服务
>systemctl start docker 


## 3. 系统设置(所有节点)  
### 3.1 关闭、禁用防火墙(让所有机器之间都可以通过任意端口建立连接)
    systemctl stop firewalld.service #停止firewall
    systemctl disable firewalld.service #禁止firewall开机启动

    firewall-cmd --state #查看默认防火墙状态（关闭后显示notrunning，开启后显示running）
### 3.2 设置系统参数 - 允许路由转发，不对bridge的数据进行处理
    #写入配置文件
    $ cat <<EOF > /etc/sysctl.d/k8s.conf
    net.ipv4.ip_forward = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    
    #生效配置文件
    $ sysctl -p /etc/sysctl.d/k8s.conf



### 3.3 配置host文件
    #vi /etc/hosts 
    172.16.91.185	master
    172.16.91.186	slave1
    172.16.91.187	slave2




## 4. 准备二进制文件(所有节点)  
kubernetes的安装有几种方式，不管是kube-admin还是社区贡献的部署方案都离不开这几种方式：  
- 使用现成的二进制文件  
- 使用源码编译安装  
- 使用镜像的方式运行  
### 4.1 使用现成的二进制文件  
>直接从官方或其他第三方下载，就是kubernetes各个组件的可执行文件。拿来就可以直接运行了。不管是centos，ubuntu还是其他的linux发行版本，只要gcc编译环境没有太大的区别就可以直接运行的。使用较新的系统一般不会有什么跨平台的问题。

### 4.2 使用源码编译安装（更新了源码的话，就必须使用这种了） 
>编译结果也是各个组件的二进制文件，所以如果能直接下载到需要的二进制文件基本没有什么编译的必要性了。

### 4.3 使用镜像的方式运行  
>同样一个功能使用二进制文件提供的服务，也可以选择使用镜像的方式。就像nginx，像mysql，我们可以使用安装版，搞一个可执行文件运行起来，也可以使用它们的镜像运行起来，提供同样的服务。kubernetes也是一样的道理，二进制文件提供的服务镜像也一样可以提供。  

从上面的三种方式中其实使用镜像是比较优雅的方案，容器的好处自然不用多说。但从初学者的角度来说容器的方案会显得有些复杂，不那么纯粹，会有很多容器的配置文件以及关于类似二进制文件提供的服务如何在容器中提供的问题，容易跑偏。  
 所以我们这里使用二进制的方式来部署。二进制文件已经这里备好，大家可以打包下载，把下载好的文件放到每个节点上，放在哪个目录随你喜欢，放好后最好设置一下**环境变量$PATH**，方便后面可以直接使用命令。(科学上网的同学也可以自己去官网找找)
[kubernetes1.9.0 二进制文件下载地址](https://pan.baidu.com/s/1BBqsU4Qj-hdh7t4u9tu8wA)  



## 5. 准备配置文件(所有节点)  
上一步我们下载了kubernetes各个组件的二进制文件，这些可执行文件的运行也是需要添加很多参数的，包括有的还会依赖一些配置文件。现在我们就把运行它们需要的参数和配置文件都准备好。
### 5.1 下载配置文件
    #到home目录下载项目
    $ cd
    $ git clone https://github.com/liuyi01/kubernetes-starter.git
    #看看git内容
    $ cd ~/kubernetes-starter && ls
### 5.2 文件说明
- gen-config.sh 
>shell脚本，用来根据每个同学自己的集群环境(ip，hostname等)，根据下面的模板，生成适合大家各自环境的配置文件。生成的文件会放到target文件夹下。
- kubernetes-no-ca 
>简易版kubernetes配置模板（剥离了认证授权）。 适合刚接触kubernetes的同学，首先会让大家在和kubernetes初次见面不会印象太差（太复杂啦~~），再有就是让大家更容易抓住kubernetes的核心部分，把注意力集中到核心组件及组件的联系，从整体上把握kubernetes的运行机制
- kubernetes-yes-ca  
>在simple基础上增加认证授权部分。大家可以自行对比生成的配置文件，看看跟simple版的差异，更容易理解认证授权的（认证授权也是kubernetes学习曲线较高的重要原因）
- service-config  
>这个先不用关注，它是我们曾经开发的那些微服务配置。 等我们熟悉了kubernetes后，实践用的，通过这些配置，把我们的微服务都运行到kubernetes集群中。

### 5.3 生成配置
这里会根据大家各自的环境生成kubernetes部署过程需要的配置文件。 在**每个节点上**都生成一遍，把所有配置都生成好，后面会根据节点类型去使用相关的配置。  

    #cd到之前下载的git代码目录
    $ cd ~/kubernetes-starter
    #编辑属性配置（根据文件注释中的说明填写好每个key-value）
    $ vi config.properties
    #生成配置文件，确保执行过程没有异常信息
    $ ./gen-config.sh simple
    #查看生成的配置文件，确保脚本执行成功
    $ find target/ -type f
    target/all-node/kube-calico.service
    target/master-node/kube-controller-manager.service
    target/master-node/kube-apiserver.service
    target/master-node/etcd.service
    target/master-node/kube-scheduler.service
    target/worker-node/kube-proxy.kubeconfig
    target/worker-node/kubelet.service
    target/worker-node/10-calico.conf
    target/worker-node/kubelet.kubeconfig
    target/worker-node/kube-proxy.service
    target/services/kube-dns.yaml  

### 5.4 执行gen-config.sh常见问题：  
1. gen-config.sh: 3: gen-config.sh: Syntax error: "(" unexpected  
- bash版本过低，运行：bash -version查看版本，如果小于4需要升级  
- 不要使用 sh gen-config.sh的方式运行（sh和bash可能不一样哦）  
2. config.properties文件填写错误，需要重新生成 再执行一次./gen-config.sh simple即可，不需要手动删除target