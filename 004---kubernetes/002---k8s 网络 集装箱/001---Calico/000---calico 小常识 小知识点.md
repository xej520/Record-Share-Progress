# <center>Calico 小知识点 小常识 总结</center> 

1. Policy资源  
- 可以定义多个规则  
- 定义的规则是有序的  
- 面向的是多个calico 节点  
    - 通过label selector 来选择满足条件的calico节点  
- Policy定义的规则&ensp;&ensp;**优先于**&ensp;&ensp;Profile定义的规则  
- policy的简称，别名:policy, policies, pol, pols  


2. Profile 资源  
- 可以定义一系列规则  
- 规则无序  
- 面向的是单个calico 节点  
- 一个calico节点可以有0个或者多个profiles  
- profile的简称，别名:profile, profiles, pro, pros

3. Workload Endpoint 资源  
- 表示一个接口连接， 使用calico网络的容器或者VM&ensp;&ensp;跟&ensp;&ensp;它所在物理机的连接  
- 官方建议，此资源最好通过calicoctl来查看


