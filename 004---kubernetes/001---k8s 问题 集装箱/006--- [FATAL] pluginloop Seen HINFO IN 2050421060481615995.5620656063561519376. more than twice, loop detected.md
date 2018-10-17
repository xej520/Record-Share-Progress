## plugin/loop: Seen "HINFO IN 2050421060481615995.5620656063561519376." more than twice, loop detected  

###  现象描述： 使用kubeadm 部署kubernetes 1.12.1后，发现dns pod的状态不稳定，一会running，一会error 
1. 查看日志：  
```
[root@master /]#kubectl logs coredns-55f86bf584-7sbtj -n kube-system
.:53
2018/10/09 10:20:15 [INFO] CoreDNS-1.2.2
2018/10/09 10:20:15 [INFO] linux/amd64, go1.11, eb51e8b
CoreDNS-1.2.2
linux/amd64, go1.11, eb51e8b
2018/10/09 10:20:15 [INFO] plugin/reload: Running configuration MD5 = 86e5222d14b17c8b907970f002198e96
2018/10/09 10:20:15 [FATAL] plugin/loop: Seen "HINFO IN 2050421060481615995.5620656063561519376." more than twice, loop detected
```  
2. 如何解决？   
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/CA78D7E89E8B46239B5A2E1F1BE080E9/20364)  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/2614A107D7394E4DAD50B9ABEDF9FE07/20495)  
github上提供的主要解决方法：   
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/83B9B8144CD74ACEA5E8BC6BAF8012C3/20492)
主要参考文献： 
跟此问题相关的文章如下：  
https://github.com/coredns/coredns/issues/2087  
https://github.com/kubernetes/kubeadm/issues/998  


