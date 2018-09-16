# <center>Viper简介</center>   
[Viper](https://github.com/spf13/viper) 是国外大神 spf13 编写的开源**配置解决方案**  
## 1、具有如下特性:  
- 设置默认值

- 可以读取如下格式的配置文件：JSON、TOML、YAML、HCL
- 监控配置文件改动，并热加载配置文件
- 从环境变量读取配置
- 从远程配置中心读取配置（etcd/consul），并监控变动
- 从命令行 flag 读取配置
- 从缓存中读取配置
- 支持直接设置配置项的值

## 2、Viper 配置读取顺序：  
- viper.Set() 所设置的值
- 命令行 flag 
- 环境变量 
- 配置文件 
- 配置中心：etcd/consul 
- 默认值 


从上面这些特性来看，

Viper 毫无疑问是非常强大的,  

而且 Viper 用起来也很方便，　　

在初始化配置文件后，读取配置只需要调用 viper.GetString()、viper.GetInt() 和 viper.GetBool() 等函数即可。  

具体使用测试用例，可以查看github上提供的测试用例


