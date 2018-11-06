

# 更新yum源  
yum install -y yum-utils device-mapper-persistent-data lvm2  
yum-config-manager --add-repo  https://download.docker.com/linux/centos/docker-ce.repo

# 安装docker-ce 
yum install -y docker-ce 



# 配置私有仓库地址  
```
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://gwusysv0.mirror.aliyuncs.com"]
}EOF
```

# 启动服务
```
systemctl enable docker
systemctl start docker  
systemctl status docker
```

docker run -itd --name a-web1 mybusybox   