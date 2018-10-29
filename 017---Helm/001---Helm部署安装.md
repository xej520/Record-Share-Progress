Helm部署安装  

参考文献：  
https://www.jianshu.com/p/5db132101a09  
# 一、前提要求
- Kubernetes1.5以上版本  
- 集群可访问到的镜像仓库  
- 执行helm命令的主机可以访问到kubernetes集群  

# 二、环境说明：  
1. 物理环境说明    

    |名称|IP|版本号|  
    |:---|:---|:---|  
    |master|172.16.91.205|centos7.5|    

2. 服务说明  
  
    |名称|版本号|  
    |:---|:---|  
    |kubernetes|v1.12.1|  
    |Helm|v2.9.1|  

# 三、部署Helm  
## 3.1 安装Helm的客户端  
### 3.1.1 方法一： 通过脚本的方式安装客户端    
1. 下载安装脚本  
    ```
    curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh

    ```
    ![脚本下载地址](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/EACF08B3BD70488F8C767DF3236DC5B1/21622)  

2. 添加执行权限 
    ```
    chmod 700 get_helm.sh
    ```
3. 运行脚本  
    ```
    ./get_helm.sh
    ```  

### 3.1.2 方案二：直接下载二进制文件    
1. 下载二进制文件
    ```
    wget https://storage.googleapis.com/kubernetes-helm/helm-v2.9.1-linux-amd64.tar.gz
    ```
    ![下载可执行的二进制文件](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/8B94EC69417C42FD97C650BE7A05D3CF/21624)  

2. 解压helm-v2.9.1-linux-amd64.tar.gz  
    ```
    tar -zxvf helm-v2.9.1-linux-amd64.tar.gz
    ```

3. 移动到/usr/local/bin目录下  
    ```
    mv linux-amd64/helm /usr/local/bin/  
    ``` 
4. 添加执行权限  
    ```
    chmod +x /usr/local/bin/helm
    ```  
5. 测试使用可以使用(获取客户端的版本号)   
    ```
    helm version
    ```

## 3.2 安装Helm的服务端tiller  
1. 准备镜像(可以根据自己的环境来准备)  
    - docker pull registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/tiller:v2.9.0  
    - docker tag 2a1d7ef9d530 gcr.io/kubernetes-helm/tiller:v2.9.0

2. 为tiller创建serviceaccount和clusterrolebinding  
    tiller的服务端是一个deployment，  
    在kube-system namespace下，  
    会去连接kube-api创建应用和删除，所以需要给他权限  
    ```
    kubectl create serviceaccount --namespace kube-system tiller  
    kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller  
    ```    
3. 初始化  
    - 方法一：自动拉取镜像  
        - helm init  
    - 方法二：指定镜像  
        - helm init -i gcr.io/kubernetes-helm/tiller:v2.9.0   

4. 为应用程序设置serviceaccount:  
    ```
    kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}' 
    ```  

    ![初始化Helm](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/4A37CBE778E7475F88FBF53C4E15AB20/21629)  

    ![查看Helm](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/D50624B3DD3243A8B0821CE6F195B912/21634)  











