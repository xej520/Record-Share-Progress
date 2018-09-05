# docker-compose 的安装
登陆官网：  
https://github.com/docker/compose/releases 

## 方式一：  
    curl -L https://github.com/docker/compose/releases/download/1.22.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
    chmod +x /usr/local/bin/docker-compose  
## 方式二：  
- 直接根据自己服务器的版本，下载对应的部署包，
![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/5ACC2CC1B05A4811BA711015E0E0A629/20096) 
- 上传到服务器的/usr/local/bin目录下
- 重新命名  
    >mv docker-compose-Linux-x86_64 docker-compose
- 检测版本号  
    > docker-compose --version