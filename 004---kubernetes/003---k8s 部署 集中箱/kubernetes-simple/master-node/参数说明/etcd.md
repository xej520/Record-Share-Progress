# etcd.service  参数说明  

- WorkingDirectory  
    定义的工作目录，配置文件等，会存放在这个位置的
- --name  
    启动这个etcd进程的名字，跟其他etcd进程不一致就可以了|
- --listen-client-urls  
    - 监听客户端的URL，  
    - http://127.0.0.1:2379 表示监听本机的访问，  
    - http://172.16.91.222:2379 表示监听其他节点的访问，使用172.16.91.222这个ip和2379端口进行访问etcd就可以；
    - 写etcd进程所在节点的真实IP即可
- --advertise-client-urls  
    http://192.168.1.102:2379 是 建议的客户端的url地址，  
    可以进行代理，  
    也可以多个node节点之间的通信  
    就是建议其他人访问etcd的地址  
- --data-dir  
    就是存储数据的地址
- --listen-peer-urls  
    - 监听的 
    - 用于etcd节点之间通信的url, 可监听多个，集群内部将通过这些url进行数据的交互(如选举，数据同步等) 
    - 如：http://172.16.91.222:2380  
- --initial-advertise-peer-urls  
   该节点同伴监听地址，这个值会告诉集群中其他节点  
- --initial-cluster 
    - 集群中所有节点的信息，
    - 格式为 node1=http://ip1:2380,node2=http://ip2:2380,… 。
    - 注意：这里的 node1 是节点的 --name 指定的名字；后面的 ip1:2380 是 --initial-advertise-peer-urls 指定的值  
- -- initial-cluster-state     
    - 新建集群的时候，这个值为 new ；  
    - 假如已经存在的集群，这个值为 existing   
- --initial-cluster-token     
    - 创建集群的 token，这个值每个集群保持唯一。
    - 这样的话，如果你要重新创建集群，即使配置和之前一样，也会再次生成新的集群和节点 uuid；
    - 否则会导致多个集群之间的冲突，造成未知的错误
- --advertise-client-urls     
    - 对外公告的该节点客户端监听地址，这个值会告诉集群中其他节点



  

