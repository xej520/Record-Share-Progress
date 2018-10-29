# 背景  
&ensp;&ensp;&ensp;&ensp;总结整理记录etcd使用过程中遇到的问题  
# 问题列表  
1. Failed to restart etcd.service: Unit is not loaded properly: Invalid argument.  
![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/3AD37E26AB7C43E68FC5952BDAF0B018/20672)  
![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/2C733652AA8447A6881F72B7D7BB2732/20675)  
2. failed to reach the peerURL(http://172.16.91.196:2380) of member e4c76aefe3f21425 (Get http://172.16.91.196:2380/version: dial tcp 172.16.91.196:2380: getsockopt: connection refused)  
    异常信息如下：  
    ```
    Oct 28 17:17:34 master etcd[952]: health check for peer e4c76aefe3f21425 could not connect: dial tcp 172.16.91.196:2380: getsockopt: connection refused
    Oct 28 17:17:34 master etcd[952]: failed to reach the peerURL(http://172.16.91.196:2380) of member e4c76aefe3f21425 (Get http://172.16.91.196:2380/version: dial tcp 172.16.91.196:2380: getsockopt: connection refused)
    Oct 28 17:17:34 master etcd[952]: cannot get the version of member e4c76aefe3f21425 (Get http://172.16.91.196:2380/version: dial tcp 172.16.91.196:2380: getsockopt: connection refused)
    Oct 28 17:17:38 master etcd[952]: failed to reach the peerURL(http://172.16.91.196:2380) of member e4c76aefe3f21425 (Get http://172.16.91.196:2380/version: dial tcp 172.16.91.196:2380: getsockopt: connection refused)
    Oct 28 17:17:38 master etcd[952]: cannot get the version of member e4c76aefe3f21425 (Get http://172.16.91.196:2380/version: dial tcp 172.16.91.196:2380: getsockopt: connection refused)
    Oct 28 17:17:39 master etcd[952]: health check for peer e4c76aefe3f21425 could not connect: dial tcp 172.16.91.196:2380: getsockopt: connection refused
    Oct 28 17:17:42 master etcd[952]: failed to reach the peerURL(http://172.16.91.196:2380) of member e4c76aefe3f21425 (Get http://172.16.91.196:2380/version: dial tcp 172.16.91.196:2380: getsockopt: connection refused)
    Oct 28 17:17:42 master etcd[952]: cannot get the version of member e4c76aefe3f21425 (Get http://172.16.91.196:2380/version: dial tcp 172.16.91.196:2380: getsockopt: connection refused)
    Oct 28 17:17:44 master etcd[952]: health check for peer e4c76aefe3f21425 could not connect: dial tcp 172.16.91.196:2380: getsockopt: connection refused
    ```  
    图形展示：  
    ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/7301BB310120475FAB324FD383D96751/21612)  
    ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/60702CB8132D41F39F3A1A59C9C3E25F/21615)    
    ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/FE23855372A54A33B421FA428A5742C4/21617)  

3. dial tcp 10.1.2.172:2380: getsockopt: no route to host   
    此为防火墙没有关闭，要关闭防火墙  
    ```
    service iptables  stop
    service firewalld  stop
    systemctl disable firewalld
    ```


