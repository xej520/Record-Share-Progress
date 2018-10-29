本文主要分析calico中的ipPool资源  
关于calico的集群部署，可以参考文章:  
https://www.jianshu.com/p/2f8d8b4d5296  
# 一、环境介绍   
1. 物理环境介绍  

    |主机名称|系统版本|IP|  
    |:---|:---|:---|  
    |master|centos7.5|172.16.91.205|  
    |slave1|centos7.5|172.16.91.206|  
    |slave2|centos7.5|172.16.91.207| 

2. 服务部署介绍  

    |服务名称|版本|
    |:---|:---|
    |docker|v17.03.2-ce|
    |etcd|v3.2.22|
    |calico|v2.6|

# 二、calico支持的模式？ 
- BGP模式 
    - 路由规则直接使用物理机网卡作为路由器转发
    - 路由即纯bgp模式，理论上ipip模式的网络传输性能低于纯bgp模式
- IPIP模式 
    - 一种妥协的overlay机制，在宿主机创建1个”tunl0”虚拟端口  
    - 通过tun10作为路由转发  
    - 分为两种模式：    
        - ipip always模式（纯ipip模式）
        - ipip cross-subnet模式（ipip-bgp混合模式），指“同子网内路由采用bgp，跨子网路由采用ipip”
# 三、如何管理ipPool的生命周期？ 
## 3.1 如何创建ipPool 资源对象？    
- 第一步：自定义yaml，如myPool.yaml
- 第二步：通过calicoctl工具，进行资源创建  
    - calicoctl create -f myPool.yaml

## 3.2 如何删除ipPool 资源对象？  
- 方法一：根据文件名删除
    - calicoctl delete -f myPool.yaml  
- 方法二：根据cidr删除  
    - calicoctl delete ipPool 10.254.0.0/24 

## 3.3 如何更新ipPool 资源对象？  
- 方法一：calicoctl apply -f myPool.yaml
- 方法二：calicoctl replace -f myPool.yaml  
![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/A54553C307094F8096E60F65525418D0/20772)  

# 四、ipPool
## 4.1 默认ipPool
![ipPool默认情况](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/9E90CDF766E643C9986F55B70FEE7D01/20779)  

## 4.2 删除默认ipPool资源(选做) 

1. 删除之前，可以先进行备份一份：  
    ```
    calicoctl get ipp -yaml > ipPool-default.yaml
    ```
2. 删除
    ![删除默认ipPool资源](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/6A6C2D9CB70141C3A69B8A2586413D23/20782)  

## 4.3 BGP模式  
### 4.3.1 创建bgp模式下的ipPool资源对象 
1. 自定义bgp模式的ipPool资源对象  
    vim ipPool-bgp.yaml
    ```
    - apiVersion: v1
    kind: ipPool
    metadata:
        cidr: 192.168.0.0/16
    spec:
        nat-outgoing: true
        disabled: false
    ```

2. 创建ipPool资源  
    ```
    calicoctl create -f ipPool-bgp.yaml
    ```
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/4E778D5B51B84F96A2BC307D43CF6686/20785)  
### 4.3.2 查看物理机网卡信息  
1. 查看master节点的网卡信息   
    ![master节点-bgp模式-网卡信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/71F8DE7211E949D49E7F5C3EEA1858F0/20787)  
2. 查看slave1节点的网卡信息   
    ![slave1节点-bgp模式-网卡信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/52833331F7904D14BF150211781F8283/20790)  
3. 查看slave2节点的网卡信息   
    ![slave2节点-bgp模式-网卡信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/210577CFCEA3473582C70B58002966B4/20792)  

### 4.3.3 查看物理机路由表信息  

1. 查看master节点的路由表信息   
    ![master节点-bgp模式-路由信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/9A74C4B3005E4E83AC93D88AA28C26DA/20795)  
2. 查看slave1节点的路由表信息   
    ![slave1节点-bgp模式-路由信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/73492F2A97A14F198F843550DD43B96A/20798)  
3. 查看slave2节点的路由表信息   
    ![slave2节点-bgp模式-路由信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/8ADDE5AE409D43B5BFAB2A704EB87148/20800)    

### 4.3.4 清理ipPool资源
>calicoctl delete -f ipPool-bgp.yaml
    ![清理资源ipPool](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/F00D4D79555C49C8BC5FA4D8C6355BB3/20803)  

#### 4.3.5 BGP模式总结：  
- BGP模式下，不会在服务器上创建tun10， 
- 路由转发功能是通过服务器的网卡对流量进行转发的  


## 4.4 IPIP模式  
### 4.4.1 如何设置成IPIP模式  
    通过显示的设置ipip属性； 
### 4.4.2 纯ipip模式
#### 4.4.2.1 创建纯ipip模式下的ipPool资源对象
1. 自定义纯ipip模式的ipPool资源对象  
    vim ipPool-ipip-always.yaml
    ```
    apiVersion: v1
    kind: ipPool
    metadata: 
    cidr: 10.254.0.0/24 
    spec:
    ipip: 
        enabled: true
        mode: always
    nat-outgoing: false
    #默认即使false, 也就是启用此ipPool
    disabled: false 
    ```
2. 创建ipPool资源  
    ```
    calicoctl create -f ipPool-ipip-always.yaml
    ```  
    ![ipip-always模式](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/2B4B9B061F1C43CAA1B0CBA1A75527A0/20808)  

#### 4.4.2.2 查看物理机网卡信息  

1. 查看master节点的网卡信息   
    ![master节点-ipip-always模式-网卡信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/97CE7BFB8E0B40DEAE9B9B5ADA80C2B1/20811)  
2. 查看slave1节点的网卡信息   
    ![slave1节点-ipip-always模式-网卡信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/B5EB9397EE92425F82DAB6D2CD2A9DB7/20813)  
3. 查看slave2节点的网卡信息   
    ![slave2节点-ipip-always模式-网卡信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/0BDAB37F51354D8792FB29A4BA74C31F/20815)  

#### 4.4.2.3 查看物理机路由表信息  

1. 查看master节点的路由表信息   
    ![master节点-ipip-always模式-路由信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/5F72D3612DA74AD8AE2EAA6D81F6C778/20817)  
2. 查看slave1节点的路由表信息   
    ![slave1节点-ipip-always模式-路由信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/CE1E396BC3F34E469AEE8B4F8D52AC8F/20820)  
3. 查看slave2节点的路由表信息   
    ![slave2节点-ipip-always模式-路由信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/6AFC356520EB4FE8A53D0D0064DE8AF7/20822)  

#### 4.4.2.4 清理ipPool资源
calicoctl delete -f ipPool-ipip-always.yaml
![清理ipip-always资源](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/4027F409CE234E0E9EC17B89E357F159/20824)  

#### 4.4.2.5 总结：
ipip-always模式， 
- 会在每个节点上，都创建tun10, 
- 通过tun10进行流量转发  


### 4.4.3 混合ipip-bgp模式
#### 4.4.3.1 创建混合ipip-bgp模式下的ipPool资源对象  
1. 自定义混合ipip-bgp模式的ipPool资源对象  
    vim ipPool-ipip-cross-subnet.yaml
    ```
    apiVersion: v1
    kind: ipPool
    metadata: 
    cidr: 10.254.0.0/24 
    spec:
    ipip: 
        enabled: true
        mode: cross-subnet
    nat-outgoing: true
    #默认即使false, 也就是启用此ipPool
    disabled: false 

    ```

2. 创建ipPool资源  
    ```
    calicoctl create -f ipPool-ipip-cross-subnet.yaml
    ``` 
    ![创建ipip-cross-subenet模式资源](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/54FC67FDAFC34A9AB4497E8625D1EFF6/20827)  

#### 4.4.3.2 查看物理机网卡信息  

1. 查看master节点的网卡信息   
    ![master节点-ipip-cross-subnet模式-网卡信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/7EB9AC6F493C463D9CD8C787D391614E/20829)  
2. 查看slave1节点的网卡信息   
    ![slave1节点-ipip-cross-subnet模式-网卡信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/0E923DF1272A4452B8770910A2572EB9/20831)  
3. 查看slave2节点的网卡信息   
    ![slave2节点-ipip-cross-subnet模式-网卡信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/A36A426633AA4964831D1F4262F2F603/20833)  


#### 4.4.3.3 查看物理机路由表信息  

1. 查看master节点的路由表信息   
    ![master节点-ipip-cross-subnet模式-路由信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/944B13FAC4894D46873634C4B1747F7A/20836)  
2. 查看slave1节点的路由表信息   
    ![slave1节点-ipip-cross-subnet模式-路由信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/44184653658B44D39E488E9D09D10C5B/20838)  
3. 查看slave2节点的路由表信息   
    ![slave2节点-ipip-cross-subnet模式-路由信息](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/AC0C4333FC3841A2971BA03694EE5823/20840)  


#### 4.4.3.4 清理ipPool资源
    calicoctl delete -f ipPool-ipip-cross-subnet.yaml

### 4.4.4 总结  
    混合ipip-bgp模式 
    - 有些节点创建tun10, 有些节点不创建tun10  
    - 通过tun10和网卡进行流量转发   


## 4.5 总结整理:  
    1. BGP模式总结：  
        - BGP模式下，不会在服务器上创建tun10， 
        - 路由转发功能是通过服务器的网卡对流量进行转发的

    2. ipip-always模式， 
        - 会在每个节点上，都创建tun10, 
        - 通过tun10进行流量转发  

    3. 混合ipip-bgp模式 
        - 有些节点创建tun10, 有些节点不创建tun10  
        - 通过tun10和网卡进行流量转发   
        - 如果calico集群里，存在多个混合ipip-bgp模式资源对象的话，只有第一个创建的ipip-bgp资源对象有效， 
        后面创建一直处于阻塞状态， 经测试，只有删除第一个后，第二个会自动创建的； 
            - 也就是说，在calico集群中，同一时刻只能是第一个ipPool资源对象有效
    4. libnetwork插件负责从ipPool池里，获取IP，指定给docker容器的  


# 五、如果calico集群里，存在多个bgp模式的ipPool资源对象的话，哪个生效？
简单的测试了几次，没发现什么规律。  
肯定是测试不够充分。  
查看calico源码：
```
func (i IpamDriver) RequestPool(request *ipam.RequestPoolRequest) (*ipam.RequestPoolResponse, error) {
	logutils.JSONMessage("RequestPool", request)

	// Calico IPAM does not allow you to request a SubPool.
	if request.SubPool != "" {
		err := errors.New(
			"Calico IPAM does not support sub pool configuration " +
				"on 'docker create network'. Calico IP Pools " +
				"should be configured first and IP assignment is " +
				"from those pre-configured pools.",
		)
		log.Errorln(err)
		return nil, err
	}

	if len(request.Options) != 0 {
		err := errors.New("Arbitrary options are not supported")
		log.Errorln(err)
		return nil, err
	}
	var poolID string
	var pool string
	var gateway string
	if request.V6 {
		// Default the poolID to the fixed value.
		poolID = i.poolIDV6
		pool = "::/0"
		gateway = "::/0"
	} else {
		// Default the poolID to the fixed value.
		poolID = i.poolIDV4
		pool = "0.0.0.0/0"
		gateway = "0.0.0.0/0"
	}

	// If a pool (subnet on the CLI) is specified, it must match one of the
	// preconfigured Calico pools.
	if request.Pool != "" {
		poolsClient := i.client.IPPools()
		_, ipNet, err := caliconet.ParseCIDR(request.Pool)
		if err != nil {
			err := errors.New("Invalid CIDR")
			log.Errorln(err)
			return nil, err
		}

		pools, err := poolsClient.List(context.Background(), options.ListOptions{})
		if err != nil {
			log.Errorln(err)
			return nil, err
		}

		f := false
		for _, p := range pools.Items {
			if p.Spec.CIDR == ipNet.String() {
				f = true
				pool = p.Spec.CIDR
				poolID = p.Name
				break
			}
		}

		if !f {
			err := errors.New("The requested subnet must match the CIDR of a " +
				"configured Calico IP Pool.",
			)
			log.Errorln(err)
			return nil, err
		}
	}

	// We use static pool ID and CIDR. We don't need to signal the
	// The meta data includes a dummy gateway address. This prevents libnetwork
	// from requesting a gateway address from the pool since for a Calico
	// network our gateway is set to a special IP.
	resp := &ipam.RequestPoolResponse{
		PoolID: poolID,
		Pool:   pool,
		Data:   map[string]string{"com.docker.network.gateway": gateway},
	}

	logutils.JSONMessage("RequestPool response", resp)

	return resp, nil
}
```  
其中，有一块代码逻辑是：  
![RequestPool](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/C8BE40AFA06144A2B996C0888A3EE725/20842)  



# 六、问题列表  
1. docker: Error response from daemon: IpamDriver.RequestAddress: IP assignment error: No configured Calico pools.  
![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/5FB578C6439449C898523E027968E677/20774) 
查看libnetwork的日志：  
![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/8EFEA4354629491EB699A1A31270BEC3/20776)   
或者说，没有定义ip 池，  
解决措施，创建一个ipPool就可以了，



