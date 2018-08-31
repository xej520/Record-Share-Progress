# 当前系统环境
| IP地址 | 主机名 | 角色 | 网络模式  |系统|内核版本|cpu|memory|
| :------|:------|:-----|:----------|:---|:-----|:---|:----|
|172.16.91.165 |master|admin,mon|bridge|centos7|3.10.0-693.el7.x86_64|2|40G|
|172.16.91.166 |slave1|osd|bridge|centos7|3.10.0-693.el7.x86_64|2|40G|
|172.16.91.167 |slave2|osd|bridge|centos7|3.10.0-693.el7.x86_64|2|40G|
# 环境配置  

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
  >172.16.91.165	master
172.16.91.166	slave1
172.16.91.167	slave2  


- 更新config权限
  chmod 644 ~/.ssh/config

## 创建新的用户guxin(所有节点)  
- 创建新的用户guxin
  > useradd -d /home/guxin -m guxin
passwd guxin
- 确保所有节点上新创建的用户guxin都有sudo权限  
  >echo "guxin ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/guxin
sudo chmod 0440 /etc/sudoers.d/guxin
sed -i s'/Defaults requiretty/#Defaults requiretty'/g /etc/sudoers  
## 更改成普通用户guxin  (master节点)
  > su - guxin
## 设置master节点上可以无密码访问其他node，设置ssh
- ssh-keygen
- ssh-copy-id guxin@slave1
- ssh-copy-id guxin@slave2
## 更新master节点上~/.ssh/config文件
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
