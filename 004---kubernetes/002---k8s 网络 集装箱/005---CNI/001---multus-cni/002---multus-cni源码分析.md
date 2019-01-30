
# 一、背景介绍


# 二、原理介绍  
## 1.1 核心原理介绍：    
1. pod是逻辑概念，是抽象的。   

2. pod是由容器组成的；  
3. 同一个pod里面，所有容器共享同一个网络栈(包括:网卡信息，ip，dns等)   
4. 研究pod的多网卡，其实也就是研究分析docker容器的多网卡   
5. 容器的本质，其实是一个被限制了的进程(被namespace进行了限制)   
6. 创建容器时，会创建一对veth pair， 我们将其中一端放入容器里，这样，在容器里，就可以看到此网卡了。    
7. 那么，如何创建多个网卡呢？  很显然，循环创建多个veth pair，依次将其中的一端都放入相同的命名空间namespace里，也就是说，在同一个容器里，多个网卡，具有同一个命名空间，不就是多网卡了么。    
	

具体，如何将veth pair的一端放入命名空间里，可以参考下面的链接，文章不错。   
	https://www.cnblogs.com/hustcat/p/3928261.html  
	https://blog.csdn.net/sld880311/article/details/77650937

## 1.2 multus-cni原理介绍  
1. Pod如何知道要创建哪些网卡呢？  
	通过注解的方式，如:  
	![](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/41E55FC2478F477AB8F6DB4209060576/23126)   
	还有别的写法，就不具体列举了。   

	这样multus-cni会去调用k8s接口，获取到pod，解析注解，根据注解的配置，如macvlan-conf，去创建相应的网卡   

2. 创建一个网络栈，需要哪些插件呢？  
https://github.com/containernetworking/plugins  
![](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/59FCBDB752124495AF6BD3570414ED29/23128)       
3. 创建多个网卡时，每个网卡会有一个配置信息，那么这些配置信息，k8s是如何识别的？  
比方说，要使用macvlan-conf.yaml配置文件： 这个配置文件什么意思呢？   
可以简单的理解成，要使用cni插件macvlan去创建网卡信息，使用host-local去分配ip信息。    
![](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/DE973D4D34E14F159B24ED10239D5264/23130)   
那么，k8s是如何识别上面的配置信息呢？  
通过CRD向k8s进行注册一个类型，进行识别的。   
![](https://note.youdao.com/yws/public/resource/eab756e5b4ffe2e93a14041c450b2408/xmlnote/0F8581A7F2144AE1A7CB31921D856513/23132)   
4. 原理介绍  






# 三、源码解析   





1. multus-cni版本  
3.0 
2. func cmdAdd(args *skel.CmdArgs, exec invoke.Exec, kubeClient k8s.KubeClient) (cnitypes.Result, error) 
args: 参数
```
{
	"ContainerID": "b470b06e26b09fb9aba5ef151b5eac80cf669d997093b66212cf6b66da234e71",
	"Netns": "/proc/9495/ns/net",
	"IfName": "eth0",
	"Args": "IgnoreUnknown=1;K8S_POD_NAMESPACE=default;K8S_POD_NAME=s0;K8S_POD_INFRA_CONTAINER_ID=b470b06e26b09fb9aba5ef151b5eac80cf669d997093b66212cf6b66da234e71",
	"Path": "/opt/cni/bin",
	"StdinData": "eyJjbmlWZXJzaW9uIjoiIiwiZGVsZWdhdGVzIjpbeyJkZWxlZ2F0ZSI6eyJpc0RlZmF1bHRHYXRld2F5Ijp0cnVlfSwidHlwZSI6ImZsYW5uZWwifV0sImt1YmVjb25maWciOiIvZXRjL2t1YmVybmV0ZXMva3ViZWxldC5jb25mIiwibmFtZSI6Im11bHR1cy1jbmktbmV0d29yayIsInR5cGUiOiJtdWx0dXMifQ=="
}
```  
对StdinData的内容:  进行base64位解码  
![StdinData 64解码](https://note.youdao.com/yws/public/resource/65c0779cc4cefe6692e5390d1990f2c7/xmlnote/0BD3BD0CA54B4C2697EE072F7471C199/22955)   
进行JSON格式化后，如下:  
```
{
	"cniVersion": "",
	"delegates": [{
		"delegate": {
			"isDefaultGateway": true
		},
		"type": "flannel"
	}],
	"kubeconfig": "/etc/kubernetes/kubelet.conf",
	"name": "multus-cni-network",
	"type": "multus"
}
```

也就是每个节点/etc/cni/net.d目录下的配置文件 
![10-multus-with-flannel.conf](https://note.youdao.com/yws/public/resource/65c0779cc4cefe6692e5390d1990f2c7/xmlnote/0068643CB9F64E7191DECBCB7D40ED19/22958)   

exec 参数指定的是： 这个参数为空，并没有传递



n, err := types.LoadNetConf(args.StdinData)  
返回结果n的类型，就是NetConf,如 
```
{
	"name": "multus-cni-network",
	"type": "multus",
	"ipam": {},
	"dns": {},
	"prevResult": null,
	"confDir": "/etc/cni/multus/net.d",
	"cniDir": "/var/lib/cni/multus",
	"binDir": "/opt/cni/bin",
	"delegates": null,
	"kubeconfig": "/etc/kubernetes/kubelet.conf",
	"logFile": "",
	"logLevel": ""
}
```

k8sArgs, err := k8s.GetK8sArgs(args)    
![获取k8sArgs内容](https://note.youdao.com/yws/public/resource/65c0779cc4cefe6692e5390d1990f2c7/xmlnote/E46D26A554A449C3B2D70C9AD55F86D7/22960)
k8sArgs的内容如下：  
![k8sArgs内容](https://note.youdao.com/yws/public/resource/65c0779cc4cefe6692e5390d1990f2c7/xmlnote/38900C1C58E845AA9E09F81CBF33BD94/22962)


result, err := invoke.DelegateAdd(delegate.Conf.Type, delegate.Bytes, exec)  
result的内容，如下：  
![result](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/B04A8414A32D41419F22FDA568729D49/22979)   
![result](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/7DCF95ED73DC4717B9B9811DCF6860E3/22982)   


# 二、conf.go文件  
1. func LoadNetConf(bytes []byte) (*NetConf, error)  
bytes []byte 就是
cmdAdd函数中的第一个参数args里的StdinData参数， 也就是/etc/cni/net.d/10-multus-with-flannel.conf文件



# 三、 k8sclient.go文件  
1. netData, err := client.GetRawWithPath(rawPath)  
rawPath的值，如下：  
apis/k8s.cni.cncf.io/v1/namespaces/default/network-attachment-definitions/macvlan-conf  
netData的值，如下：  需要进行64位解码： 
![netData](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/4C5351FD74524B40BD6868A13CE1D8E4/22972)
```
{
	"apiVersion": "k8s.cni.cncf.io/v1",
	"kind": "NetworkAttachmentDefinition",
	"metadata": {
		"creationTimestamp": "2019-01-10T07:57:29Z",
		"generation": 1,
		"name": "macvlan-conf",
		"namespace": "default",
		"resourceVersion": "5269",
		"selfLink": "/apis/k8s.cni.cncf.io/v1/namespaces/default/network-attachment-definitions/macvlan-conf",
		"uid": "607b0a47-14ad-11e9-a526-000c29fe0a86"
	},
	"spec": {
		"config": "{ \"cniVersion\": \"0.3.0\", \"type\": \"macvlan\", \"master\": \"ens33\", \"mode\": \"bridge\", \"ipam\": { \"type\": \"host-local\", \"subnet\": \"192.168.1.0/24\", \"rangeStart\": \"192.168.1.200\", \"rangeEnd\": \"192.168.1.216\", \"routes\": [ { \"dst\": \"0.0.0.0/0\" } ], \"gateway\": \"192.168.1.1\" } }"
	}
}
```
从源码中，查看定义的类型:  
types.go文件  
![NetworkAttachmentDefinition](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/B7CBA879BDEF4DBE9637FE28679C8108/22971)   

![其实，就是从mavlan-conf中的内容](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/D679DE396675404F9DD8FC70D5BAD358/22970)


# 四、 types.go  

delegate, err := types.LoadDelegateNetConf(configBytes, net.InterfaceRequest)  

delegate的内容:  
![delegate(就是网络代理)](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/B6C4EE81D26449B6B4B569A436D287D5/22977)


# 如何设置从网卡，就是容器内从网卡名字的设置
![效果展示](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/68E9F19CC3704B8A82B367C401CECFE0/23043)  
![注解](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/28C943432A7E45F0918B67C57A8B10EE/23045) 
![源码解析](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/8A4597F49834412BB037C095EBA527E6/23047) 
![对应的类型](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/5953586887174495954E2FE903EBFA7D/23049)  


问题描述：  
rpc error: code = Unknown desc = NetworkPlugin cni failed to teardown pod "_" network: Multus: Err in getting k8s network from pod: getPodNetworkAnnotation: failed to query the pod  in out of cluster comm: resource name may not be empty   
是因为，multusc-cni向k8s查询配置时，查询不到。这里是缺少默认的配置  
flannel-conf.yaml
![](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/FA98A03EC65446CBB2FBD1D0AB3508BF/23081)  
![](https://note.youdao.com/yws/public/resource/65bed9be947daa8ee70f724a1079d7e5/xmlnote/857972D39B7E49F2800CFBD89F91B6B1/23079)  






