# 一、Jenkins 容器化 部署  
## 1.1 部署环境说明   
  | IP地址 | 主机名 |  网络模式  |系统|服务|
| :------|:------|:-------|:---|:---|
|172.16.91.121|master|桥接|centos7.5|jenkins、docker|
|172.16.91.222|harbor|桥接|centos7.5|harbor、sonar、docker|
![centos版本](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/7FB89839A884407B9ECD0CD679861C35/23188)   


## 1.2 版本说明
|服务|版本|
|:---|:---|
|jenkins|2.150.2|
|docker|18.06.1-ce|

## 1.3 准备Jenkins依赖的全局工具   
为了验证Jenkins是否可以同时支持不同版本的全局工具，添加了mvn3.6,3.3.9， 以及jdk的不同版本。(可以根据自己的需要进行下载)  
![全局工具](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/A301EAA5DD754241A473F1D4FB2BBE64/23184)  
通过下面的连接，可以进行下载  
https://pan.baidu.com/s/1OcsUcWVOe9oxru9tXPJFNA  
如果jenkins的版本不合适的话，可以通过下面的地址，下载合适的war包  
http://mirrors.jenkins.io/war-stable/  

## 1.4 容器化部署sonar(非必须)   
为了进行语法规则测试，才安装的sonar,  
根据自己的实际情况，进行安装。  
可以参考下面的链接，进行容器化部署安装:  
https://www.jianshu.com/p/d36a12105e9d  

## 1.5 容器化部署harbor(非必须) 
可以参考其他博客。  

## 1.6 构建jenkins所需要的Dockerfile  
![Dockerfile](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/97C4627BDDF545FB9EB4DAE6BAF55975/23186)     

`构建镜像`:   
```
docker build -t jenkins .
```  

## 1.7 编写启动脚本  
(`这里的脚本是略有问题，最下面有新的`) 
![启动jenkins脚本](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/C38A15EB5BBB4631AA2E675EB0A93A4A/23190)
`为什么这里用了很多挂载呢？ ` 
1. 如果将依赖的工具，全局添加到Dockerfile的话，将来不好维护。每次变化，都需要重新打镜像，势必镜像会越来越大。   
2. 将jenkins的主目录，挂载出来了，可以提高数据的可靠性，方便维护，使用。  

`使用环境变量的说明`
1. JENKINS_HOME, 必须有这个。不然的话，第一次重启的时候，需要重新设置, 不会用挂载出来的jenkins_home  
2. JAVA_HOME, 因为启动jenkins服务需要使用java命令，当然你可以使用带有java环境的基础镜像。   

![Dockerfile, 启动脚本](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/BD3C16FCB00B41249FAE94D114CEEBA4/23192)   
![挂载路径](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/97AF1BD4F6084400995D2FFF31C38E06/23194)

下面，可以启动jenkins服务。  
登陆到页面172.16.91.121:8888/jenkins，按照提示，进行就可以了。   

## 1.8 主要配置文件说明   
a. credentials.xml  
![credentials.xml](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/2FF2C6700728457DB2FB8F87EB07C0F1/23196)
b. hudson.tasks.Maven.xml
![hudson.tasks.Maven.xml.xml](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/1BC6BA92A9864AA0B6C15994715FA50C/23200)  
c. config.xml   
![config.xml](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/13B56159B55E474C88542174CED65A74/23198)  

# 二、设置Jenkins   
## 1. 安装插件   
根据自己的需要，安装插件。  
系统管理-->插件管理  
我这里安装maven，docker，git，github,sonarquber   
遇到的问题，插件选项不可用   
![插件问题](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/6D77ABAEC6C54B818822ACF48603B745/23202)   
![解决措施](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/53449D67021C4BE8BFFDF43ABD0485E5/23204)

## 2. 安装插件的方式   
方式一：  参考下面的链接   
https://www.jianshu.com/p/3b5ebe85c034   
方式二： 参考下面的链接   
http://updates.jenkins-ci.org/download/plugins/  
下载好插件后，如docker.hpi, 然后，`直接copy到plugins目录下，重启jenkins服务就可以了`。     
## 3. 在系统设置里 配置sonar   
![配置sonarqube servers](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/1F57CCFFB1D54931945B5044866111C5/23206)    
![在sonar里获取server authentication token](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/607252CA3FF540F0BC3154291DEA4C0B/23208)   

## 4. 在全局工具里配置  
### 4.1 配置Maven Configuration  
![配置settings.xml](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/6F8E7AD9605E4F9A9C71764E45D123E4/23210)   

### 4.2 配置JDK    
![配置jdk](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/E253DC86FB354F23A73DE185F1132255/23214)  
### 4.3 配置git
![配置git](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/86EF1F7A60EC45C58770D70EFFD76831/23217)  
### 4.4 配置SonarQube Scanner   
![配置SonarQube Scanner](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/897365DC21A04D4F80E0C78C1738F8E0/23219)  

### 4.5 配置Maven   
![配置maven](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/1A76ED3B30784D10830FECDF952640E5/23221)  

### 4.6 配置docker      
![配置docker](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/17892AECED364624B42EEDD806D7FF60/23223)  

# 二、Jenkins测试  
测试的功能包括: 代码托管、编译、构建镜像并上传到私有仓库、语法规则校验    
## 2.1 代码托管   
![代码托管](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/8FA1704187F8485E98C70F340FA66E69/23226)   

## 2.2 maven打包   
![maven](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/807817EEB631404CA2DA804DD38FD643/23228)  

## 2.3 构建镜像   
![构建镜像1](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/5929ACC21A324181BEF9C7B0616DBC15/23230)  
![构建镜像2](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/93D11AC50BB64249BC39EC6FB90649B4/23232)  

## 2.4 语法规则校验   
![语法规则校验](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/918B8B97AEA544A7B7BFB98A0DCEEA9D/23234)  

保存后，立即构建，就可以了。   

出问题的话，可以从打印日志里，发现问题。   
还可以发现`运行的命令`。   
![maven命令](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/EE66DC120B1C4A6A918E696B9BF0CD17/23236)   
![构建镜像命令](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/F12EE402165541028FF184C01DFFE65D/23238)   
![push镜像到harbor命令](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/15DF0BF8266C46C4ABBE5430DFA2C1FE/23240)   
![语法规则校验命令](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/666A88D622C14A75929DED2485B358A9/23242)   

可见，可以简单的认为，jenkins，就是将你要做的事情，串行执行，功能都是通过添加插件实现的。   

# 备注：  
突然发现，启动脚本中，maven仓库的挂载，忘记修改了。   

![jenkins启动脚本](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/EF424410E46C4878ACB39AB4002019C9/23246)  




















