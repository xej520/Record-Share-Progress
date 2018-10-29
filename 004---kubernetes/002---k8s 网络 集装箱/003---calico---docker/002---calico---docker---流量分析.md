

# 创建一个calico网络  
```
docker network create --driver calico --ipam-driver calico-ipam cal_net1
```
![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/5266087B59C84249AFD06090E43C2032/20741)  
# 创建容器  
```
docker run --net cal_net1 --name web1 -itd -p 8080:80 mynginx  
```


