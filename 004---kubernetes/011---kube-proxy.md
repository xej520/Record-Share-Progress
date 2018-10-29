__Kube-Proxy简述__   

参考文献：  
https://ywnz.com/linuxyffq/2530.html


运行在每个节点上，监听 API Server 中服务对象的变化，再通过管理 IPtables 来实现网络的转发  
Kube-Proxy 目前支持三种模式：  
- UserSpace  
    - k8s v1.2 后就已经淘汰  
- IPtables
    - 目前默认方式   
- IPVS
    - 需要安装ipvsadm、ipset 工具包和加载 ip_vs 内核模块  

下面我们来说说这几种模式的异同：
# 1、UserSpace  
UserSpace 是让 Kube-Proxy 在用户空间监听一个端口，所有的 Service 都转发到这个端口，然后 Kube-Proxy 在内部应用层对其进行转发。
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/7298C6CE7D4A4CFBB7421F9E866FA56D/20593)  
Kube-Proxy 会为每个 Service 随机监听一个端口 (Proxy Port)，并增加一条 IPtables 规则。  
从客户端到 ClusterIP:Port 的报文都会被重定向到 Proxy Port，Kube-Proxy 收到报文后，通过 Round Robin (轮询) 或者 Session Affinity（会话亲和力，即同一 Client IP 都走同一链路给同一 Pod 服务）分发给对应的 Pod。   
这种方式最大的缺点显然就是 UserSpace 会造成所有报文都走一遍用户态，造成整体性能下降，这种方在 Kubernetes 1.2 以后已经不再使用了。
# 2、Iptables  
IPtables 方式完全由 IPtables 来实现，这种方式直接使用 IPtables 来做用户态入口，而真正提供服务的是内核的 Netilter。  
Kube-Proxy 只作为 Controller，这也是目前默认的方式。  

Kube-Proxy 的 IPtables 方式也是支持 Round Robin 和 Session Affinity 特性。
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/501BF9B56DC64731977466B571E0AFB2/20596)  

Kube-Proxy 监听 Kubernetes Master 增加和删除 Service 以及 Endpoint 的消息。对于每一个 Service，Kube Proxy 创建相应的 IPtables 规则，并将发送到 Service Cluster IP 的流量转发到 Service 后端提供服务的 Pod 的相应端口上。  
注：  
虽然可以通过 Service 的 Cluster IP 和服务端口访问到后端 Pod 提供的服务，但该 Cluster IP 是 Ping 不通的。  
其原因是 __Cluster IP 只是 IPtables 中的规则，并不对应到一个任何网络设备__。  
IPVS 模式的 Cluster IP 是可以 Ping 通的。
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/2B72CC93C5C64234B24A7EFA890E5BB7/20601)  

# 3、IPVS  
Kubernetes 从 1.8 开始增加了 IPVS 支持，IPVS 相对 IPtables 效率会更高一些。  
使用 IPVS 模式需要在运行 Kube-Proxy 的节点上安装 ipvsadm、ipset 工具包和加载 ip_vs 内核模块。  

当 Kube-Proxy 以 IPVS 代理模式启动时，Kube-Proxy 将验证节点上是否安装了 IPVS 模块，如果未安装，则 Kube-Proxy 将回退到 IPtables 代理模式。  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/7A88562365F54272B647DB35F0EB34E8/20598)  

这种模式，Kube-Proxy 会监视 Kubernetes Service 对象 和 Endpoints，调用 Netlink 接口以相应地创建 IPVS 规则并定期与 Kubernetes Service 对象 和 Endpoints 对象同步 IPVS 规则，以确保 IPVS 状态与期望一致。访问服务时，流量将被重定向到其中一个后端 Pod。  

与 IPtables 类似，IPVS 基于 Netfilter 的 Hook 功能，但使用哈希表作为底层数据结构并在内核空间中工作。这意味着 IPVS 可以更快地重定向流量，并且在同步代理规则时具有更好的性能。此外，IPVS 为负载均衡算法提供了更多选项，例如：rr (轮询调度)、lc (最小连接数)、dh (目标哈希)、sh (源哈希)、sed (最短期望延迟)、nq(不排队调度)等。  

注：  
IPVS 是 LVS 项目的一部分，是一款运行在 Linux Kernel 当中的 __4 层负载均衡器__，性能异常优秀。使用调优后的内核，可以轻松处理每秒 10 万次以上的转发请求。   
目前在中大型互联网项目中，IPVS 被广泛的用于承接网站入口处的流量。































