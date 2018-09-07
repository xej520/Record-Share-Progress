# mesos 原理介绍  资源调度 资源分配
## 有几个问题需要清晰？  
- mesos调度原理  
- mesos是如何资源分配的 
- 外面是如何访问服务的


## mesos 调度  
    mesos分为两级调度架构  
### 第一级调度  
1. mesos的master去调度mesos的slave，去运行任务，   
    如hadoop任务；
2.  mesos master 协调全部的slave
    确定每个slave的可用资源  
### 第二级调度  
&ensp;&ensp;&ensp;&ensp;由Framework来完成，如marathon框架，   
&ensp;&ensp;&ensp;&ensp;将framework注册到mesos集群，从而可以使用mesos集群的资源  
#### Framework包括两部分： 
- 调度器，mesos master可以直接跟调度器进行沟通  
- 执行器，运行在slave上，由mesos slave 来进行调度  

#### mesos 调度图  
![](https://note.youdao.com/yws/public/resource/005b9d9146ba9fe1df87371df7ef8da7/xmlnote/49C0E2A7C0F448618718F69072414340/20106)  
#### mesos资源分配图  
![](https://note.youdao.com/yws/public/resource/005b9d9146ba9fe1df87371df7ef8da7/xmlnote/506777518C3D4326B8168EBBF347E82E/20110)  
- slave向mesos master 来汇报自己的资源情况 
- 由master触发资源分配模块，得到的响应是framework1要求分配的全部资源  
- mesos master 向framework1发送资源邀约，描述了slave1上的可用资源 
- framework1的调度器跟mesos master进行沟通，需要在slave1上运行两个任务:   
    - 第一个任务：需要两个cpu，1g内存，
    - 第二个任务：1个cpu，2g内存
- mesos master 向 slave1 下发任务，分配资源给framework1 的执行器；接下来，由执行器来启动这两个任务了

&ensp;&ensp;&ensp;&ensp;Framework的调度器，就是跟mesos来进行谈判资源的，调度器需要运行的资源看看mesos够不够，framework将执行器告诉master，master将执行器发送给slave去执行，
执行器，可以理解为一段代码，可以跟master进行沟通的代码 
## marathon 
>适合运行长期不间断的服务，web应用，数据库，缓存，redis  

&ensp;&ensp;&ensp;&ensp;mesos
其实并不能单独的存在，必须依赖framework存在，两者协作才能完成  
![](https://note.youdao.com/yws/public/resource/005b9d9146ba9fe1df87371df7ef8da7/xmlnote/1E2E6237C4484960944E1A4909E04004/20112)  
mesos，数据中心的内核  
![](https://note.youdao.com/yws/public/resource/005b9d9146ba9fe1df87371df7ef8da7/xmlnote/636A078FC72B467DA490599D6D23E032/20114)  
整合了mesos，marathon，负载均衡，服务发现  
展示了客户端是如何访问服务的过程
上面的图，包含两个主线，  
第一个主线是master进行资源分配的  
第二个主线是，服务访问过程  
- 第一个主线介绍：  
1、slave向master报告自己的资源情况  
2、master向framework，如marathon发送资源邀约  
3、marathon根据当前自己任务需要的资源情况，告诉master，我打算在哪个slave上运行，我需要多少cpu，memory，并且把执行器发送给master  
4、master发送任务  
5、slave调度执行器  
6、执行器启动任务  
- 第二个主线介绍  
&ensp;&ensp;&ensp;&ensp;client访问marathon-lb, 就是负载均衡和服务发现，marathon-lb订阅了marathon的事件，从而知道marathon发生了什么  