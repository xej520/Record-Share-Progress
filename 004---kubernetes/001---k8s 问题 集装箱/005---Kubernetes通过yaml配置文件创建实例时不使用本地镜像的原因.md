# [转] Kubernetes通过yaml配置文件创建实例时不使用本地镜像的原因   


就是为神马利用k8s创建pod时，会一直远程拉取镜像？  
## 1. 查看kubelet日志或者docker日志    
```
Jun 12 17:02:03 k8s ntpd[789]: Listen normally on 18 veth5737c59 fe80::c4e0:e4ff:fe31:a648 UDP 123 
 
Jun 12 17:02:03 k8s ntpd[789]: new interface(s) found: waking up resolver 
 
Jun 12 17:02:03 k8s dockerd-current[1164]: time="2017-06-12T17:02:03.659830673+08:00" level=warning msg="Error getting v2 registry: Get https://reg.docker.lc/v2/: dial tcp 10.0.10.9:443: getsockopt: no route to host" 
 
Jun 12 17:02:03 k8s dockerd-current[1164]: time="2017-06-12T17:02:03.659899809+08:00" level=error msg="Attempting next endpoint for pull after error: Get https://reg.docker.lc/v2/: dial tcp 10.0.10.9:443: getsockopt: no route to host" 
 
Jun 12 17:02:06 k8s kube-scheduler[6154]: I0612 17:02:06.544109    6154 leaderelection.go:247] lock is held by k8s-node and has not yet expired 
 
Jun 12 17:02:06 k8s dockerd-current[1164]: time="2017-06-12T17:02:06.667426272+08:00" level=error msg="Attempting next endpoint for pull after error: Get https://reg.docker.lc/v1/_ping: dial tcp 10.0.10.9:443: getsockopt: no route to host" 
 
Jun 12 17:02:06 k8s kubelet[1345]: E0612 17:02:06.668855    1345 docker_manager.go:2295] container start failed: ErrImagePull: image pull failed for reg.docker.lc/share/nginx:latest, this may be because there are no credentials on this request.  details: (Get https://reg.docker.lc/v1/_ping: dial tcp 10.0.10.9:443: getsockopt: no route to host) 
 
Jun 12 17:02:06 k8s kubelet[1345]: E0612 17:02:06.668956    1345 pod_workers.go:184] Error syncing pod cb61d30a-4f4d-11e7-8ea8-000c29e9277a, skipping: failed to "StartContainer" for "nginx" with ErrImagePull: "image pull failed for reg.docker.lc/share/nginx:latest, this may be because there are no credentials on this request.  details: (Get https://reg.docker.lc/v1/_ping: dial tcp 10.0.10.9:443: getsockopt: no route to host)" 
 
Jun 12 17:02:06 k8s kubelet[1345]: E0612 17:02:06.942588    1345 docker_manager.go:2295] container start failed: ImagePullBackOff: Back-off pulling image "reg.docker.lc/share/nginx:latest" 
 
Jun 12 17:02:06 k8s kubelet[1345]: E0612 17:02:06.942643    1345 pod_workers.go:184] Error syncing pod cb61d30a-4f4d-11e7-8ea8-000c29e9277a, skipping: failed to "StartContainer" for "nginx" with ImagePullBackOff: "Back-off pulling image \"reg.docker.lc/share/nginx:latest\"" 
```  
## 2. 分析原因？  
后来对比了几个网上下载回来的yaml配置文件，有一个参数选项：imagePullPolicy: Always ，镜像的拉取策略，总是拉取；但是我的配置文件中并没有添加这个选项，根据这样可以想象到，默认就可能是Always的，于是网上搜了一下，同样有网友遇到这样的情况，都是会自动到远程拉取镜像，并不使用本地的镜像。

## 3. 那么这个参数的可选项有哪些呢？

官方其实已经说明了，只是没有详细看文档；https://kubernetes.io/docs/concepts/containers/images/

By default, the kubelet will try to pull each image from the specified registry. However, if the imagePullPolicy property of the container is set to IfNotPresent or Never, then a local image is used (preferentially or exclusively, respectively).

默认情况是会根据配置文件中的镜像地址去拉取镜像，如果设置为IfNotPresent 和Never就会使用本地镜像。

IfNotPresent ：如果本地存在镜像就优先使用本地镜像。

Never：直接不再去拉取镜像了，使用本地的；如果本地不存在就报异常了。

## 4. 参数的作用范围：  
```
spec: 
  containers: 
    - name: nginx 
      image: image: reg.docker.lc/share/nginx:latest 
      imagePullPolicy: IfNotPresent   #或者使用Never 
```