__以静态方式部署etcd集群__   

### __何为静态方式部署呢？__  
>预先已知etcd集群中有哪些节点，在启动时通过--initial-cluster参数直接指定好etcd的各个节点地址   

# 一、测试环境  
|主机名|系统版本|IP|服务|
|:---|:---|:---|:---|
|master|centos7.5|172.16.91.195|etcd|
|slave01|centos7.5|172.16.91.196|etcd|
|slave02|centos7.5|172.16.91.197|etcd|

# 二、部署方式说明  
目前大概有以下几种部署方式吧：   
- yum 
- docker 
- rpm 
- k8s  
本文采用yum方式安装部署  
其他方式，docker，rpm方式可以参考其他博文，或者官网  
https://github.com/etcd-io/etcd/releases  
## 2.1 通过yum方式安装的etcd服务，配置文件位置如下：  
|名称|位置|
|:---|:---|
|etcd|/usr/bin/etcd|
|etcdctl|/usr/bin/etcdctl|
|etcd.service|/lib/systemd/system/etcd.service|
|etcd.conf|/etc/etcd/etcd.conf|


# 三、部署etcd服务(不带认证)  
## 3.1 通过yum方式部署安装etcd  
在三台节点上，分别使用下面的命令，来安装etcd
>yum install -y etcd  

注意：  
- 通过yum方式安装的etcd服务，是交给linux的系统服务来维护生命周期的
- 此时etcd服务并没有起来，需要手动启动etcd服务  
    ![](https://note.youdao.com/yws/public/resource/beefe632a59e659716553180a808c6bf/xmlnote/2703F30326BC41F0A5E61C2C42A12C47/20679)  

由于默认的配置只能自己本节点使用，因此这里先不启动etcd服务  

## 3.2 更新配置文件  
### 3.2.1 更新etcd.service  
1. master节点，配置文件如下所示：  
	```
	[root@master lib]# more /lib/systemd/system/etcd.service 
	[Unit]
	Description=Etcd Server
	After=network.target
	After=network-online.target
	Wants=network-online.target

	[Service]
	Type=notify
	WorkingDirectory=/var/lib/etcd/
	EnvironmentFile=-/etc/etcd/etcd.conf
	User=root
	# set GOMAXPROCS to number of processors
	ExecStart=/usr/bin/etcd  \
		--name=${ETCD_NAME} \
		--data-dir=${ETCD_DATA_DIR} \
		--listen-client-urls=${ETCD_LISTEN_CLIENT_URLS} \
		--listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
		--advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \
		--initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \
		--initial-advertise-peer-urls=${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
		--initial-cluster-state=${ETCD_INITIAL_CLUSTER_STATE} \
		--initial-cluster=${ETCD_INITIAL_CLUSTER}


	Restart=on-failure
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target
	```

2. slave1节点，配置文件如下所示： 
	```
	[root@slave1 lib]# more /lib/systemd/system/etcd.service 
	[Unit]
	Description=Etcd Server
	After=network.target
	After=network-online.target
	Wants=network-online.target

	[Service]
	Type=notify
	WorkingDirectory=/var/lib/etcd/
	EnvironmentFile=-/etc/etcd/etcd.conf
	User=root
	# set GOMAXPROCS to number of processors
	ExecStart=/usr/bin/etcd  \
		--name=${ETCD_NAME} \
		--data-dir=${ETCD_DATA_DIR} \
		--listen-client-urls=${ETCD_LISTEN_CLIENT_URLS} \
		--listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
		--advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \
		--initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \
		--initial-advertise-peer-urls=${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
		--initial-cluster-state=${ETCD_INITIAL_CLUSTER_STATE} \
		--initial-cluster=${ETCD_INITIAL_CLUSTER}

	Restart=on-failure
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target
	```

3. slave2节点，配置文件如下所示：  
	```
	[root@slave2 ~]# more /lib/systemd/system/etcd.service 
	[Unit]
	Description=Etcd Server
	After=network.target
	After=network-online.target
	Wants=network-online.target

	[Service]
	Type=notify
	WorkingDirectory=/var/lib/etcd/
	EnvironmentFile=-/etc/etcd/etcd.conf
	User=root
	# set GOMAXPROCS to number of processors
	ExecStart=/usr/bin/etcd  \
		--name=${ETCD_NAME} \
		--data-dir=${ETCD_DATA_DIR} \
		--listen-client-urls=${ETCD_LISTEN_CLIENT_URLS} \
		--listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
		--advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \
		--initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \
		--initial-advertise-peer-urls=${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
		--initial-cluster-state=${ETCD_INITIAL_CLUSTER_STATE} \
		--initial-cluster=${ETCD_INITIAL_CLUSTER}


	Restart=on-failure
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target
	```

### 3.2.2 更新etcd.conf  
1. master节点，配置文件如下所示：  
	```
	[root@master lib]# more /etc/etcd/etcd.conf 
	#[Member]
	ETCD_DATA_DIR="/var/lib/etcd-jingtai/data"
	ETCD_LISTEN_PEER_URLS="http://172.16.91.195:2380"
	ETCD_LISTEN_CLIENT_URLS="http://127.0.0.1:2379,http://172.16.91.195:2379"
	ETCD_NAME="master"

	#[Clustering]
	ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.16.91.195:2380"
	ETCD_ADVERTISE_CLIENT_URLS="http://127.0.0.1:2379,http://172.16.91.195:2379"
	ETCD_INITIAL_CLUSTER="master=http://172.16.91.195:2380,slave1=http://172.16.91.196:2380,slave2=http://172.16.91.197:2380"
	ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster001"
	ETCD_INITIAL_CLUSTER_STATE="new"
	```

2. slave1节点，配置文件如下所示： 
	```
	[root@slave1 etcd-jingtai]# more /etc/etcd/etcd.conf 
	#[Member]
	ETCD_DATA_DIR="/var/lib/etcd-jingtai/data"
	ETCD_LISTEN_PEER_URLS="http://172.16.91.196:2380"
	ETCD_LISTEN_CLIENT_URLS="http://127.0.0.1:2379,http://172.16.91.196:2379"
	ETCD_NAME="slave1"

	#[Clustering]
	ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.16.91.196:2380"
	ETCD_ADVERTISE_CLIENT_URLS="http://127.0.0.1:2379,http://172.16.91.196:2379"
	ETCD_INITIAL_CLUSTER="master=http://172.16.91.195:2380,slave1=http://172.16.91.196:2380,slave2=http://172.16.91.197:2380"
	ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster001"
	ETCD_INITIAL_CLUSTER_STATE="new"
	```

3. slave2节点，配置文件如下所示：  
	```
	[root@slave2 ~]# more /etc/etcd/etcd.conf 
	#[Member]
	ETCD_DATA_DIR="/var/lib/etcd-jingtai/data"
	ETCD_LISTEN_PEER_URLS="http://172.16.91.197:2380"
	ETCD_LISTEN_CLIENT_URLS="http://127.0.0.1:2379,http://172.16.91.197:2379"
	ETCD_NAME="slave2"

	#[Clustering]
	ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.16.91.197:2380"
	ETCD_ADVERTISE_CLIENT_URLS="http://127.0.0.1:2379,http://172.16.91.197:2379"
	ETCD_INITIAL_CLUSTER="master=http://172.16.91.195:2380,slave1=http://172.16.91.196:2380,slave2=http://172.16.91.197:2380"
	ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster001"
	ETCD_INITIAL_CLUSTER_STATE="new"
	```

## 3.3 重启etcd服务  
1. 创建data目录 
>mkdir -p /var/lib/etcd-jingtai/data  
2. 启动服务
>systemctl daemon-reload  
systemctl enable etcd   
systemctl start etcd  
systemctl status etcd   

# 四、测试etcd服务  
查看服务状态
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/06898B4407D04143A64EA8ADE621FFA9/20738)  


# 五、问题列表  
1. request sent was ignored (cluster ID mismatch: peer[3fc33c18ca0433f2]=3b1291d31874b132, local=9818358e4a92d2d9)   request cluster ID mismatch (got 9818358e4a92d2d9 want 3b29dfa314757683)  
	现象是： 一个etcd集群里，有两个leader， 或者说，这两个etcd实例有各自的集群ID，  
	第3个etcd实例，分别向这两个发送请求
	![master](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/43C62797D291426ABD1666822F2FB1A9/20732)  
	![slave1](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/0DD0299BD8754F6E9EA06FBDC3E38A76/20730)  
	![slave2](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/FDD732521D0E4B859FB3C3C8D7231384/20734)  

	原因：  
		在搭建集群前，先进行了单实例的etcd部署测试， 即master, slave1节点分别进行了单实例部署测试，   
		也就是历史数据的影响。   
		删除历史数据即可。   
		rm -rf /var/lib/etcd/*

# 六、说明：  
在部署过程中，遇到的主要问题有： 
- 配置文件名称写错了  
- etcd.service里，启动服务ExecStart里不能有空白行，注释  
- 权限问题，User=etcd,改成root  
- 历史数据的影响  















