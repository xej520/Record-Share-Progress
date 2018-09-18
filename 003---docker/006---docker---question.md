本文档已经更新到GitHub上：  
[查看地址](https://github.com/xej520/Record-Share-Progress/blob/master/003---docker/006---docker---question.md)

# docker 问题列表   
## Error parsing reference: "golang:alpine as builder" is not a valid repository/tag: invalid reference format  
### 解决措施？
#### 查看docker的版本  
>docker --version  

    Docker version 1.13.1, build 94f4240/1.13.1

#### 需要升级docker的版本  
具体升级过程，可以查看[007---如何升级docker的版本.md](https://github.com/xej520/Record-Share-Progress/blob/master/003---docker/007---%E5%A6%82%E4%BD%95%E5%8D%87%E7%BA%A7docker%E7%9A%84%E7%89%88%E6%9C%AC.md)文档  

## Get https://registry-1.docker.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers) 
- 原因？  
    因为在下载官方镜像点的镜像国内访问速度太慢  
- 解决措施？  
    使用加速器，如下所示：  
    更新/etc/docker/daemon.json文件  
    ```
    {
        "registry-mirrors":["https://registry.docker-cn.com","https://docker.mirrors.ustc.edu.cn"],
        "insecure-registries":["172.16.91.222:80"]
    }
    ```  
    主要是更新registry-mirrors 这个属性



