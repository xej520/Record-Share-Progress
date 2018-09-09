# mesos 部署  
## 部署mesos HA的话，需要安装的组件？  
- zookeeper 
- mesos master 
- mesos slave 
- marathon 
- marathon-lb  

## 环境  
### 服务器信息
|name|ip|role|task|  
|:--|:--|:--|:--| 
|master|172.16.91.165|master|MesosMaster、zookeeper、marathon-lb| 
|node1|172.16.91.166|worker|MesosSlave、Marathon| 
|node2|172.16.91.167|worker|MesosSlave|  
### 关系图 
图是借鉴的，但组件之间的调用关系是一样的。  
MesosMaster, MesosSlave, Marathon 都需要向zookeeper进行注册  
![](https://note.youdao.com/yws/public/resource/12ee10065fc6c44c1ff631ba40a40b06/xmlnote/B562735CA8FA470C8AB22B290CC793EA/18981)  
&ensp;  
&ensp;
## 主要搭建流程  
>备注:  
    本次部署过程，采用docker化的方式部署mesos  
### 整个部署流程分为下面几步？  
- 拉取镜像，如master，slave，marathon，marathon-lb,zookeeper  
    - 镜像官网:  
https://hub.docker.com/r/mesosphere/  
- 编写启动脚本  

- 运行脚本，启动服务   
&ensp;  
&ensp;
### 部署Mesos  
#### 部署MesosMaster 
1. 拉取master镜像&ensp;&ensp;(https://hub.docker.com/r/mesosphere/mesos-master/tags/ )  
    >docker pull mesosphere/mesos-master:1.4.1   
 
2. 编写启动脚本(setup-master.sh   )
    ```
    #!/bin/bash 
    docker stop mesos-master
    docker rm mesos-master
    docker run -d --net=host \
        --hostname=172.16.91.165 \
        --name=mesos-master
        -e MESOS_PORT=5050 \
        -e MESOS_ZK=zk://172.16.91.222:2181/mesos \
        -e MESOS_QUORUM=1 \
        -e MESOS_REGISTRY=in_memory \
        -e MESOS_LOG_DIR=/var/log/mesos \
        -e MESOS_WORK_DIR=/var/tmp/mesos \
        -v "$(pwd)/log/mesos:/var/log/mesos" \
        -v "$(pwd)/work/mesos:/var/tmp/mesos" \
        mesosphere/mesos-master:1.4.1 --no-hostname_lookup --ip=172.16.91.165
    ```   
    环境变量MESOS_QUORUM的值是n/2+1,如n表示master的节点个数，如只有一个master节点时，n=1,则MESOS_QUORUM=1
&ensp;
#### 部署MesosSlave  
1. 拉取slave镜像&ensp;&ensp;( https://hub.docker.com/r/mesosphere/mesos-slave/)  
    >docker  pull  mesosphere/mesos-slave:1.4.1   
2.  编写启动脚本(setup-slave.sh   )
    ```
    #!/bin/bash
    docker stop mesos-slave
    docker rm mesos-slave
    docker run -d --net=host --privileged \
        --name=mesos-slave \
        -e MESOS_PORT=5051 \
        -e MESOS_MASTER=zk://172.16.91.222:2181/mesos \
        -e MESOS_SWITCH_USER=0 \
        -e MESOS_CONTAINERIZERS=docker,mesos \
        -e MESOS_LOG_DIR=/var/log/mesos \
        -e MESOS_WORK_DIR=/var/tmp/mesos \
        -v "$(pwd)/log/mesos:/var/log/mesos" \
        -v "$(pwd)/work/mesos:/var/tmp/mesos" \
        -v /var/run/docker.sock:/var/run/docker.sock \dock
        -v /sys:/sys \
        -v /usr/local/bin/docker:/usr/local/bin/docker \
        mesosphere/mesos-slave:1.4.1  --no-systemd_enable_support \
        --no-hostname_lookup --ip=172.16.91.166
    ```  
注意参数：
MESOS_CONTAINERIZERS=docker,mesos  
docker 表明，支持docker容器的部署   
mesos 表示，mesos同样实现了自己的类似于docker的功能，如果不加mesos的话，创建marathon  app 中，就不支持命令行；   
报异常:  
 failed to start: None of the enabled containerizers(docker) could create a container for the provided TaskInfo/ExecutorInfo message 
![](https://note.youdao.com/yws/public/resource/005b9d9146ba9fe1df87371df7ef8da7/xmlnote/FC7D21D0C31A49A8A762380ADFFED14D/20120)  

&ensp;
### 部署Marathon  
1. 拉取marathon镜像&ensp;&ensp;(https://hub.docker.com/r/mesosphere/marathon/)
    >docker pull mesosphere/marathon:v1.5.2      
2. 编写启动脚本(setup-marathon.sh  )
    ```
    #!/bin/bash
    docker stop marathon
    docker rm marathon
    docker run -d --net=host \
        --name=marathon \
        -e HTTP_PORT=7070 \
        -e MARATHON_HTTP_PORT=7070 \
        172.16.91.222:80/mesos/marathon:v1.5.2 \
        --master zk://172.16.91.222:2181/mesos \
        --zk zk://172.16.91.222:2181/marathon
    ``` 
web ui访问marathon：  
172.16.91.165:7070  

&ensp;
### 部署Marathon-lb  
1. 拉取marathon-lb镜像&ensp;&ensp;(https://hub.docker.com/r/mesosphere/marathon-lb/)  
    > docker pull mesosphere/marathon-lb:v1.11.1   
2. 编写启动脚本(setup-marathon-lb.sh   )
    ```
    #!/bin/bash
    docker stop marathon-lb
    docker rm marathon-lb
    docker run -d --net=host \
        --name=marathon-lb \
        -e PORTS=9090 \
        --group external \
        --marathon http://172.16.91.165:7070 \
        mesosphere/marathon-lb:v1.11.1 sse 
        

    #!/bin/bash
    docker stop marathon-lb
    docker rm marathon-lb
    docker run -d -p 9090:9090 \
        -e PORTS=9090 \
        --name=marathon-lb \
        172.16.91.222:80/mesos/marathon-lb:v1.11.1 sse  \
        --group=external \
        --marathon http://172.16.91.165:7070 

    ``` 
注意参数：
 --group external 是给marathon-lb的分组，external是随便写的
&ensp;
### 部署zookeeper  
请查看文档[001---服务docker化启动.md  ](https://github.com/xej520/Record-Share-Progress/blob/master/003---docker/001---%E6%9C%8D%E5%8A%A1docker%E5%8C%96%E5%90%AF%E5%8A%A8.md)中zookeeper启动部分  
