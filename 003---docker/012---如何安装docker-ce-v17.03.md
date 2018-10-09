# 如何安装部署docker 17.03-ce 

# 一、卸载旧的docker版本  
```
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```  
# 二、 配置阿里云Docker Yum源  
```
# Set up repository
yum install -y yum-utils device-mapper-persistent-data lvm2

# Use Aliyun Docker
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

```  

# 三、 安装指定版本
## 3.1 查看当前支持的版本
```
yum list docker-ce --showduplicates
```
## 3.2 安装较旧版本（比如Docker 17.03.2) 时需要指定完整的rpm包的包名，并且加上--setopt=obsoletes=0 参数：
```
# on a new system with yum repo defined, forcing older version and ignoring obsoletes introduced by 17.06.0
yum install -y --setopt=obsoletes=0 \
   docker-ce-17.03.2.ce-1.el7.centos.x86_64 \
   docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch
``` 

## 3.3 或者 安装Docker较新版本（比如Docker 18.03.0)时加上rpm包名的版本号部分：
```
yum install docker-ce-18.03.0.ce
```
## 3.4 或者安装Docker最新版本，无需加版本号：
```
 yum install docker-ce
```  
# 四、 启动Docker服务  
```
systemctl enable docker
systemctl start docker
```
# 五、 问题？ 
&ensp;&ensp;&ensp;&ensp;启动服务时，报错systemctl restart docker  
```
Job for docker.service failed because the control process exited with error code. See "systemctl status docker.service" and "journalctl -xe" for details.
```  
1. 分析原因？   
使用 journalctl -xe 查看详情
```
[root@master /]# journalctl -xe
Apr 27 13:48:23 *** dockerd[1744]: time="2018-04-27T13:48:23.423296237+08:00" level=info msg="libcontainerd: new containerd process, pid: 1747"
Apr 27 13:48:24 *** dockerd[1744]: time="2018-04-27T13:48:24.432236287+08:00" level=error msg="[graphdriver] prior storage driver overlay2 failed: driver not supported"
```  
2. 解决方案？   
修改docker镜像，容器存放位置  
/var/lib/docker   
    ```
    mv /var/lib/docker /var/lib/docker.old  
    ```  
3. 重新启动docker服务  
    ```
    systemctl restart docker
    ```  
4. 检查是否能查看镜像  
    ```
    docker images 
    ``` 
5. 确认无误后卸载临时挂载点，删除/var/lib/docker.old  
    ```
    rm -rf /var/lib/docker.old
    ```  

参考文献：  
http://www.mamicode.com/info-detail-2272231.html 
https://blog.csdn.net/nklinsirui/article/details/80610058  
https://blog.csdn.net/qq_39629343/article/details/80168084 
