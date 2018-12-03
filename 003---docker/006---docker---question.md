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


## [ERROR] [token.go:48]: Unexpected error when creating the token, error: Unable to parse image from string: shera-test  
- 现状描述：  
    ![上传镜像到harbor](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/480FF728FC1941BC8004F843C93199D1/22228)  
    ![查看harbor日志](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/187664C0434D4C278D149C8A99976457/22230)   
- 原因一？  
    镜像目录不符合要求：应该是两级目录以上才可以的， 如  
    172.16.91.111:5000/mysql:v3.2&ensp;&ensp;不符合要求  
    172.16.91.111:5000/mysql/mysql:v3.2&ensp;&ensp;符合要求  
- 原因二？   
    还有一种情况，可能是tag不符合要求：  
    ![tag问题](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/C100E03AD36D4CDF82726774FAA1EF46/22232)   
    可以参考这个链接
    https://github.com/goharbor/harbor/issues/2351   

