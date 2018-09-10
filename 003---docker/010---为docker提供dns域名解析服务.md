# 为docker提供dns域名解析服务  
为docker提供域名解析服务的方式
- yum方式部署  
- docker方式部署

## 环境介绍  
|ip|role|
|:---|:---|
|172.16.91.222|dns server|
|172.16.91.166|client|
|172.16.91.167|client| 

## yum方式部署DNS服务 [物理部署]
### 具体安装过程如下：  
#### 使用下面的命令  
>yum install -y dnsmasq  
#### 打开/etc/hosts文件 ，添加要解析的域名，如
>echo "172.16.91.165 lb.guxin.com" >> /etc/hosts  
#### 启动dnsmasq服务  
>service dnsmasq restart  
#### 查看dnsmasq服务的状态  
>service dnsmasq status  

注意：  
&ensp;&ensp;&ensp;&ensp;每次更新/etc/hosts都要重启dnsmasq服务，重新加载/etc/hosts文件，
### 其他节点(166,167)上的docker，如何访问dns呢?  
&ensp;&ensp;&ensp;&ensp;需要更新/etc/docker/daemon.json，添加dns键值对 , 如 
```
{
  "registry-mirrors": ["https://registry.docker-cn.com"],
  "insecure-registries":["172.16.91.222:80"],
   "dns":["172.16.91.222"]
}

```  
### 客户端节点，需要重新启动docker服务，从而加载配置文件  
>systemctl docker restart  

&ensp;    
&ensp; 
## docker 版本的DNS服务器部署  
### 下载镜像    
>docker pull jpillora/dnsmasq  
#### 打标签    
>docker tag jpillora/dnsmasq  172.16.91.222:80/dns/dnsmasq
#### push到harbor上  
>docker push 172.16.91.222:80/dns/dnsmasq 
### 编写配置文件/root/dns/dnsmasq.conf  //配置文件位置，根据自己需求存放
```
#dnsmasq config, for a complete example, see:
#  http://oss.segetech.com/intra/srv/dnsmasq.conf
#dns解析日志
log-queries
#域名与IP映射
address=/lb.guxin.com/172.16.91.165

```  
说明:  
>docker 容器内部 会将lb.guxin.com解析成172.17.205.28
### 编写启动脚本 
```
#!/bin/bash 
docker stop xej-dnsmasq
docker rm xej-dnsmasq

docker run \
    --name xej-dnsmasq \
    -d \
    -p 53:53/udp \
    -p 6060:8080 \
    -v /root/dns/dnsmasq.conf:/etc/dnsmasq.conf \
    --log-opt "max-size=100m" \
    -e "HTTP_USER=admin" \
    -e "HTTP_PASS=123456" \
    --restart always \
    172.16.91.222:80/dns/dnsmasq
```

### web UI登陆  
    http://172.16.91.222:6060   
    username:admin  
    password:123456    


### 测试  
#### 其他节点(166,167)上的docker，如何访问dns呢?  
&ensp;&ensp;&ensp;&ensp;需要更新/etc/docker/daemon.json，添加dns键值对 , 如 
```
{
  "registry-mirrors": ["https://registry.docker-cn.com"],
  "insecure-registries":["172.16.91.222:80"],
   "dns":["172.16.91.222"]
}

```  
#### 客户端节点，需要重新启动docker服务，从而加载配置文件  
>systemctl docker restart  

其实，docker部署跟yum部署，都是加载相同的配置文件，可以从日志中观察出来