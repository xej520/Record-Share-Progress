# <center>cfssl使用</center>  
参考文献如下:  
http://blog.51cto.com/liuzhengwei521/2120535?utm_source=oschina-app  
https://blog.csdn.net/wangzhkai/article/details/80290279    
## 基本思想： 
1. 先创建根证书CA  
再由根证书去签发其他子证书， 如 签发服务端证书，客户端证书 
2. 根证书这一侧， 需要配置策略文件，也就是给其他证书签发时，使用的配置策略  
3. 申请证书这一侧，需求提前准备一份身份说明，如国家，州，区，系统，等信息    

整个过程，有点类似于 你去办理身份证的过程，CA 公安局，证书就是你的身份证  
****
****
# 公钥基础设施(PKI)/CFSSL证书生成工具的使用  
## 公钥基础设施(PKI)  
### 基础概念  
1. CA(Certification Authority)证书?   
    指的是权威机构给我们颁发的证书。

2. 密钥?  
    就是用来加解密用的文件或者字符串。  
    密钥在非对称加密的领域里，指的是私钥和公钥，他们总是成对出现，其主要作用是加密和解密。常用的加密强度是2048bit。

3. RSA?  
    即非对称加密算法。  
    非对称加密有两个不一样的密码，一个叫私钥，另一个叫公钥，用其中一个加密的数据只能用另一个密码解开，用自己的都解不了，也就是说用公钥加密的数据只能由私钥解开

### 证书的编码格式  
- PEM(Privacy Enhanced Mail):   
    通常用于数字证书认证机构（Certificate Authorities，CA），扩展名为.pem, .crt, .cer, 和 .key。  
    内容为Base64编码的ASCII码文件，有类似"-----BEGIN CERTIFICATE-----" 和 "-----END CERTIFICATE-----"的头尾标记。  
    服务器认证证书，中级认证证书和私钥都可以储存为PEM格式（认证证书其实就是公钥）。  
    Apache和nginx等类似的服务器使用PEM格式证书。
- DER(Distinguished Encoding Rules):  
    与PEM不同之处在于其使用二进制而不是Base64编码的ASCII。  
    扩展名为.der，但也经常使用.cer用作扩展名，所有类型的认证证书和私钥都可以存储为DER格式。  
    Java使其典型使用平台

### 证书签名请求CSR 
**CSR(Certificate Signing Request)，** 

它是向CA机构申请数字×××书时使用的请求文件。在生成请求文件前，我们需要准备一对对称密钥。私钥信息自己保存，请求中会附上公钥信息以及国家，城市，域名，Email等信息，CSR中还会附上签名信息。当我们准备好CSR文件后就可以提交给CA机构，等待他们给我们签名，签好名后我们会收到crt文件，即证书。  
**注意：**  
CSR并不是证书。而是向权威证书颁发机构获得签名证书的申请。  

把CSR交给权威证书颁发机构,权威证书颁发机构对此进行签名,完成。保留好CSR,当权威证书颁发机构颁发的证书过期的时候,你还可以用同样的CSR来申请新的证书,key保持不变.

### 数字签名  
数字签名就是"非对称加密+摘要算法"，其目的不是为了加密，而是用来防止他人篡改数据。

其核心思想是：比如A要给B发送数据，A先用摘要算法得到数据的指纹，然后用A的私钥加密指纹，加密后的指纹就是A的签名，B收到数据和A的签名后，也用同样的摘要算法计算指纹，然后用A公开的公钥解密签名，比较两个指纹，如果相同，说明数据没有被篡改，确实是A发过来的数据。假设C想改A发给B的数据来欺骗B，因为篡改数据后指纹会变，要想跟A的签名里面的指纹一致，就得改签名，但由于没有A的私钥，所以改不了，如果C用自己的私钥生成一个新的签名，B收到数据后用A的公钥根本就解不开。

常用的摘要算法有MD5、SHA1、SHA256。

使用私钥对需要传输的文本的摘要进行加密，得到的密文即被称为该次传输过程的签名。

### 数字证书和公钥  
数字证书则是由证书认证机构（CA）对证书申请者真实身份验证之后，用CA的根证书对申请人的一些基本信息以及申请人的公钥进行签名（相当于加盖发证书机 构的公章）后形成的一个数字文件。实际上，数字证书就是经过CA认证过的公钥，除了公钥，还有其他的信息，比如Email，国家，城市，域名等。

## CFSSL工具  
****
### CFSSL介绍  
项目地址： https://github.com/cloudflare/cfssl

下载地址： https://pkg.cfssl.org/

参考链接： https://blog.cloudflare.com/how-to-build-your-own-public-key-infrastructure/

CFSSL是CloudFlare开源的一款PKI/TLS工具。 CFSSL 包含一个命令行工具 和一个用于 签名，验证并且捆绑TLS证书的 HTTP API 服务。 使用Go语言编写。
CFSSL包括：

- 一组用于生成自定义 TLS PKI 的工具


- cfssl程序，是CFSSL的命令行工具


- multirootca程序是可以使用多个签名密钥的证书颁发机构服务器


- mkbundle程序用于构建证书池


- cfssljson程序，从cfssl和multirootca程序获取JSON输出，并将证书，密钥，CSR和bundle写入磁盘   

PKI借助数字证书和公钥加密技术提供可信任的网络身份。通常，证书就是一个包含如下身份信息的文件：

- 证书所有组织的信息
- 公钥
- 证书颁发组织的信息
- 证书颁发组织授予的权限，如证书有效期、适用的主机名、用途等 
- 使用证书颁发组织私钥创建的数字签名  

### 安装cfssl  
这里我们只用到cfssl工具和cfssljson工具：  
```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```  
cfssl工具，子命令介绍:  

- bundle: 创建包含客户端证书的证书包
- genkey: 生成一个key(私钥)和CSR(证书签名请求)
- scan: 扫描主机问题
- revoke: 吊销证书
- certinfo: 输出给定证书的证书信息， 跟cfssl-certinfo 工具作用一样
- gencrl: 生成新的证书吊销列表
- selfsign: 生成一个新的自签名密钥和 签名证书
- print-defaults: 打印默认配置，这个默认配置可以用作模板
- serve: 启动一个HTTP API服务
- gencert: 生成新的key(密钥)和签名证书  
    - -ca：指明ca的证书  
    - -ca-key：指明ca的私钥文件  
    - -config：指明请求证书的json文件  
    - -profile：与-config中的profile对应，是指根据config中的profile段来生成证书的相关信息  
- ocspdump
- ocspsign
- info: 获取有关远程签名者的信息
- sign: 签名一个客户端证书，通过给定的CA和CA密钥，和主机名
- ocsprefresh
- ocspserve
### 创建认证中心(CA)  
CFSSL可以创建一个获取和操作证书的内部认证中心。

运行认证中心需要一个CA证书和相应的CA私钥。任何知道私钥的人都可以充当CA颁发证书。因此，私钥的保护至关重要。
  
#### 生成CA证书和私钥(root 证书和私钥) 
创建一个文件ca-csr.json： 
```
{
  "CN": "www.51yunv.com",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "51yunv",
      "OU": "ops"
    }
  ]
}
```  
术语介绍:

- CN: Common Name，浏览器使用该字段验证网站是否合法，一般写的是域名。非常重要。浏览器使用该字段验证网站是否合法
- C: Country， 国家
- L: Locality，地区，城市
- O: Organization Name，组织名称，公司名称
- OU: Organization Unit Name，组织单位名称，公司部门
- ST: State，州，省

生成CA证书和CA私钥和CSR(证书签名请求): 
```
# cfssl gencert -initca ca-csr.json | cfssljson -bare ca  ## 初始化ca
# ls ca*
ca.csr  ca-csr.json  ca-key.pem  ca.pem
```
该命令会生成运行CA所必需的文件ca-key.pem（私钥）和ca.pem（证书），还会生成ca.csr（证书签名请求），用于交叉签名或重新签名。  
小提示： 
>使用现有的CA私钥，重新生成：  
```
cfssl gencert -initca -ca-key key.pem ca-csr.json | cfssljson -bare ca
```
>使用现有的CA私钥和CA证书，重新生成： 

    cfssl gencert -renewca -ca cert.pem -ca-key key.pem

查看cert(证书信息): 
>cfssl certinfo -cert ca.pem  

查看CSR(证书签名请求)信息： 
>cfssl certinfo -csr ca.csr  

配置证书生成策略:  
配置证书生成策略，让CA软件知道颁发什么样的证书。  
# vim ca-config.json  
```
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "etcd": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "8760h"
      }
    }
  }
}
``` 
这个策略，有一个默认的配置，和一个profile，可以设置多个profile，这里的profile是etcd。  

- 默认策略，指定了证书的有效期是一年(8760h) 
- etcd策略，指定了证书的用途
- signing, 表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE
- server auth：表示 client 可以用该 CA 对 server 提供的证书进行验证
- client auth：表示 server 可以用该 CA 对 client 提供的证书进行验证  

### cfssl常用命令：  

- cfssl gencert -initca ca-csr.json | cfssljson -bare ca ## 初始化ca
- cfssl gencert -initca -ca-key key.pem ca-csr.json | cfssljson -bare ca ## 使用现有私钥, 重新生成
- cfssl certinfo -cert ca.pem
- cfssl certinfo -csr ca.csr  
**** 
****
# k8s部署之使用CFSSL创建证书  
## 安装CFSSL 
```
curl -s -L -o /bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64  

curl -s -L -o /bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 

curl -s -L -o /bin/cfssl-certinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 

chmod +x /bin/cfssl*
```
## 容器相关证书类型
**注意**  
 其实并不能这么任务，可以认为是ca颁发其他证书的策略，目前提供了三种而已，因根据具体情况来进行配置
 ```
 client certificate： 用于服务端认证客户端,例如etcdctl、etcd proxy、fleetctl、docker客户端
server certificate: 服务端使用，客户端以此验证服务端身份,例如docker服务端、kube-apiserver
peer certificate: 双向证书，用于etcd集群成员间通信

 ```  
## 创建CA证书  
### 生成默认CA配置  
```
mkdir /opt/ssl
cd /opt/ssl
cfssl print-defaults config > ca-config.json
cfssl print-defaults csr > ca-csr.json
```
### 修改ca-config.json,分别配置针对三种不同证书类型的profile,其中有效期43800h为5年  

```
{
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {
            "server": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}

```  
### 修改ca-csr.config  
```
{
    "CN": "Self Signed Ca",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "O": "Netease",
            "ST": "SH",            
            "OU": "OT"
        }    ]
}

```
### 生成CA证书和私钥  
>cfssl gencert -initca ca-csr.json | cfssljson -bare ca 

生成ca.pem、ca.csr、ca-key.pem(CA私钥,需妥善保管)

## 签发Server Certificate  
> cfssl print-defaults csr > server-csr.json  

修改默认的配置 server-csr.json  
```
vim server-csr.json
{
    "CN": "Server",
    "hosts": [
        "192.168.1.1"
       ],
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH"
        }
    ]
}

```
### 生成服务端证书和私钥
>cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server  

## 签发Client Certificate
> cfssl print-defaults csr > admin-csr.json  

修改默认的配置 admin-csr.json  
```
vim admin-csr.json
{
    "CN": "Client",
    "hosts": [],
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH"
        }
    ]
}

```
### 生成客户端证书和私钥  
>cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

## peer certificate  
### 生成peer端证书和私钥
> cfssl print-defaults csr > kube-proxy-csr.json
修改默认的配置 admin-csr.json   
```
vim kube-proxy-csr.json
{
    "CN": "member1",
    "hosts": [
        "192.168.1.1"
    ],
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH"
        }
    ]
}

```
### 为节点member1生成证书和私钥: 
>cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy 
针对etcd服务,每个etcd节点上按照上述方法生成相应的证书和私钥  
## 最后校验证书 
校验生成的证书是否和配置相符  
```
openssl x509 -in ca.pem -text -noout
openssl x509 -in server.pem -text -noout
openssl x509 -in client.pem -text -noout
```

