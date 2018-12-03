通过实践操作，分析出在docker原生网络环境下，数据包的走向； 
分析工具:  
- iptables
- tcpdump   
本文主要分析在单机模式，不考虑跨节点通信；   

# 初始环境介绍   
1. docker版本  
![docker version](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/19BEC59EEC9D46ED99219E20A0ED610B/22199)    
2. 虚拟机环境   

|**系统类型**|**IP**|**role**|**cpu**|**memory**|**hostname**|
|:---|:---|:---|:---|:---|:---|
|CentOS 7.4.1708|172.16.91.225|master|4|2G|master|
3. 为了追踪数据包的流向，需要查看iptables，ebtables的日志    
    - 加载内核模块  
        - modprobe nf_log_ipv4   
        ![](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/99ED88727639420CB30CAE79C408A3D6/22215)  
    - 设置日志存储文件  
        - 本次测试是在centos7环境下，可以将日志存储到某一个文件里，如:
        vim /etc/rsyslog.conf  
        添加一行  
        kern.*  /var/log/iptables.log
        ![设置日志目录](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/BD18FB406D104C2AAA6237AAE9AC5072/22204)   
        - 重新启动rsyslogd服务  
            service rsyslog restart  
            service rsyslog status
    - 对iptables设置追踪日志  
        - iptables -t raw -A PREROUTING -p icmp -j TRACE  
        - iptables -t raw -A OUTPUT -p icmp -j TRACE   
        - 注意，最好不要使用LOG，因为对iptables的规则链入侵比较大，你需要对很多每条链都要设置LOG   
        ![设置iptables追踪](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/664F364FB61A4C7BA53F9D1873F906BD/22217)   
    - 对ebtables设置追踪日志(这个选做，为了简单，暂不使用ebtables)  
        ebtables -t broute -A BROUTING -p ipv4 --ip-proto 1 --log-level 6 --log-ip --log-prefix "TRACE: eb:broute:BROUTING" -j ACCEPT  
        ebtables -t nat -A OUTPUT -p ipv4 --ip-proto 1 --log-level 6 --log-ip --log-prefix "TRACE: eb:nat:OUTPUT"  -j ACCEPT  
        ebtables -t nat -A PREROUTING -p ipv4 --ip-proto 1 --log-level 6 --log-ip --log-prefix "TRACE: eb:nat:PREROUTING" -j ACCEPT  
        ebtables -t filter -A INPUT -p ipv4 --ip-proto 1 --log-level 6 --log-ip --log-prefix "TRACE: eb:filter:INPUT" -j ACCEPT  
        ebtables -t filter -A FORWARD -p ipv4 --ip-proto 1 --log-level 6 --log-ip --log-prefix "TRACE: eb:filter:FORWARD" -j ACCEPT    
        ebtables -t filter -A OUTPUT -p ipv4 --ip-proto 1 --log-level 6 --log-ip --log-prefix "TRACE: eb:filter:OUTPUT" -j ACCEPT  
        ebtables -t nat -A POSTROUTING -p ipv4 --ip-proto 1 --log-level 6 --log-ip --log-prefix "TRACE: eb:nat:POSTROUTING" -j ACCEPT  

4. 准备测试镜像  
    - docker pull registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/mybusybox
    - docker tag 252a8571b6b1 mybusybox   
5. 安装tcpdump  
    yum install -y tcpdump  
6. 默认网桥docker0基本信息  
    ![docker0基本信息](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/7748635C939E42D480108453ABF7F384/22211)   

# bridge网络环境   
## 创建容器  
docker run --name web1 -itd -p 8080:80 mybuxbox  

## 场景一：从宿主机到容器的访问   
1. ping web1    
![ping web1](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/93C744435A844E9998977103AA52A669/22210)   

2. 抓包docker0  
![抓包](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/5609DACFADBC4243A6F382654BF23519/22212)    

3. 查看iptables日志  
![iptables log](https://note.youdao.com/yws/public/resource/325637fdd3e566a5d270882de12217ce/xmlnote/D8DAF6C08CFD46E4BFDE56A802E191A0/22220)    
注意：抓包图和iptables日志图里的数据包序号不一致，是因为，不同时间段做的测试。   
4. 日志格式说明：TRACE:tablename:chainname:type:rulenum？     
    - 当匹配到的是普通rules时，type=”rule”;  
    - 当碰到一个user-defined chain的return target时，type=”return”；  
    - 当匹配到built-in chain(比如：PREROUTING、INPUT、OUTPUT、FORWARD和POSTROUTING)的default policy时，type=”policy”       


## 场景二：从容器访问宿主机   


## 场景三：同一个网络下，从容器到容器的访问   



## 场景四：不在同一个网络下，从容器到容器的访问   






# host网络环境  


# none网络环境   



