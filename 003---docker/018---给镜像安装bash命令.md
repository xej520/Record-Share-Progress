Alpine Docker为了精简体积，是没有安装bash的，但我们可以依照需要定制一个安装bash的镜像  

dockerfile的内容，如下：  
```
FROM alpine:3.7

MAINTAINER Rethink 
#更新Alpine的软件源为国内（清华大学）的站点，因为从默认官源拉取实在太慢了。。。
RUN echo "https://mirror.tuna.tsinghua.edu.cn/alpine/v3.4/main/" > /etc/apk/repositories

RUN apk update \
        && apk upgrade \
        && apk add --no-cache bash \
        bash-doc \
        bash-completion \
        && rm -rf /var/cache/apk/* \
        && /bin/bash

```






