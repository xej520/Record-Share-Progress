参考文献：  
https://blog.csdn.net/hxpjava1/article/details/80061468  
https://www.jianshu.com/p/0fc3c5e78eff  

# 一、简述  
- 单节点部署etcd   
- 添加认证  

# 二、CFSSL工具的安装  
## 2.1 下载可执行的cfssl二进制文件  
```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64  
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64  
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 
```  
## 2.2 添加执行权限
```
chmod +x cfssl*
```
## 2.3 更新名称，移动到/usr/local/bin  
1. 创建目录  
    ```
    mkdir -p /usr/local/bin
    ```
2. 移动命令 
    ```
    cp -v cfssl_linux-amd64 /usr/local/bin/cfssl 
    cp -v cfssljson_linux-amd64 /usr/local/bin/cfssljson 
    cp -v cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo 
    ``` 
3. 查看命令  
    ```
    ls /usr/local/bin
    ```
## 2.4 主要操作步骤图形化展示  
如下图所示：  
![下载cfssl可执行二进制文件](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/ECBB47D8CBD04F438712F7E2D102930F/20614)  
![对二进制文件重新命名](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/11E427E8ECE04FF18002CE09F7F099ED/20617)  


# 三、创建证书  
## 3.1 创建根证书存放的位置 
```
mkdir -p /etc/ssl
mkdir -p /etc/ssl/ca 
cd /etc/ssl/ca 
```

## 3.2 创建CA证书  
1. 创建ca配置文件(ca-config.json)  
    主要作用就是，不同组件证书的形式，有效期是不一样的，因此，在ca配置中心，应该有一个配置文件，来具体的配置，就是ca-config.json  
    ```
    { 
        "signing": { 
            "default": { 
                "expiry": "876000h"
            }, 
            "profiles": { 
                "etcd": { 
                    "usages": [ 
                        "signing", 
                        "key encipherment", 
                        "server auth", 
                        "client auth"
                    ], 
                    "expiry": "876000h"
                }
            }
        }
    }

    ```
    __字段说明__    
    ```
    "ca-config.json"：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile； 
    "signing"：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE； 
    "server auth"：表示client可以用该 CA 对server提供的证书进行验证； 
    "client auth"：表示server可以用该CA对client提供的证书进行验证；

    ```

2. 创建ca证书签名请求(ca-csr.json)  
    什么意思呢？  
    就是申请证书的人，你得把你的基本信息告诉证书颁发中心吧，证书颁发中心，根据你填写的基本信息，来生成你的证书啊。 
    ca-csr.json就是属于你的基本信息，  
    你申请办理户口，身份证等证件时，不也需要填写一些基本信息么， 作用类似的。  
    ```
    { 
        "CN": "etcd", 
        "key": { 
            "algo": "rsa", 
            "size": 2048
            }, 
        "names": [
            { 
                "C": "CN", 
                "ST": "shenzhen", 
                "L": "shenzhen", 
                "O": "etcd", 
                "OU": "System"
            }
        ] 
    }
    ```  
    __参数介绍__  
    ```
    "CN"：Common Name，etcd 从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法； 
    "O"：Organization，etcd 从证书中提取该字段作为请求用户所属的组 (Group)； 
    这两个参数在后面的kubernetes启用RBAC模式中很重要，因为需要设置kubelet、admin等角色权限，那么在配置证书的时候就必须配置对了，具体后面在部署kubernetes的时候会进行讲解。 "在etcd这两个参数没太大的重要意义，跟着配置就好。"

    ```
3. 开始生成CA的证书和私钥(就是根证书之类文件)  
    ```
    cfssl gencert -initca ca-csr.json | cfssljson -bare ca
    ```


## 3.3 颁发etcd证书  
0. 创建etcd 证书存放的位置  
    ```
    mkdir -p /etc/ssl/etcd
    cd /etc/ssl/etcd
    ```
1. 创建etcd-csr.json文件  
    就是填写一些etcd组件的基本信息  
    认证中心根据这个文件，生成证书  
    ```
    {
    "CN": "etcd",
    "hosts": [
        "127.0.0.1",
        "172.16.91.195"
    ],
    "key": {	
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
        "C": "CN",
        "ST": "shenzhen",
        "L": "shenzhen",
        "O": "etcd",
        "OU": "System"
        }
    ]
    }
    ```
    __说明:__    
    ```
    [^_^]:
        如果 hosts 字段不为空则需要指定授权使用该证书的 IP 或域名列表，由于该证书后续被 etcd 集群使用，所以填写IP即可。
    [>_<]:
        因为本次部署etcd是单台，那么则需要填写单台的IP地址即可。  
        “C”：国家  
    该"hosts"是可以使用该证书域名列表。‘CN’，kube-apiserver从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法；   

    该"names"值实际上是名称对象的列表。每个名称对象应至少包含一个“C”，“L”，“O”，“OU”或“ST”值（或这些的任意组合）。这些值是：         
    “L”：地区或城市（如城市或城镇名称）
    “O”：组织 Organization，kube-apiserver从证书中提取该字段作为请求用户所属的组 (Group)；
    “OU”：组织单位，如负责拥有密钥的部门; 它也可以用于“做生意”（DBS）的名称
    “ST”：州或省
    ```

2. 生成etcd证书  
    ```
    cfssl gencert -ca=/etc/ssl/ca/ca.pem -ca-key=/etc/ssl/ca/ca-key.pem -config=/etc/ssl/ca/ca-config.json -profile=etcd etcd-csr.json | cfssljson -bare etcd  
    ```
3. 查看证书生成情况  
    ```
    ls /etc/ssl/etcd
    ```

## 3.4 主要操作步骤展示  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/F1A4E9B0ACC04770B61CBF4BA43C3BB6/20619)   
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/57225E5500A841908BABBF981A638FB3/20622)   

# 四、部署etcd服务 
## 4.1 安装etcd  
```
yum install -y etcd
```

## 4.2 编写etcd配置文件  
### 4.2.1 创建etcd.service配置文件
让systemd来管理etcd服务  
1. etcd.service(不带认证参数)  
    vim /usr/lib/systemd/system/etcd.service  
    ```
    [Unit]
    Description=Etcd Server
    After=network.target
    After=network-online.target
    Wants=network-online.target
    Documentation=https://github.com/coreos

    [Service]
    Type=notify
    WorkingDirectory=/var/lib/etcd/
    ExecStart=/usr/bin/etcd \
    --name=harbor \
    --data-dir=/var/lib/etcd \  
    --listen-client-urls=http://172.16.91.195:2379,http://127.0.0.1:2379 \
    --advertise-client-urls=http://172.16.91.195:2379 \
    
    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target
    ```  
2. etcd.service(带有认证参数)  
    vim /usr/lib/systemd/system/etcd.service
    ```
    [Unit]
    Description=Etcd Server
    After=network.target
    After=network-online.target
    Wants=network-online.target
    Documentation=https://github.com/coreos

    [Service]
    Type=notify
    WorkingDirectory=/var/lib/etcd/
    ExecStart=/usr/bin/etcd \
    --name=172.16.91.195 \
    --data-dir=/var/lib/etcd \ 
    --initial-cluster-state=new \
    --listen-client-urls=https://172.16.91.195:2379,http://127.0.0.1:2379 \
    --advertise-client-urls=https://172.16.91.195:2379 \
    --listen-peer-urls=https://172.16.91.195:2380 \
    --initial-advertise-peer-urls=https://172.16.91.195:2380 \ 
    --cert-file=/etc/ssl/ca/etcd/etcd.pem \
    --key-file=/etc/ssl/ca/etcd/etcd-key.pem \
    --peer-cert-file=/etc/ssl/etcd/etcd.pem \
    --peer-key-file=/etc/ssl/etcd/etcd-key.pem \
    --peer-trusted-ca-file=/etc/ssl/ca/ca.pem \
    --trusted-ca-file=/etc/ssl/ca/ca.pem   \
    --initial-cluster-token=etcd-cluster-0 


    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target

    ```

    ![参数之间不允许有空白行](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/83F7D21201A64479BF4752F0C55A3035/20625)  

## 4.3 启动etcd服务  
```
mkdir -p /var/lib/etcd
systemctl daemon-reload
systemctl start etcd
systemctl status etcd
```

# 五、客户端测试etcd  
1. 查看etcd集群状态(不带认证)
    ```
    etcdctl cluster-health
    ```
2. 查看etcd集群状态(带有认证)
    - etcdctl --ca-file=/etc/ssl/ca/ca.pem --cert-file=/etc/ssl/etcd/etcd.pem --key-file=/etc/ssl/etcd/etcd-key.pem --endpoints=https://127.0.0.1:2379 cluster-health  
    - etcdctl --ca-file=/etc/ssl/ca/ca.pem --cert-file=/etc/ssl/etcd/etcd.pem --key-file=/etc/ssl/etcd/etcd-key.pem --endpoints=https://172.16.91.195:2379 cluster-health
    - etcdctl --ca-file=/etc/ssl/ca/ca.pem --endpoints=https://172.16.91.195:2379 cluster-health  


# 六、如何做到只监听本机(只允许本机使用etcd服务)
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/7D8560A4FB5E457FA3E189D7B9231987/20634)  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/E8BA35AF7C7C40CAACA7C47C482FFAC3/20630)  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/F4D9313B87A1415791AE6234B0C52611/20639)
