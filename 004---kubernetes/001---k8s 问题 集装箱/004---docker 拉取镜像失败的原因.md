# docker 拉取镜像失败的原因  
查看日志  
A、systemctl status docker 
B、journalctl -f -u docker
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/6367B8A6A01C4379B2FAE33B9BE0820B/20368)  



# 问题一： 找不到域名registry-1.docker.io： 
0. 事故现场：  
    ```
    error pulling image configuration: Get https://registry-1.docker.io/v2/library/hello-world/blobs/sha256:f2a91732366c0332ccd7afd2a5c4ff2b9af81f549370f7a19acd460f87686bc7: dial tcp 35.169.231.249:443: i/o timeout

    ```
1. 若是centos系统话，安装依赖工具： 
    ```
    yum install -y bind-tils  
    ```  
2. 查看到域名registry-1.docker.io可用的IP  
    ![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/3E5CD4B49F1D4DC08EA9FA911D7A1235/20370)  
3. 添加域名解析  
   ```
   echo "54.175.43.85 registry-1.docker.io" >> /etc/hosts

   ```  
4. 重新拉取镜像  
    ```
    docker pull nginx
    ```      