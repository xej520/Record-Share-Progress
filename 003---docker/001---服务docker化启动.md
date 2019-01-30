#  zookeeper服务&ensp;&ensp;docker化运行
##  主要步骤:  
### 下载zookeeper镜像:  
    docker pull zookeeper:3.5  
### 编写启动脚本start-zk.sh 
    docker stop xej-zk
    docker rm xej-zk
    docker run --name xej-zk -p 2181:2181 --restart always -d zookeeper:3.5  

# mysql服务&ensp;&ensp;docker化运行  
## 主要步骤： 
### 下载mysql镜像：  
    docker pull mysql:latest  
### 编写启动脚本start-mysql.sh  
    #!/bin/bash
    cur_dir=`pwd`
    docker  stop  xej-mysql
    docker  rm  xej-mysql
    docker run --name xej-mysql -v ${cur_dir}/data:/var/lib/mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:latest  
### 链接测试：  
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/32D64D1FF5DD4CB0A62626D55C85851E/20082)  

如何使用命令行登陆mysql  
mysql -uroot -p123456
docker exec -it xej-mysql bash  
![登陆mysql](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/E927B7DE7D38493A8B056043BFB9CB05/22241)   

# redis服务&ensp;&ensp;docker化运行  
## 主要步骤：  
### 下载镜像：
    docker pull hub.c.163.com/public/redis:2.8.4  
### 编写启动脚本start-redis.sh  
    #!/bin/bash
    docker stop xej-redis
    docker rm xej-redis
    docker run -idt -p 6379:6379 -v `pwd`/data:/data --name xej-redis -v `pwd`/redis.conf:/etc/redis/redis_default.conf hub.c.163.com/public/redis:2.8.4  
### 准备配置文件： 
    随便找一个普通 的配置文件就可以了，
    运行时，如果报错的话，就注释掉，  例如下面的配置文件: 
    https://www.cnblogs.com/kreo/p/4423362.html
### 添加执行权限  
    chmod+x start-redis.sh  
### 启动服务  
    ./start-redis.sh  
### 验证服务是否正常？ 
#### 1. 检测端口号：  
    netstat -an | grep 6379  
#### 2. telnet 登陆？  
    telnet ip 6379  
    能够进入说明服务没问题 
    输入:set book hadoop   
    返回:ok  
    说明redis可以正常提供服务，在window上，同样是这么测试即可。


# sonar 容器化部署   
部署sonar 需要先部署一个数据库，这里使用postgresql了
## 1.1 准备镜像  
```
docker pull postgres:10.4   
docker pull sonarqube:7.1    
```  

## 1.2 编写脚本  
1. start-postgresql.sh
```
#!/bin/bash

docker stop postgresql
docker rm postgresql

docker run -d --name postgresql -p 5432:5432 \
	-e POSTGRES_USER=sonar \
	-e POSTGRES_PASSWORD=sonar \
	-e POSTGRE_DB=sonar   \
	-v /root/xej-sonar/data/postgresql/data:/var/lib/postgresql/data \
	postgres:10.4
```
2. start-sonar.sh
```
#!/bin/bash

docker stop sonarqube
docker rm sonarqube

docker run --name sonarqube --link postgresql -e SONARQUBE_JDBC_URL=jdbc:postgresql://postgresql:5432/sonar -p 9000:9000 -d -v /root/xej-sonar/data/sonarqube/data:/opt/
sonarqube/data -v /root/xej-sonar/data/sonarqube/extensions:/opt/sonarqube/extensions sonarqube:7.1

```  
3. 在宿主机上，创建数据存储目录(`可以替换成自己的目录`)  
```
mkdir -p /root/xej-sonar/data/postgresql/data
mkdir -p /root/xej-sonar/data/sonarqube/data  
mkdir -p /root/xej-sonar/data/sonarqube/extensions
```

4. 给脚本设置执行权限  
```
chmod +x start*
```  
5. 启动
```
./start-postgresql.sh   
./start-sonar.sh
```  
`注意`: 先启动postgresql脚本，成功后，才启动sonar脚本(`需要等一会才能访问成功`)

## 1.3 访问sonar
localhost:9000   
![访问sonar](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/90B8579395AC4E5EAEC11DA698BBB220/23177)





