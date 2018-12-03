# 调试iptables   

## 步骤一 设置iptables规则

    iptables -t raw -A OUTPUT -p icmp -j LOG  
    iptables -t raw -A PREROUTING -p icmp -j LOG 
 
    iptables -t raw -A OUTPUT -p icmp -j TRACE   
    iptables -t raw -A PREROUTING -p icmp -j TRACE 
`备注：`   
- 不知道为什么，我这里必须同时设置LOG,TRACE这两个动作，调式日志才能打印出来  
- 重启虚拟机后，这些规则会消失   
## 步骤二  
设置日志存储路径  
vim /etc/rsyslog.conf
kern.*  /var/log/iptables.log  

## 步骤二
重启rsyslog服务   
service rsyslog status
service rsyslog restart






















