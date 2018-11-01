__基于kubeadm+离线方式部署kubernetes v1.9.0__  
参考文献如下：  
http://blog.51cto.com/bestlope/2151855?source=dra  
https://www.jianshu.com/p/a4847af544de  
https://segmentfault.com/a/1190000011764684

# 一、部署背景 
&ensp;&ensp;&ensp;&ensp;由于近期要研究分析Service Catalog， 需要搭建一个对应的k8s集群， 选择的版本号是v1.9.0  

# 二、环境介绍  
|**系统类型**|**IP**|**role**|**cpu**|**memory**|**hostname**|
|:---|:---|:---|:---|:---|:---|
|CentOS 7.4.1708|172.16.91.155|master|4|2G|master|
|CentOS 7.4.1708|172.16.91.156|worker|2|1G|slave1|
|CentOS 7.4.1708|172.16.91.157|worker|2|1G|slave2|  

# 三、组件部署方式说明  
1.  组件部署说明  

    |组件名称|版本|部署方式|维护方式|
    |:---|:---|:---|:---|
    |kubeadm|v1.9.0|rpm||
    |kubelet|v1.9.9|rpm|systemd|
    |kubectl|v1.9.0|rpm||
    |kube-apiserver|v1.9.0|kubeadm|pod|
    |kube-scheduler-master|v1.9.0|kubeadm|pod|
    |kube-controller-manager-master|v1.9.0|kubeadm|pod|


2. 整体部署过程介绍(做到心里有底)   
    安装主要过程：  
    - 部署docker  
    - 导入/下载k8s镜像  
    - 部署kubeadm, kubelet, kubectl 
    - 初始化集群(master节点)  
    - 部署k8s网络(采用Calico方案)  
    - 增加节点(扩容)


# 四、安装环境准备工作(所有节点)  

## 4.1 添加基础相关依赖包(所有节点)  
```
yum install -y epel-release
yum install -y yum-utils device-mapper-persistent-data lvm2 net-tools conntrack-tools wget vim  ntpdate libseccomp libtool-ltdl 
```

## 4.2 主机映射(所有节点)  
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

172.16.91.215	master
172.16.91.216	slave1
172.16.91.217	slave2
```

## 4.3 ssh免密码登陆(在master节点上 )  
 
- ssh-keygen   
- ssh-copy-id root@slave1    
- ssh-copy-id root@slave2    

## 4.4 关闭防火墙(所有节点)  
- systemctl stop firewalld  
- systemctl disable firewalld

## 4.5 关闭Swap(所有节点)    
- swapoff -a 
- sed -i 's/.*swap.*/#&/' /etc/fstab  
![swap](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/8F27F98AC450428E9431EFB599730D75/21648)  
防止kubeadm初始化时，报如下错误信息：  
![kubeadm init error Swap](https://upload-images.jianshu.io/upload_images/12975960-df4e448ebd829393.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/933/format/webp)  
## 4.6 设置内核(所有节点)   
### 4.6.1 设置netfilter模块    
```
modprobe br_netfilter
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl -p /etc/sysctl.d/k8s.conf
ls /proc/sys/net/bridge
```   
设置目的：防止kubeadm报路由警告信息  

### 4.6.2 打开ipv4的转发功能 (所有节点)  
```
# 执行下面的命令
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf  
# 保存执行
sysctl -p  
```
![net.ipv1.ip_forward](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/D546799854F84D4F86EC802E8F6C88B0/21650)  

如果不打开的话，在将从节点加入到集群时，会报以下的问题？  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/4764194B67B84BDDBF62EACFA5C2FEBE/20362)  

### 4.6.3 更新内核参数  
```
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
echo "* soft nproc 65536"  >> /etc/security/limits.conf
echo "* hard nproc 65536"  >> /etc/security/limits.conf
echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf
echo "* hard memlock  unlimited"  >> /etc/security/limits.conf  
或者  
echo "* soft nofile 65536" >> /etc/security/limits.conf && 
echo "* hard nofile 65536" >> /etc/security/limits.conf && 
echo "* soft nproc 65536"  >> /etc/security/limits.conf &&  
echo "* hard nproc 65536"  >> /etc/security/limits.conf && 
echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf && 
echo "* hard memlock  unlimited"  >> /etc/security/limits.conf  
```

## 4.6.4 关闭Selinux(所有节点)   
```
setenforce  0 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/selinux/config  
```
![selinux设置](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/0DFF2D2400774344A7CB044E7ABAA343/21652)  
## 4.7 配置ntp(所有节点)   
```
systemctl enable ntpdate.service
echo '*/30 * * * * /usr/sbin/ntpdate time7.aliyun.com >/dev/null 2>&1' > /tmp/crontab2.tmp
crontab /tmp/crontab2.tmp
systemctl start ntpdate.service
```
### 4.7.1 发现系统时间跟实际时间不对，如何解决 
```
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime  
ntpdate us.pool.ntp.org  
date
```    

# 五、下载部署包  
1. 下载部署包(物理机操作)  
    链接：https://pan.baidu.com/s/1fwBxEzOdtD5WpFlo_kMmCw 密码：zfup   
    本人是下载到/root目录下  
    ![k8s部署包](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/6A1D72E1FCFD459D9566B2F56EC70D76/21712)  
2. 将部署包传输到其他从节点上去(master节点)
    ```
    scp k8s.tar.gz slave1:/root/
    scp k8s.tar.gz slave2:/root/
    ```

3. 解压(所有节点)  
    ```
    tar -zxvf k8s.tar.gz 
    ```  
    ![传输部署包](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/3147DD5606784D31B80B0CDCB014FDBD/21655)  
# 六、 部署  

## 6.1 部署docker 
具体可以参考其他文章，  
目前使用的版本是:  
```
[root@slave2 ~]# docker version
Client:
 Version:      17.03.2-ce
 API version:  1.27
 Go version:   go1.7.5
 Git commit:   f5ec1e2
 Built:        Tue Jun 27 02:21:36 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.03.2-ce
 API version:  1.27 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   f5ec1e2
 Built:        Tue Jun 27 02:21:36 2017
 OS/Arch:      linux/amd64
 Experimental: false
```

### 6.1.1 添加镜像加速器(所有节点) 
如果没有的话，可以在阿里云上注册，获取自己的镜像加速器； 
```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxxxx.mirror.aliyuncs.com"]
}
EOF
```  

### 6.1.2 启动docker服务(所有节点)   
```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 6.2 导入镜像(所有节点)  
```
cd /root/k8s/image
find . -name "*.tar" -exec docker image load -i {} \; 
find . -name "*.tar.gz" -exec docker image load -i {} \;
```  
![导入镜像](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/753D5D5394AF43EEBA7FA21192EE6149/21657)   
![slave1导入镜像](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/FD4BDFE9CEE547DB985F349048D7B347/21662)  
![slave2导入镜像](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/B6A69BB8A6424A369941F7A786088523/21659)  
![load calico image](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/0726DF930A5D417FBFC1868D89A69D0A/21697)  

## 6.3 部署kubeadm,kubectl, kubelet通过安装RPM包(所有节点)
```  
cd /root/k8s/rpm
rpm -ivh socat-1.7.3.2-2.el7.x86_64.rpm
rpm -ivh kubernetes-cni-0.6.0-0.x86_64.rpm  kubelet-1.9.9-9.x86_64.rpm  kubectl-1.9.0-0.x86_64.rpm
rpm -ivh kubeadm-1.9.0-0.x86_64.rpm
```
![master节点上rpm kubeadm、kubelet、 kubectl](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/137E100A3402490B98D06D762251CB38/21667)  
![slave1节点上rpm kubeadm、kubelet、 kubectl](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/9E75664B76E94D43B82659683340F052/21669)    
![slave2节点上rpm kubeadm、kubelet、 kubectl](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/6F0B9609841F4DDD84558373311615E7/21671)  
### 6.3.1 更新kubelet配置文件(所有节点)  

0. 查看一下docker的Cgroup Driver的值 ？  
```
docker info | grep "Cgroup Driver"  
```
![Cgroup Driver](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/201C03F651C44AB58E78FAC2B99E8E59/21674)  

1. 更新配置文件(所有节点)  
    ```
    sed -i 's#systemd#cgroupfs#g' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
    ```

2. 重新启动kubelet服务(所有节点)  
    ```  
    systemctl daemon-reload  
    systemctl enable kubelet  
    ```  
    ![更新kubelet配置文件](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/D103B7BB8F3243528E3DC8211346599C/21676)
3. 命令部署效果：(master节点上部署即可)(选做)   
    ```
    yum install -y bash-completion
    source /usr/share/bash-completion/bash_completion
    source <(kubectl completion bash)
    echo "source <(kubectl completion bash)" >> ~/.bashrc 
    ```

## 6.4 初始化集群(master节点)  
```
kubeadm init --kubernetes-version=v1.9.0 --pod-network-cidr=10.224.0.0/16 --token-ttl=0 --ignore-preflight-errors=all
``` 
 
__初始化正确结果，打印信息如下：__ 
![kubedam init成功](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/B87BB39A03074539942AFE1A9CFFABB7/21685)  

### 6.4.1 初始化时，报[kubelet-check] It seems like the kubelet isn't running or healthy. 
如果初始化时，始终报这个错；  
- 方法一：可以参考下面的文章    
https://segmentfault.com/a/1190000011707194  
- 方法二：查看master节点上kubelet进程是否正常启动(master节点操作)  
    - journalctl -u kubelet -n100
    - rm -rf  /etc/kubernetes/pki  
    - systemctl restart kubelet
    - kubeadm reset 
    - kubeadm init --kubernetes-version=v1.9.0 --pod-network-cidr=10.224.0.0/16 --token-ttl=0 --ignore-preflight-errors=all  
![kubelet-check](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/2AC11FB0EC014526A1F3C8FA63F9B153/21679)  
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/E0CB62271A6240B8B190D83F66B8AA8E/21681)  
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/D67013275AFE43B2B1D1061582E99694/21683)

### 6.4.2 若初始化失败时的解决措施(2种方式) (master节点) 
- 方式一(推荐这种方式简单明了)：  
    >kubeadm reset

- 方式二：  
    >rm -rf /etc/kubernetes/*.conf  
    rm -rf /etc/kubernetes/manifests/*.yaml  
    docker ps -a |awk '{print $1}' |xargs docker rm -f  
    systemctl  stop kubelet  

### 6.4.2 配置kubectl的认证信息(master节点)
配置kubectl的配置文件  
- 若是非root用户  
    >mkdir -p $HOME/.kube  
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  
    sudo chown $(id -u):$(id -g) $HOME/.kube/config  
- 若是root用户  
    >echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile  
    source ~/.bash_profile  
![创建kubectl配置文件](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/ACF2D97A47A94D999AE582D46A436803/21687)  
做了这一步操作后，就不会报类似这样的错误了：  
```
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```  

### 6.4.3 简单测试下 
- 查看master节点状态  
    kubectl get node   
- 查看pod资源情况  
    kubectl get pod -n kube-system -o wide    
- 查看组件运行状态  
    kubectl get componentstatus  
- 查看kubelet运行状况  
    systemctl status kubelet  
![集群状态](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/30F27C9D27F24148B6A2F50164F34828/21689)  


## 6.5 k8s网络部署，安装calico插件， 从而实现pod间的网络通信   
1. 修改calico.yaml中的CALICO_IPV4POOL_CIDR值  
    ![更新CALICO_IPV4POOL_CIDR](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/BC3120A46E5C4B72A74B21CA45DFAFEC/21692)  

2. 由于提供的etcd镜像跟calico.yaml中定义的不同，需要重新打一个tag(master节点)  
    ```
    docker tag 1406502a6459  quay.io/coreos/etcd:v3.1.10
    ```
3. 部署calico服务  
    ```
    kubectl create -f calico.yaml   
    ```

4. 查看pod状态 
    ```
    kubectl get pod -n kube-system  
    ```

5. 查看节点状态  
    ```
    kubectl get node   
    ```  
    ![集群状态](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/42C44EE34C2F447A908BB9BA16BD18EA/21702)  

## 6.6 扩容集群节点(将其他从节点slave1,slave2添加到集群里)  
1. 分别登陆到slave1， slave2上，运行下面的命令即可了(注意，要改成自己的)  
    ```
    kubeadm join 172.16.91.135:6443 --token yj2qxf.s4fjen6wgcmo506w --discovery-token-ca-cert-hash sha256:6d7d90a6ce931a63c96dfe9327691e0e6caa3f69082a9dc374c3643d0d685eb9
    ```  
    假如：忘记上面的token，可以使用下面的命令，找回（master节点上执行） 
    ```
    kubeadm token create --print-join-command
    ```
    ![join slave1](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/686F3915D3474DBFA23B61128E2098EF/21704)  
2. 再次查看pod的状态 
    ```
    kubectl get pods --all-namespaces -owide
    ```
3. 查看节点状态 
    ``` 
    kubectl get node
    ```  
    ![node status](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/C0A03C96D9EB4B7AB98C7195757B2D5C/21707)  

# 七、dns服务测试  
1. 准备测试用的yaml， pod-for-dns.yaml
    ```
    apiVersion: v1
    kind: Pod
    metadata:
    name: dns-test
    namespace: default
    spec:
    containers:
    - image: busybox:1.28.4
        command:
        - sleep
        - "3600"   
        imagePullPolicy: IfNotPresent
        name: dns-test
    restartPolicy: Always

    ```  
    注意：busybox的版本号，有些版本号测试失败  
2. 创建pod 
    ```
    kubectl create -f pod-for-dns.yaml
    ```
3. 测试dns服务  
    ```
    [root@master ~]# kubectl exec dns-test -- nslookup kubernetes
    Server:    10.96.0.10
    Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

    Name:      kubernetes
    Address 1: 10.96.0.1 webapp.default.svc.cluster.local

    ```
    ![test dns](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/C2D51CF928714288ACCB15E307AF2B9A/21709)  












