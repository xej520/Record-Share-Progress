# 经验:  
  >在安装部署ceph集群前，可以先去网上看看其他网友是如何安装的，找找其中的共同点，  
  说明这些是必须的安装的，
  做到心里有数后，才进行安装自己的ceph集群  
## 主要参考的文献如下:  
  http://docs.ceph.org.cn/start/  
  https://blog.csdn.net/styshoo/article/details/55471132  
  https://blog.csdn.net/Notzuonotdied/article/details/69668519  
  https://www.cnblogs.com/netmouser/p/6876846.html  
  https://www.cnblogs.com/ytc6/p/7388654.html  
# 当前系统环境
| IP地址 | 主机名 | 角色 | 网络模式  |系统|内核版本|cpu|memory|
| :------|:------|:-----|:----------|:---|:-----|:---|:----|
|172.16.91.165 |master|admin,mon|bridge|centos7|3.10.0-693.el7.x86_64|2|40G|
|172.16.91.166 |slave1|osd|bridge|centos7|3.10.0-693.el7.x86_64|2|40G|
|172.16.91.167 |slave2|osd|bridge|centos7|3.10.0-693.el7.x86_64|2|40G|
# 环境配置   

>注意：可以先将CentOS-Base.repo文件进行备份，如果后面利用yum进行安装包时，失败的话，可以将阿里云的CentOS-Base.repo删除，还原成备份的CentOS-Base.repo即可。

## 更新yum源(master节点)  
- 删除默认的yum源码(国外的yum源太慢了)
  >yum clean all  
  rm -rf /etc/yum.repos.d/*.repo

- 使用阿里云作为base源  
  >wget -O /etc/yum.repos.d/CentOS-Base.repo  http://mirrors.aliyun.com/repo/Centos-7.repo
- 下载阿里云的epel源  
  >wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo  
- 修改里面的系统版本为7.3.1611,当前用的centos的版本的的yum源可能已经清空了  
  >sed -i '/aliyuncs/d' /etc/yum.repos.d/CentOS-Base.repo  
sed -i '/aliyuncs/d' /etc/yum.repos.d/epel.repo  
sed -i 's/$releasever/7.3.1611/g' /etc/yum.repos.d/CentOS-Base.repo      

- 添加ceph的yum源  vim /etc/yum.repos.d/ceph.repo
  ```
  [ceph]

  name=ceph

  baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/x86_64/

  gpgcheck=0

  priority =1

  [ceph-noarch]

  name=cephnoarch

  baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/noarch/

  gpgcheck=0

  priority =1

  [ceph-source]

  name=Ceph source packages

  baseurl=http://mirrors.aliyun.com/ceph/rpm-jewel/el7/SRPMS

  gpgcheck=0

  priority=1
  ```  
## 禁用selinux(所有节点)
>sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

## 防火墙  (所有节点)
- 方案一： 关闭防火墙
  >systemctl stop firewalld
systemctl disable firewalld

## 安装NTP 服务(所有节点)  
  >yum install -y ntp ntpdate ntp-doc  
ntpdate 0.us.pool.ntp.org  
hwclock --systohc  
systemctl enable ntpd.service  
systemctl start ntpd.service  

## 更新/etc/hosts文件(master节点)  
```
172.16.91.165	master
172.16.91.166	slave1
172.16.91.167	slave2   
```


## 创建新的用户guxin(所有节点)  
- 创建新的用户guxin
  > useradd -d /home/guxin -m guxin  
passwd guxin
- 确保所有节点上新创建的用户guxin都有sudo权限  
  >echo "guxin ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/guxin
sudo chmod 0440 /etc/sudoers.d/guxin
sed -i s'/Defaults requiretty/#Defaults requiretty'/g /etc/sudoers  
## 接下来改成普通用户guxin操作  (master节点)
  > su - guxin
## 设置master节点上可以无密码访问其他node，设置ssh
- ssh-keygen
- ssh-copy-id guxin@slave1
- ssh-copy-id guxin@slave2
## 创建~/.ssh/config文件(master节点上)
- 创建config文件
  ```
  Host    master
      Hostname master
      User guxin
  Host    slave1
      Hostname slave1
      User guxin
  Host    slave2
      Hostname slave2
      User guxin
  ```  
## 优先级/首选项 (所有节点)
在centos系统中，要确保你的包管理器安装了优先级/首选项包且已经启动了  

    sudo yum install -y yum-plugin-priorities  
# 部署ceph  
## 安装ceph-deploy软件包  
- 在master节点执行下面的命令
  > sudo yum install -y yum-utils && sudo yum-config-manager --add-repo https://dl.fedoraproject.org/pub/epel/7/x86_64/ && sudo yum install --nogpgcheck -y epel-release && sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 && sudo rm /etc/yum.repos.d/dl.fedoraproject.org*  

- 更新软件库并安装 ceph-deploy(master节点)
  >sudo yum update && sudo yum install -y ceph-deploy

## 创建目录  
  用于保存ceph-deploy生成的配置文件和密钥对  
  
    mkdir my-cluster && cd my-cluster
## 创建集群  
1. 创建集群(在管理节点master上)  
    1. ceph-deploy new master  
    2. ls  
      ceph.conf  ceph-deploy-ceph.log  ceph.mon.keyring  
2. 把 Ceph 配置文件里的默认副本数从 3 改成 2 ，这样只有两个 OSD 也可以达到 active + clean 状态。把下面这行加入 [global] 段：  
  osd pool default size = 2  
3. 如果你有多个网卡，可以把 public network 写入 Ceph 配置文件的 [global] 段  
    >vim /home/guxin/my-cluster/ceph.conf  添加如下内容:  
  public network = {ip-address}/{netmask}  
4. 安装 Ceph  
    >ceph-deploy install master slave1 slave2  
  
5. sudo ceph --version  
    >ceph version 10.2.11 (e4b061b47f07f583c92a050d9e84b1813a35671e)

6. 配置初始 monitor(s)、并收集所有密钥：  
  ceph-deploy mon create-initial  
  完成上述操作后，当前目录里应该会出现这些密钥环：  
    - {cluster-name}.client.admin.keyring
    - {cluster-name}.bootstrap-osd.keyring
    - {cluster-name}.bootstrap-mds.keyring
    - {cluster-name}.bootstrap-rgw.keyring
## 添加OSD进程  
1. 添加两个 OSD 。为了快速地安装，这篇快速入门把目录而非整个硬盘用于 OSD 守护进程。登录到 Ceph 节点、并给 OSD 守护进程创建一个目录。  
  创建osd0的工作目录&nbsp;&nbsp;&nbsp;(在slave1节点上)  
    sudo mkdir /var/local/osd0  
  创建osd1的工作目录&nbsp;&nbsp;&nbsp;(在slave2节点上)   
    sudo mkdir /var/local/osd1  
  准备OSD&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(在master节点上)  
    ceph-deploy osd prepare slave1:/var/local/osd0 slave2:/var/local/osd1  
  激活OSD&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(master节点)  
    ceph-deploy osd activate slave1:/var/local/osd0 slave2:/var/local/osd1  
2. 用 ceph-deploy 把配置文件和 admin 密钥拷贝到管理节点和 Ceph 节点，这样你每次执行 Ceph 命令行时就无需指定 monitor 地址和 ceph.client.admin.keyring 了  
  ceph-deploy admin master slave1 slave2 
3. 确保你对 ceph.client.admin.keyring 有正确的操作权限  
  sudo chmod +r /etc/ceph/ceph.client.admin.keyring  
4. 检查集群的监控状态  
  ceph health  

        HEALTH_OK   
5. 查看集群的实时状态  
  ceph -w   
  ```
  cluster 19751c70-5213-4f33-8189-dd0557054cce
     health HEALTH_OK
     monmap e1: 1 mons at {master=172.16.91.165:6789/0}
            election epoch 3, quorum 0 master
     osdmap e10: 2 osds: 2 up, 2 in
            flags sortbitwise,require_jewel_osds
      pgmap v24: 64 pgs, 1 pools, 0 bytes data, 0 objects
            14153 MB used, 61569 MB / 75723 MB avail
                  64 active+clean

2018-09-01 04:45:38.680144 mon.0 [INF] pgmap v24: 64 pgs: 64 active+clean; 0 bytes data, 14153 MB used, 61569 MB / 75723 MB avail

  ```



# 安装中出现的问题  
## 1. Bad owner or permissions on /home/guxin/.ssh/config   [ceph_deploy][ERROR ] RuntimeError: connecting to host: slave1 resulted in errors: HostNotFound slave1  
  ls /home/guxin/.ssh/  

    -rw-rw-r-- 1 guxin guxin  168 Aug 31 05:30 config
  说明该config文件的权限是664, 因此需要将文件的权限改成600  
  sudo chmod 600  /home/guxin/.ssh/config  
## 2.  ** ERROR: error creating empty object store in /var/local/osd0: (13) Permission denied    
  ls /var/local  
  drwxr-xr-x 2 root root 85 Sep  1 04:16 osd0  
  很明显，创建osd0时，用的是guxin用户，但是，osd0的用户确是root，因此需要修改osd0文件的所有者以及组  
  在slave1节点上&nbsp;&nbsp;用root身份运行  
  chown -R ceph:ceph /var/local/osd0  
  在slave2节点上&nbsp;&nbsp;用root身份运行  
  chown -R ceph:ceph /var/local/osd1   
  **注意**   
  用户组就是ceph, 不是上面自己创建的用户guxin, 已经定死了，就是ceph














