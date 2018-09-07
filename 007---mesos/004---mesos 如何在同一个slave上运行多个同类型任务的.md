## 如何实现在同一个slave运行多个任务？  
- mesos 使用了隔离模块，隔离模块使用进程隔离机制，来运行这些任务
如使用了linux的cgroup，
- 后来，mesos增加对docker的支持，就使用了docker本身的隔离技术
不管使用什么隔离技术，都需要将执行器打包，发送给slave，在slave上运行  