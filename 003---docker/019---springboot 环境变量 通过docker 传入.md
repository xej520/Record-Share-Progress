当docker启动SpringBoot打包的服务时，且一些参数需要从外界获取而非写死在properties文件里，通过以下两步完成此需求：    
 1.  在配置文件中配置环境变量   
 ```
spring.redis.host=${REDIS_HOST:127.0.0.1}
spring.redis.port=6379
spring.redis.timeout=30000
以上表是REDIS_HOST在系统环境变量中获取，如果获取不到默认值为127.0.0.1
 ```
2.  在启动docker容器时传入环境参数 
```
docker run -d --name test2 ｛镜像名｝ -e REDIS_HOST=192.168.0.1
```  


