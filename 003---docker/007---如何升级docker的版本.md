# 如何升级docker的版本 ？  
##### 查看当前的docker的版本？  
- docker --version  
        
    Docker version 1.13.1, build 94f4240/1.13.1  
## 升级具体步骤：
### 查找主机上关于docker的软件包  
>rpm -qa | grep docker    

![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/19234536107A470AB81A46DAC3DE9A65/20103)  
### 移除相关软件包  
>yum remove -y docker-client-1.13.1-63.git94f4240.el7.centos.x86_64  
yum remove -y docker-common-1.13.1-63.git94f4240.el7.centos.x86_64  
yum remove -y docker-1.13.1-63.git94f4240.el7.centos.x86_64  
### 校验docker 是否还可以用  
>docker 
### 使用curl升级到最新版本&ensp;&ensp;(失败的话，多尝试几次)
>curl -fsSL https://get.docker.com/ | sh  
### 重新启动  
>systemctl restart docker  
### 设置为开机启动docker 
>systemctl enable docker  
### 查看docker版本信息  
>docker -v
