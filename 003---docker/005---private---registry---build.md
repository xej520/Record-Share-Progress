# 私有仓库搭建  (两种方案)  
- registry&ensp;&ensp;(非生产) 
- harbor&ensp;&ensp;(生产环境，企业级私有仓库)

## 方案一 利用registry搭建
### 拉取register镜像  
    docker pull registry:2  
### 创建容器  
    docker run -d -p 5000:5000  registry:2 
### 给本地镜像打一个tag 
    docker tag zookeeper:3.5 localhost:5000/zookeeper:3.5  
### 将本地仓库上传到私有仓库中  
    docker push localhost:5000/zookeeper:3.5 
![](https://note.youdao.com/yws/public/resource/f1faa52351e2fa440809d43482961828/xmlnote/6F434C5FED884D108E34C18C4D053FDE/20100)   
### 此方案存在的问题？  
- 存在单点故障问题
- 跟用户交互不友好，方便  
### 总结？  
    生产环境一般不会用的  

## 方案二 利用harbor搭建   
属于企业级的私有仓库方案  
### 前提环境准备？         
>需要提前安装docker-compse,   
请查看003---docker-compose setup.md文档  

### 安装harbor 
#### harbor官网  
>https://github.com/goharbor/harbor/releases     
找到Download Binary中的Harbor offline installer， 采用离线下载安装
#### 下载  
>wget https://github.com/goharbor/harbor/releases/download/v1.2.2/harbor-offline-installer-v1.2.2.tgz

#### 解压  
> tar -zxvf harbor-offline-installer-v1.2.2.tgz

#### 进入harbor目录  
> cd harbor
#### 修改配置文件harbor.cfg  
>A、将hostname的值设置成服务器的IP。  或者设置成域名，如hub.guxin.com &ensp;&ensp;(必做)  
B、将secretkey_path = /data  改成 secretkey_path = ./data      
C、如果是mac系统的话，需要加上8080， hostname = IP:8080   
D、修改登录密码harbor_admin_password = 123456
#### 修改配置文件docker-compose.yaml，  (若是window系统，选做)
>A、就是修改此文件中volumes的值，统一添加上&ensp;&ensp;.&ensp;&ensp;表示将日志，数据，配置文件都存在当前工作目录下  
B、若系统是mac系统的话，  
&ensp;&ensp;a、还需要修改端口号，改成8080:80  

#### 修改data目录的权限  
>chmod 755 data   
#### 开始安装脚本   
>./install.sh   
#### 添加http访问方式
##### 方法一：
>在启动docker服务的命令后面，添加  
--insecure-registry=172.16.91.165:80
##### 方法二: 
>    vim /etc/docker/daemo.json   
{  
  "registry-mirrors":["https://registry.docker-cn.com"],  
  "insecure-registries":["172.16.91.165:80"]  
}    

注意：  
方法一，方法二中，参数的名字是不一的，要看清楚   
 insecure-registry、insecure-registries

#### 通过页面访问harbor  
>用户名:admin  
密码:123456  
http://172.16.91.165

#### 将本地镜像上传到harbor上  
##### docker login 172.16.91.165:80
>用户名:admin  
密码:123456

##### 将本地镜像打tag  
>docker tag message-thrift-go-service 172.16.91.165:80/micro-service/message-thrift-go-service  
##### 将本地镜像上传到harbor上  
>docker push 172.16.91.165:80/micro-service/message-thrift-go-service   

注意:  
>虽然使用的是默认端口80，但是，你最好还是显示的注明    
#### 其他服务器如何访问私有仓库  
##### 更新配置文件，如  
>    vim /etc/docker/daemo.json   
{  
  "registry-mirrors":["https://registry.docker-cn.com"],    
  "insecure-registries":["172.16.91.165:80"]  
}    
##### 重新启动docker服务  
>systemctl restart docker  
##### 拉取镜像  
>docker pull 172.16.91.165:80/micro-service/message-thrift-go-service

### 当前配置文件  
#### docker-compose.yml  
```
version: '2'
services:
  log:
    image: vmware/harbor-log:v1.2.2
    container_name: harbor-log 
    restart: always
    volumes:
      - /var/log/harbor/:/var/log/docker/:z
    ports:
      - 127.0.0.1:1514:514
    networks:
      - harbor
  registry:
    image: vmware/registry:2.6.2-photon
    container_name: registry
    restart: always
    volumes:
      - /data/registry:/storage:z
      - ./common/config/registry/:/etc/registry/:z
    networks:
      - harbor
    environment:
      - GODEBUG=netdns=cgo
    command:
      ["serve", "/etc/registry/config.yml"]
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "registry"
  mysql:
    image: vmware/harbor-db:v1.2.2
    container_name: harbor-db
    restart: always
    volumes:
      - /data/database:/var/lib/mysql:z
    networks:
      - harbor
    env_file:
      - ./common/config/db/env
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "mysql"
  adminserver:
    image: vmware/harbor-adminserver:v1.2.2
    container_name: harbor-adminserver
    env_file:
      - ./common/config/adminserver/env
    restart: always
    volumes:
      - /data/config/:/etc/adminserver/config/:z
      - /data/secretkey:/etc/adminserver/key:z
      - /data/:/data/:z
    networks:
      - harbor
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "adminserver"
  ui:
    image: vmware/harbor-ui:v1.2.2
    container_name: harbor-ui
    env_file:
      - ./common/config/ui/env
    restart: always
    volumes:
      - ./common/config/ui/app.conf:/etc/ui/app.conf:z
      - ./common/config/ui/private_key.pem:/etc/ui/private_key.pem:z
      - /data/secretkey:/etc/ui/key:z
      - /data/ca_download/:/etc/ui/ca/:z
      - /data/psc/:/etc/ui/token/:z
    networks:
      - harbor
    depends_on:
      - log
      - adminserver
      - registry
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "ui"
  jobservice:
    image: vmware/harbor-jobservice:v1.2.2
    container_name: harbor-jobservice
    env_file:
      - ./common/config/jobservice/env
    restart: always
    volumes:
      - /data/job_logs:/var/log/jobs:z
      - ./common/config/jobservice/app.conf:/etc/jobservice/app.conf:z
      - /data/secretkey:/etc/jobservice/key:z
    networks:
      - harbor
    depends_on:
      - ui
      - adminserver
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "jobservice"
  proxy:
    image: vmware/nginx-photon:1.11.13
    container_name: nginx
    restart: always
    volumes:
      - ./common/config/nginx:/etc/nginx:z
    networks:
      - harbor
    ports:
      - 80:80
      - 443:443
      - 4443:4443
    depends_on:
      - mysql
      - registry
      - ui
      - log
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "proxy"
networks:
  harbor:
    external: false


```  
#### harbor.cfg  
```
## Configuration file of Harbor

#The IP address or hostname to access admin UI and registry service.
#DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
hostname = 172.16.91.165

#The protocol for accessing the UI and token/notification service, by default it is http.
#It can be set to https if ssl is enabled on nginx.
ui_url_protocol = http

#The password for the root user of mysql db, change this before any production use.
db_password = root123

#Maximum number of job workers in job service  
max_job_workers = 3 

#Determine whether or not to generate certificate for the registry's token.
#If the value is on, the prepare script creates new root cert and private key 
#for generating token to access the registry. If the value is off the default key/cert will be used.
#This flag also controls the creation of the notary signer's cert.
customize_crt = on

#The path of cert and key files for nginx, they are applied only the protocol is set to https
ssl_cert = /data/cert/server.crt
ssl_cert_key = /data/cert/server.key

#The path of secretkey storage
secretkey_path = /data

#Admiral's url, comment this attribute, or set its value to NA when Harbor is standalone
admiral_url = NA

#The password of the Clair's postgres database, only effective when Harbor is deployed with Clair.
#Please update it before deployment, subsequent update will cause Clair's API server and Harbor unable to access Clair's database.
clair_db_password = password

#NOTES: The properties between BEGIN INITIAL PROPERTIES and END INITIAL PROPERTIES
#only take effect in the first boot, the subsequent changes of these properties 
#should be performed on web ui

#************************BEGIN INITIAL PROPERTIES************************

#Email account settings for sending out password resetting emails.

#Email server uses the given username and password to authenticate on TLS connections to host and act as identity.
#Identity left blank to act as username.
email_identity = 

email_server = smtp.mydomain.com
email_server_port = 25
email_username = sample_admin@mydomain.com
email_password = abc
email_from = admin <sample_admin@mydomain.com>
email_ssl = false

##The initial password of Harbor admin, only works for the first time when Harbor starts. 
#It has no effect after the first launch of Harbor.
#Change the admin password from UI after launching Harbor.
harbor_admin_password = 123456

##By default the auth mode is db_auth, i.e. the credentials are stored in a local database.
#Set it to ldap_auth if you want to verify a user's credentials against an LDAP server.
auth_mode = db_auth

#The url for an ldap endpoint.
ldap_url = ldaps://ldap.mydomain.com

#A user's DN who has the permission to search the LDAP/AD server. 
#If your LDAP/AD server does not support anonymous search, you should configure this DN and ldap_search_pwd.
#ldap_searchdn = uid=searchuser,ou=people,dc=mydomain,dc=com

#the password of the ldap_searchdn
#ldap_search_pwd = password

#The base DN from which to look up a user in LDAP/AD
ldap_basedn = ou=people,dc=mydomain,dc=com

#Search filter for LDAP/AD, make sure the syntax of the filter is correct.
#ldap_filter = (objectClass=person)

# The attribute used in a search to match a user, it could be uid, cn, email, sAMAccountName or other attributes depending on your LDAP/AD  
ldap_uid = uid 

#the scope to search for users, 1-LDAP_SCOPE_BASE, 2-LDAP_SCOPE_ONELEVEL, 3-LDAP_SCOPE_SUBTREE
ldap_scope = 3 

#Timeout (in seconds)  when connecting to an LDAP Server. The default value (and most reasonable) is 5 seconds.
ldap_timeout = 5

#Turn on or off the self-registration feature
self_registration = on

#The expiration time (in minute) of token created by token service, default is 30 minutes
token_expiration = 30

#The flag to control what users have permission to create projects
#The default value "everyone" allows everyone to creates a project. 
#Set to "adminonly" so that only admin user can create project.
project_creation_restriction = everyone

#Determine whether the job service should verify the ssl cert when it connects to a remote registry.
#Set this flag to off when the remote registry uses a self-signed or untrusted certificate.
verify_remote_cert = on
#************************END INITIAL PROPERTIES************************
#############


```
