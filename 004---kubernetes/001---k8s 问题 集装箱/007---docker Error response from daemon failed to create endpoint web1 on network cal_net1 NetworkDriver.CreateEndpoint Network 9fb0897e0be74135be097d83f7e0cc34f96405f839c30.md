docker Error response from daemon failed to create endpoint web1 on network cal_net1 NetworkDriver.CreateEndpoint Network 9fb0897e0be74135be097d83f7e0cc34f96405f839c307d44ce56e9aa274f82c inspection error Cannot connect to the Docker daemon at unixvarrundocker.sock. Is the docker daemon running  

- 原因之一：  
    可能是由于docker进程没有起来  
- 原因之二：  
    网络cal_net1的原因, 
- 原因之三： 
    calico服务没有挂载/var/run/docker.sock的文件  
1. 原因之一的解决方案？  
    重启docker进程      

2. 原因之二的解决方案：
    事故现场回顾：  
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/69023E491DD84BCAA7A3D495B8597D17/20743)  
    很明显，calico网络出了问题，206,207节点的bird进程没有启动起来。   

    查看端口BGP 协议是通过TCP 连接来建立邻居的，因此可以用netstat 命令验证 BGP Peer
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/FAFDCD24644D41A88AB97344D8080F11/20745)  

    解决措施：  
    登陆206,207服务器，重新启动calico服务即可  
    如登陆206服务器 
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/E0D9F93C9FA64542AD7F8FC1E97DE704/20747)  
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/71E488508EA04ABEBAE9B9A68100CE93/20749)  
3. 原因之三的解决方案： 
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/7A2054A5DB204A69994A84FDBC7B270A/20753)    
    根本原因是由于在启动calico服务的时候，没有挂载/var/run/docker.sock目录，导致的;   
    查看calico服务进程：    
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/87EFB64C1A51483E8827E39A68F06322/20757)  
    因此，需要重新添加此目录的映射   
    -v /var/run/docker.sock:/var/run/docker.sock  
    参考文献：  
    https://github.com/projectcalico/calico/issues/1303  
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/19BEDAF19128476EA720713838CA959F/20755)  
