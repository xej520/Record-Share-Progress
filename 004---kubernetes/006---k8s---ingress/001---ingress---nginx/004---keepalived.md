
### 使用keepalived方式， 测试  
#### 部署keepalived     
在安装部署了nginx-ingress-controller的两个节点上，部署keepalived进程   

1. 使用yum来安装(slave1, slave2节点操作)  
```
yum install keepalived -y
```
2. 查看版本
```
[root@slave1 ~]# keepalived -v
Keepalived v1.3.5 (03/19,2017), git commit v1.3.5-6-g6fa32f2

Copyright(C) 2001-2017 Alexandre Cassen, <acassen@gmail.com>

Build options:  PIPE2 LIBNL3 RTA_ENCAP RTA_EXPIRES RTA_PREF FRA_OIFNAME FRA_TUN_ID RTAX_CC_ALGO RTAX_QUICKACK LIBIPTC LIBIPSET_DYNAMIC LVS LIBIPVS_NETLINK VRRP VRRP_AUTH VRRP_VMAC SOCK_NONBLOCK SOCK_CLOEXEC FIB_ROUTING INET6_ADDR_GEN_MODE SNMP_V3_FOR_V2 SNMP SNMP_KEEPALIVED SNMP_CHECKER SNMP_RFC SNMP_RFCV2 SNMP_RFCV3 SO_MARK
[root@slave1 ~]# 

```
 
3. 修改配置文件(slave1, slave2节点操作)   
目前，将slave1作为主节点，slave2作为备节点  
主节点的配置文件如下：  
vim /etc/keepalived/keepalived.conf  #用vim打开编辑
```
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 55
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {
        172.16.91.110/16 dev ens33 label ens33:1
    }
}  
......

```  


备节点配置文件如下：  
vim /etc/keepalived/keepalived.conf  #用vim打开编辑    
```
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 55
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {
        172.16.91.110/16 dev ens33 label ens33:1
    }
}  
.....  
```  
修改了以下属性：  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/F0138AE01AF946EFA465144F417DA685/20587)  

6. 启动keepalived服务(slave1, slave2节点操作)   
```
systemctl restart keepalived
```
7. 查看网卡情况(slave1, slave2节点操作) 
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/DD0B5C73416B4294A6F9F0F1A9697416/20589)  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/3249571BA91A42A8AD7D23AB6B9DF3C6/20591)  

#### 开始测试 
##### 使用curl方式来测试  


##### 使用host方式来测试  


