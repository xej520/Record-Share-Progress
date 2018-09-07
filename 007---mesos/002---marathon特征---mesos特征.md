
### Mesos的特征  
- 强大的资源管理  
这是mesos的核心，这就是为什么会把mesos称为分布式的kernel
mesos能够保证集群内的所有用户，来平等的使用资源，cpu，memory，storage  

- Kernel和Framework的分离  
mesos只负责资源的调度和管理，非常轻量，
- 门槛比较低，易于使用  
- 大厂使用，爱奇艺，数人云，siri, 推特  

### Marathon的特征？  
- 高可用(marathon本身也支持集群方式运行的)
- Constraints(标签功能，就是进行资源邀约时，有针对性，将任务运行cpu比较多，或者运行在某些指定的机器上)
- 服务发现  负载均衡
- 健康检查(基于tcp，基于HTTP，基于shell命令)
- 支持事件订阅
自己写应用，向marathon进行事件订阅，marathon就会将事件推送，包括服务的运行，停止，重启等事件
- 完善的REST API(marathon提供了非常漂亮的web UI, 可以查看每个运行实例的状态，)