# docker 内部是如何对域名进行解析的？  
## 大概有以下几个方案吧？ 
- 方案一：修改docker容器内部的/etc/hosts文件，添加映射关系  
- 方案二：使用dns进行解析

&ensp;  
## 方案一：  
1. 进入docker容器  
    >docker exec -it 容器名称  bash  
2. 更新hosts文件，添加域名映射关系  
    >echo "172.16.91.110 lb.guxin.com"  >> /etc/hosts  

3. 测试  
    >ping lb.guxin.com

## 方案二：  
具体部署方式请参考文档[010---为docker提供dns域名解析服务.md ](https://github.com/xej520/Record-Share-Progress/blob/master/003---docker/010---%E4%B8%BAdocker%E6%8F%90%E4%BE%9Bdns%E5%9F%9F%E5%90%8D%E8%A7%A3%E6%9E%90%E6%9C%8D%E5%8A%A1.md)
