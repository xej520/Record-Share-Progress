# 一、背景介绍  
&ensp;&ensp;&ensp;&ensp;一直对--advertise-client-urls这个参数迷迷糊糊的，搞不清到底是做什么的，网上的一些解释也似懂非懂的，因此，本篇文章就做一个小测试，去探寻究竟。  
仅供参考! 
# 二、测试环境介绍 
|主机名|IP|系统|服务|
|:---|:---|:---|:---|
|master|172.16.91.195|centos7.5|etcd(单例)|
|harbor|172.16.91.222|centos7.5|无|

# 三、参数说明  
&ensp;&ensp;&ensp;&ensp;etcd有要求，如果--listen-client-urls被设置了，那么就必须同时设置--advertise-client-urls，所以即使设置和默认相同，也必须显式设置

# 四、测试  
## 4.1 测试1: 将--advertise-client-urls设置成http://127.0.0.1:2379  
1. 更新配置文件 
![更新配置文件](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/AA427466131541AFAADB00165F723AC5/20644)
2. 重新启动etcd服务 
    ```
    systemctl daemon-reload
    systemctl restart etcd 
    ```
3. 在本机上测试  
etcdctl --endpoints=http://127.0.0.1:2379 --debug ls
![本机测试](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/69E32B536F5D41F7B9EAE5BB7CF8C5AC/20642)  
4. 在同一个局域网的其他机器上测试 
![harbor节点上测试](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/A810BA3666C5450A9E3CA721C1E39110/20646)  

## 4.2 测试2: 将--advertise-client-urls设置成http://172.16.91.195:2379  
1. 更新配置文件 
![更新配置文件](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/4DCA687D44254B44B7F8AF03CACBD61A/20649)  
2. 重新启动etcd服务 
    ```
    systemctl daemon-reload
    systemctl restart etcd 
    ```
3. 在本机上测试  
etcdctl --endpoints=http://127.0.0.1:2379 --debug ls
![本机测试](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/99DBEF87F8BA414E83B97B035E9D406B/20651)  

4. 在同一个局域网的其他机器上测试 
![harbor节点上测试](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/BC671C3B8B134DB0AC9FB1BDA8730132/20653)  
## 4.3 测试3： 将--advertise-client-urls设置成http://172.16.91.222:2379 
1. 更新配置文件 
![更新配置文件](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/036773E4ACA24FC693FB7CDB9148FD4B/20656)  
2. 重新启动etcd服务 
    ```
    systemctl daemon-reload
    systemctl restart etcd 
    ```
3. 在本机上测试  
![本机测试](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/3DB4EBA46805438F8810E941EB108727/20658)
4. 在同一个局域网的其他机器上测试 
![harbor节点上测试](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/CD5F268065DF43FF9049773B67550575/20660)
## 4.4 测试4： 将--advertise-client-urls设置成http://172.16.91.222,http://127.0.0.1:2379,http://172.16.91.195:2379
1. 更新配置文件 
![更新配置文件](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/09477639FEBB43FEBC00338DA88B4672/20667)  
2. 重新启动etcd服务 
    ```
    systemctl daemon-reload
    systemctl restart etcd 
    ```
3. 在本机上测试  
![本机测试](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/752F8171C17441CDB7EDA4419464B6F5/20663)  
4. 在同一个局域网的其他机器上测试 
![harbor节点上测试](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/0BE316F66D194592BD195C845EBC1E02/20665)  

## 4.5 分析  
观看上面测试中，debug输出的信息，会发现etcdctl的基本工作流程  
### 4.5.1 etcdctl的基本工作流程？ 
![etcdctl的基本工作流程](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/CDCD372A23E941419A5CFDE1D8ABFE9D/20670)  


## 4.6 总结： 
- --advertise-client-urls  
    - 就是客户端(etcdctl/curl等)跟etcd服务进行交互时请求的url  
- --listen-client-urls  
    - 这个参数是etcd服务器自己监听时用的，也就是说，监听本机上的哪个网卡，哪个端口  
- 说明etcdctl的底层逻辑，应该是调用curl跟etcd服务进行交换




