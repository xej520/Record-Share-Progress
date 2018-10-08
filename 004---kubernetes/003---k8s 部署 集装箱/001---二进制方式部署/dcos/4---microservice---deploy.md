# 四、kubernetes集群部署微服务
## 1. 微服务部署方案 - 思路整理
##### 我们有如下微服务：
- 消息服务：message-service
- 课程dubbo服务：course-dubbo-service
- 课程web服务：course-edge-service
- 用户thrift服务：user-thrift-service
- 用户web服务：user-edge-service
- API网关：api-gateway


##### 把它们放到kubernetes集群运行我们要考虑什么问题？
- 哪些服务适合单独成为一个pod？哪些服务适合在一个pod中？  
    - 1：消息服务 单独部署在一个pod里
    - 2：课程dubbo服务 与 课程web服务适合在一个pod里部署   
    - 3：用户thrift服务 与 用户web服务适合在一个pod里部署
    - 3：api网关适合在一个pod里服务  
- 在一个pod里面的服务**如何彼此访问**？他们的服务如何**对外提供服务**？  
    - 直接通过本机的端口去访问，如127.0.0.1:20880 
    - service的ClusterIP:port
- 单独的pod如何对外提供服务？  
    - service的ClusterIP:port

- 哪个服务作为整个服务的入口(api-gateway)，入口服务如何对外提供服务？  
    - 通过NodePort，集群外也可以访问服务   





## 2. 搞定配置
配置的模板已经为大家准备好了，但是还需要大家做一下处理才能使用哦，参考下面脚本：
```bash
$ cd ~/kubernetes-starter/service-config/
$ ls
api-gateway.yaml     message-service.yaml
course-service.yaml  user-service.yaml
#替换变量 - (hub.mooc.com:8080是我的环境的镜像仓库地址，大家修改为各自的仓库)
$ sed -i 's/{{HUB}}/hub.mooc.com:8080/g' *
```
## 3. 部署服务
##### 部署前准备：
- **要过一遍我们现有的代码配置，看看是否有需要修改的，修改后需要新生成镜像**
- **要启动好微服务依赖的服务，像zookeeper，mysql，registry等**

