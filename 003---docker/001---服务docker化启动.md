# zookeeper服务&ensp;&ensp;docker化运行  
## 主要步骤:  
### 下载zookeeper镜像:  
    docker pull zookeeper:3.5  
### 编写启动脚本start-zk.sh  
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
### chmod+x start-redis.sh  
### ./start-redis.sh  
### 验证服务是否正常？ 
#### 检测端口号：  
    netstat -an | grep 6379  
#### telnet 登陆？  
    telnet ip 6379  
    能够进入说明服务没问题 
    输入:set book hadoop   
    返回:ok  
    说明redis可以正常提供服务，在window上，同样是这么测试即可。


