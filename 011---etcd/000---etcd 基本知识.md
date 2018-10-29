# 一句话介绍etcd 
    etcd与其说是一个提供一致性服务的分布式系统，不如说是一个 __分布式kv数据库__

# etcd 主要参数介绍  
1. 参数名称解析 

|参数名称|解释|  
|:---|:---| 
|--name|方便理解的节点名称，默认为 default，在集群中应该保持唯一，可以使用 hostname|
|--data-dir|服务运行数据保存的路径，默认为 ${name}.etcd|
|--snapshot-count|指定有多少事务（transaction）被提交时，触发截取快照保存到磁盘|
|--heartbeat-interval|leader 多久发送一次心跳到 followers。默认值是 100ms|
|--eletion-timeout|重新投票的超时时间，如果 follow 在该时间间隔没有收到心跳包，会触发重新投票，默认为 1000 ms|
|--listen-peer-urls|和同伴通信的地址，比如 http://ip:2380，如果有多个，使用逗号分隔。需要所有节点都能够访问，所以不要使用 localhost！|
|--listen-client-urls|对外提供服务的地址：比如 http://ip:2379,http://127.0.0.1:2379，客户端会连接到这里和 etcd 交互|
|--advertise-client-urls|对外公告的该节点客户端监听地址，这个值会告诉集群中其他节点|
|--initial-advertise-peer-urls|该节点同伴监听地址，这个值会告诉集群中其他节点|
|--initial-cluster|集群中所有节点的信息，格式为 node1=http://ip1:2380,node2=http://ip2:2380,…。注意：这里的 node1 是节点的 --name 指定的名字；后面的 ip1:2380 是 --initial-advertise-peer-urls 指定的值|
|--initial-cluster-state|新建集群的时候，这个值为 new；假如已经存在的集群，这个值为 existing|
|--initial-cluster-token|创建集群的 token，这个值每个集群保持唯一。这样的话，如果你要重新创建集群，即使配置和之前一样，也会再次生成新的集群和节点 uuid；否则会导致多个集群之间的冲突，造成未知的错误|
|--auto-compaction-retention|自动清理压缩的时间，默认单位是小时|
|||


所有以 --init 开头的配置都是在 bootstrap 集群的时候才会用到，后续节点的重启会被忽略。  
NOTE：所有的参数也可以通过环境变量进行设置，--my-flag 对应环境变量的 ETCD_MY_FLAG；但是命令行指定的参数会覆盖环境变量对应的值。

2. --listen-client-urls  和  --advertise-client-urls ？
    - --listen-client-urls
        - 用于指定etcd和客户端的连接端口
    - --advertise-client-urls  
        - 用于指定etcd服务器之间通讯的端口 
        - etcd有要求，如果-listen-client-urls被设置了，那么就必须同时设置-advertise-client-urls，所以即使设置和默认相同，也必须显式设置

3. --auto-compaction-retention?  
由于ETCD数据存储多版本数据，随着写入的主键增加历史版本需要定时清理，　默认的历史数据是不会清理的，数据达到2G就不能写入，必须要清理压缩历史数据才能继续写入；  
所以根据业务需求，在上生产环境之前就提前确定，历史数据多长时间压缩一次；　我们的生产环境现在升级后是默认一小时压缩一次数据。这样可以极大的保证集群稳定，减少内存和磁盘占用

4. --max-request-bytes？  
etcd Raft消息最大字节数，ETCD默认该值为1.5M; 但是很多业务场景发现同步数据的时候1.5M完全没法满足要求，所以提前确定初始值很重要；　
由于1.5M导致我们线上的业务无法写入元数据的问题，  
我们紧急升级之后把该值修改为默认32M,但是官方推荐的是10M，大家可以根据业务情况自己调整

5. --quota-backend-bytes？  
ETCDdb数据大小，默认是２G,当数据达到２G的时候就不允许写入，必须对历史数据进行压缩才能继续写入；　
参加1里面说的，我们启动的时候就应该提前确定大小，官方推荐是8G,这里我们也使用8G的配置


# etcd 底层通信协议介绍 
使用的是grpc  

# etcd 与 zk的不同 


# etcd 使用场景？  
- 配置管理和服务发现
    - 这些场景读多写少

# etcd 不适合做什么？  
- etcd默认只保存1000个历史事件，所以不适合有大量更新操作的场景， 会导致数据的丢失 

# etcd 注意事项？  
- 目前etcd没有图形化工具  

# etcd构建自身高可用集群主要有三种形式?  
- 静态发现
    - 预先已知etcd集群中有哪些节点，在启动时通过--initial-cluster参数直接指定好etcd的各个节点地址  
- etcd动态发现 
    - 通过已有的etcd集群作为数据交互点，然后在扩展新的集群时实现通过已有集群进行服务发现的机制。
    - 比如官方提供的：discovery.etcd.io
- DNS动态发现
    - 通过DNS查询方式获取其他节点地址信息

# etcd集群中节点个数的说明？ 
&ensp;&ensp;&ensp;&ensp;通常按照需求将集群节点部署为3，5，7，9个节点。  
&ensp;&ensp;&ensp;&ensp;这里能选择偶数个节点吗？最好不要这样。  
&ensp;&ensp;&ensp;&ensp;原因有二：
- 偶数个节点集群不可用风险更高，表现在选主过程中，有较大概率或等额选票，从而触发下一轮选举。
- 偶数个节点集群在某些网络分割的场景下无法正常工作。当网络分割发生后，将集群节点对半分割开。此时集群将无法工作。按照RAFT协议，此时集群写操作无法使得大多数节点同意，从而导致写失败，集群无法正常工作。
