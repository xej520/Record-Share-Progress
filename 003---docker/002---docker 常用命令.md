
##  docker 常用命令

### 1、删掉一个容器
    docker kill -s KILL frosty_cori
    frosty_cori 为容器名称
### 2、创建一个容器，传递环境变量
    docker run -e "ROLE=slave" -it spark:0.1 /bin/bash
    
    注意环境变量参数-e的位置，不能放在后面，要放在run命令后面  
### 3、批量获取仓库名为spark的镜像，把镜像ID打印出来  
    docker images | grep spark | awk '{print $3}'
    
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/EC6FCECBEFEA46919EA6541E97699FB5/17496) 

### 4、批量删除镜像  
    docker rmi $(docker images | grep spark | awk '{print $3}')
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/D530020B596D4CB49B2AC8F78E29FB38/17499)  
如果需要强制删除镜像的话，需要加参数-f, 如docker rmi -f ......  


### 5、搭建docker私有仓库  
#### 5.1 在主节点172.16.91.210上下载镜像registry  
    docker pull registry
#### 5.2 根据新下载的registry镜像，创建一个容器，
    A、先创建本地存储镜像的地址
    mkdir -p /usr/local/docker_registry
   
    B、创建registry容器：
    docker run -d -v /usr/local/docker_registry/:/var/lib/registry -p 5000:5000 --restart=always --privileged=true --name registry registry:latest  
   
    说明：/var/lib/registry 地址，是镜像在容器中的存储地址
    
#### 5.3 打开/etc/sysconfig/docker文件  
    找到OPTIONS选项，在最后面，添加上--insecure-registry 172.16.91.210:5000   
    
    添加这个 参数的目的是为了 下面的报错异常“Get https://172.16.91.210:5000/v1/_ping: http: server gave HTTP response to HTTPS client”  
    
    或者，在下面的文件中/etc/docker/daemon.json，添加一行"insecure-registries":["172.16.91.210:5000"] 也可以的，如我的文件内容：
    {
	"registry-mirrors":["https://registry.docker-cn.com"],
	"insecure-registries":["172.16.91.210:5000"]
    }
    
#### 5.3.1 修改了配置文件，需要重启docker服务， 
    systemctl daemon-reload
    systemctl restart docker
#### 5.3.2 查看设置是否生效呢？
    docker info
    看看安全镜像，有没有设置进去就可以了
    
#### 5.4 给本地镜像spark:0.1打一个标签
    docker tag spark:0.1 172.16.91.210:5000/spark 
    docker tag 96e2682c20f6  172.16.3.50:5000/ftp-1.4
##### 5.4.1 删除一个标签
    docker rmi -f ftp:1.0
    
#### 5.5 将新打的镜像上传到私有仓库中去
    docker push 172.16.91.210:5000/spark  
#### 5.6 查看是否上传到私有仓库了  
    docker images  
    
#### 5.7  其他机器如何查询私有镜像呢？或者下载镜像呢？
     需要重新做一次5.3操作步骤，就可以了
#### 5.8 构建镜像
    docker build -t ftp .  
#### 5.9 docker run 参数介绍
    -h --hostname string  container host name
    --privileged 使用这个参数创建的容器，具有以下几个方面的特性：
        A、容器具有真正的root权限，如果没有此参数，容器中的root用户，相当于host主机上的一个普通用户，
        B、此root用户不能使用mount挂载命令
        C、使用了该参数后，就允许在容器中创建容器了   
#### 6.0 查看容器的日志  
    docker logs -f 容器ID  
    docker logs -f 90(容器ID，只需要写前几个就可以了，自动会匹配的)
    
#### 7.0 查看当前docker容器可以使用的网络  
    docker network ls  
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/8A55321E4A204CE2B68A5DCB0226BB4F/20022)   
8. 创建calico类型的网络  
```
docker network create --driver calico --ipam-driver calico-ipam cal_net1
``` 