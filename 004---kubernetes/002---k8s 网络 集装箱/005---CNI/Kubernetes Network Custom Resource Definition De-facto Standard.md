
`Kubernetes Network Custom Resource Definition De-facto Standard`
# 1. Goals  
本文档提出了将Kubernetes pod连接到`一个或多个逻辑或物理网络`的要求和过程的规范，包括使用`容器网络接口（CNI）`连接pod网络的插件的要求。

## 1.1 Non-Goals of Version 1 
出于简化和/或需要达成一些合理的共识，本文件特别没有涉及某些问题。 这些问题可能会在本规范的未来版本中得到解决。

### 1.1.1 Scheduling and resource management
在资源管理工作组中正在努力整合调度和资源管理（例如，确保节点没有分配比可用网络资源更多的pod），其中涉及该工作组和SIG网络的各个成员。   
本规范的未来版本可能包含资源管理的各个方面。

### 1.1.2 对Kubernetes API的更改
此规范明确避免了对现有Kubernetes API的更改。   
这些变化需要更长的上游流程，但希望通过证明概念并提供多个pod网络附件的一个或多个实际实现，  
该文档可以作为未来这些变化的基础。

### 1.1.3 Interaction with the Kubernetes API  
未指定其他pod网络附件与Kubernetes API及其对象（如服务Services，端点Endpoints，代理proxies等）的交互。 
SIG网络在2017年7月/ 8月详细讨论了该主题，并决定需要更改Kubernetes API，这是该规范的明确非目标。

### 1.1.4 Changes to the Kubernetes CNI Driver
为确保轻松使用此规范及其实现，Kubernetes CNI驱动程序不需要进行任何更改（例如pkg / kubelet / dockershim / network / cni / *）。   
已经提出了对驱动程序的更改以支持多个pod网络附件，并且已经编写了多个概念验证，但是对于多个附件应该如何工作以及因此CNI驱动程序中需要进行哪些更改尚未达成上游共识。

### 1.1.5 Enabling Implementations That Are Not CNI Plugins(启用不是CNI插件的实现)
此规范仅尝试启用Kubernetes网络插件，该插件使用像dockershim，CRI-O和rkt这样的Kubernetes运行时的CNI驱动程序。 该规范期望通过实现来引导pod网络附件操作，该实现需要是CNI插件。 该规范的未来版本可能会重新审视此要求。

# 2. Definitions
## 2.1 Implementation
实现此规范的Kubernetes网络插件;  
从Kubernetes 1.11开始，这被定义为符合容器网络接口（CNI）规范v0.1.0或更高版本的插件。  
应该将Kubernetes配置为调用所有pod网络操作的实现，然后实现根据pod的注释和本规范中定义的Custom Resources确定要执行的其他操作。
## 2.2 Kubernetes Cluster-Wide Default Network
根据Kubernetes的当前行为和要求连接所有pod的网络。

## 2.3 Network Attachment(Network Attachment)   
允许pod直接与给定逻辑或物理网络通信的方法。   
通常（但不一定）每个附件采用放置在pod的网络命名空间中的内核网络接口的形式。  
每个附件可能导致分配给pod的零个或多个IP地址。

## 2.4 CNI Delegating Plugin
符合本规范的实现，它将pod网络附件/分离操作委托给符合CNI规范的其他插件。 例子包括Multus和CNI-Genie。

此规范将CNI Delegating Plugin的特定要求和建议置于标记为此类的部分下。   
如果实现不是CNI委托插件，则可以忽略这些标记部分及其任何子部分中的要求和建议。

# 3. NetworkAttachmentDefinition Object
此规范定义NetworkAttachmentDefinition自定义资源对象，该对象描述如何将pod连接到对象引用的逻辑或物理网络。  
## 3.1 Custom Resource Definition (CRD)  
CRD告诉Kubernetes API如何公开NetworkAttachmentDefinition对象。 请参阅下面的对象本身的定义。
```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: network-attachment-definitions.k8s.cni.cncf.io
spec:
  group: k8s.cni.cncf.io
  version: v1
  scope: Namespaced
  names:
	plural: network-attachment-definitions
	singular: network-attachment-definition
	kind: NetworkAttachmentDefinition
	shortNames:
	- net-attach-def
  validation:
    openAPIV3Schema:
      properties:
        spec:
          properties:
            config:
          	   type: string
```

## 3.2 NetworkAttachmentDefinition Object Definition  
NetworkAttachmentDefinition对象本身仅包含“spec”部分。 其定义（以Go形式）应为：
```
type NetworkAttachmentDefinition struct {
    metav1.TypeMeta
    // Note that ObjectMeta is mandatory, as an object
    // name is required
    metav1.ObjectMeta

    // Specification describing how to add or remove network
    // attachments for a Pod. In the absence of valid keys in
    // the Spec field, the implementation shall attach/detach an
    // implementation-known network referenced by the object’s
    // name.
    // +optional
    Spec NetworkAttachmentDefinitionSpec `json:"spec"`
}


type NetworkAttachmentDefinitionSpec struct {
    // Config contains a standard JSON-encoded CNI configuration
    // or configuration list which defines the plugin chain to
    // execute. The CNI configuration may omit the 'name' field
    // which will be populated by the implementation when the
    // Config is passed to CNI delegate plugins.
    // +optional
    Config string `json:"config,omitempty"`
}

```
### 3.2.1 YAML Example: CNI config JSON in object
```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: a-bridge-network
spec:
  config: '{
    "cniVersion": "0.3.0",
    "name": "a-bridge-network",
    "type": "bridge",
    "bridge": "br0",
    "isGateway": true,
    "ipam": {
      "type": "host-local",
      "subnet": "192.168.5.0/24",
      "dataDir": "/mnt/cluster-ipam"
    }
}'

```
### 3.2.2 YAML Example: CNI configlist JSON in object  
```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: another-bridge-network
spec:
  config: '{
  "cniVersion": "0.3.0",
  "name": "another-bridge-network",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "br0",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.5.0/24"
      }
    },
    {
      "type": "port-forwarding"
    },
    {
      "type": "tuning",
      "sysctl": {
        "net.ipv4.conf.all.log_martians": "1"
      }
    }
  ]
}'

```
### 3.2.3 YAML Example: Limited CNI config required ("thick" plugin)
```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: a-bridge-network
spec:
  config: '{
    "cniVersion": "0.3.0",
    "type": "awesome-plugin"
}'

```
### 3.2.4 YAML Example: Implementation-specific Network Reference
```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: a-bridge-network

```
## 3.3 NetworkAttachmentDefinition Object Naming Rules 
根据Kubernetes验证命名空间名称的方式，有效的NetworkAttachmentDefinition对象名称必须由DNS-1123标签格式的单元组成。 建议每个DNS-1123标签单元不超过63个字符。  
“DNS-1123标签必须由小写字母数字字符或' - '组成，并且必须以字母数字字符开头和结尾”
Kubernetes错误消息

可以通过匹配此正则表达式来验证标签：  
```
[a-z0-9]([-a-z0-9]*[a-z0-9])?
```
### 3.3.1 Examples 
```
example-namespace/attachment-name
attachment-name
```

## 3.4 CNI Delegating Plugin Requirements  
对于CNI委托插件的实现，该实现将实际的附件/分离“委托”给一个或多个附加的CNI插件。 NetworkAttachmentDefinition对象包含确定要执行哪些CNI插件以及传递给它们的选项的必要信息。  

### 3.4.1 Determining CNI Plugins for a NetworkAttachmentDefinition Object
CNI委托插件必须按列出的顺序使用以下规则来确定哪个CNI插件要执行以便由NetworkAttachmentDefinition对象描述的pod对给定网络的附件执行：
1. 如果NetworkAttachmentDefinition.Spec中存在“config”键，则键值的内容用于根据CNI规范执行插件  
2. 如果磁盘上存在CNI .configlist文件，其JSON“name”键与NetworkAttachmentDefinition对象的名称匹配，则会加载其内容并用于根据CNI规范执行插件。  
3. 如果磁盘上存在CNI.config文件，其JSON“name”键与NetworkAttachmentDefinition对象的名称匹配，则会加载其内容并用于根据CNI规范执行插件。  
4. 否则，网络请求必须失败   
### 3.4.2 Spec.Config and the CNI JSON 'name' Field
如果Spec.Config键有效但其数据省略了CNI JSON'name'字段，则CNI委托插件应在将CNI JSON配置发送到委托插件之前，将包含NetworkAttachmentDefinition对象名称的'name'字段添加到CNI JSON。   
这旨在简化“厚”插件的配置，其中委托插件需要最少的配置。

### 3.4.2.1 "Name" Injection Example
给定以下NetworkAttachmentDefinition对象：  
```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: a-bridge-network
spec:
  config: '{
    "cniVersion": "0.3.0",
    "type": "awesome-plugin"
  }'

```

CNI委托插件将以下CNI JSON配置发送到'awesome-plugin'二进制文件，该二进制文件通过将NetworkAttachmentDefinition对象名称作为其数据注入“name”字段而生成：
```
{
    "cniVersion": "0.3.0",
    "name": "a-bridge-network",
    "type": "awesome-plugin"
}

```
### 3.4.3 NetworkAttachmentDefinition Object and CNI JSON Configuration Naming Considerations  
根据Kubernetes要求，每个NetworkAttachmentDefinition对象必须具有名称。 CNI规范强烈建议CNI JSON配置包含名称，未来的CNI规范版本将需要名称。  
此规范强烈建议NetworkAttachmentDefinition对象名称与NetworkAttachmentDefinition对象引用的CNI JSON配置中的“name”键匹配（无论是存储在磁盘上还是存储在NetworkAttachmentDefinition.Spec.Config键中）。  
这减少了用户的混淆，并使NetworkAttachmentDefinition对象和CNI配置之间的映射更加清晰。  
强烈建议使用此匹配，因为它可能会将Kubernetes API对象命名要求放在外部定义的资源上。   
命名最终由集群管理员决定。  

# 4. Network Attachment Selection Annotation  
要选择应附加pod的一个或多个辅助（“sidecar”）网络，此规范定义Pod对象注释。 由于此批注选择的附件是辅助附件，因此Kubernetes本身不知道这些网络附件，并且可能无法通过标准Kubernetes API获取有关它们的信息。

网络附件选择注释可用于选择群集范围内的默认网络的附加附件，超出所需的初始群集范围的默认网络附件。  

## 4.1 Annotation Name and Format
Pod对象注释名称应为“k8s.v1.cni.cncf.io/networks”。 注释值应以两种可能的格式之一指定，如下所述。 本规范的实现必须支持这两种格式。  
请注意，即使注释引用的对象是NetworkAttachmentDefinition对象，注释的名称也是“k8s.v1.cni.cncf.io/networks”。 这是故意的。  

### 4.1.1 Comma-delimited Format  
此格式旨在是一种非常简单，用户友好的格式，仅由逗号分隔的NetworkAttachmentDefinition对象引用组成。 每个对象引用的格式是（a）<NetworkAttachmentDefinition对象名称>以引用pod命名空间中的NetworkAttachmentDefinition对象，或（b）<NetworkAttachmentDefinition对象命名空间> / <NetworkAttachmentDefinition对象名称>以引用不同命名空间中的NetworkAttachmentDefinition对象。   
```
kind: Pod
metadata:
  name: my-pod
  namespace: my-namespace
  annotations:
    k8s.v1.cni.cncf.io/networks: net-a,net-b,other-ns/net-c
```
### 4.1.2 JSON List Format
此格式允许用户为网络附件提供特定于pod的要求。 
例如，如果最终处理网络附件的插件支持这些选项，则这些选项可能包括特定的IP地址或MAC地址，无论是实现本身还是委托插件。  
```
kind: Pod
metadata:
  name: my-pod
  namespace: my-namespace
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [
        {"name":"net-a"},
        {
          "name":"net-b",
          "ips": ["1.2.3.4"],
          "mac": "aa:bb:cc:dd:ee:ff"
        },
        {
          "name":"net-c",
          "namespace":"other-ns"
        }
      ]

```    
#### 4.1.2.1 JSON List Format Key Definitions
为此格式的网络列表中的每个网络附件映射定义以下键。 保留所有不包含句点字符的键名称，以确保将来可以扩展此规范。 写入除本规范中定义的密钥之外的密钥的实现必须使用反向域名表示法（例如“org.foo.bar.key-name”）来命名非标准密钥。  
#### 4.1.2.1.1 "name"  
带有string类型值的必需键是NetworkAttachmentDefinition对象的名称，可以是Pod的命名空间（如果“namespace”键缺失或为空），也可以是“namespace”键指定的另一个命名空间。  
#### 4.1.2.1.2 "namespace"
这个值为string类型的可选键是由“name”键命名的NetworkAttachmentDefinition对象的名称空间。  
#### 4.1.2.1.3 "ips"  
这个值为string-array的可选键需要处理此网络附件的插件将给定的IP地址分配给pod。 此键的值必须至少包含一个数组元素，并且每个元素必须是有效的IPv4或IPv6地址。 如果该值无效，则网络附件选择注释应无效并由实现忽略。    
##### 4.1.2.1.3.1 "ips" Example   
```
  annotations:
     k8s.v1.cni.cncf.io/networks: |
      [
        {
          "name":"net-b",
          "ips": ["10.2.2.42", "2001:db8::5"]
        }
      ]

```   
##### 4.1.2.1.3.2 CNI Delegating Plugin Requirements
CNI委托插件必须向CNI“args”映射添加“ips”键，并将其值设置为符合“ips”键值的“ips”键值的转换，如CNI的CONVENTIONS.md中所述。 由于该规范要求实现遵循“ips”，但插件可能会忽略CNI“args”，因此实现必须确保在返回的CNI请求结构中将所请求的IP地址分配给网络附件的接口; 如果尚未分配，则实施必须使网络附件失败。

鉴于紧接在前的网络附件选择注释示例，CNI委托插件会将数据转换为以下CNI JSON配置片段，该片段在“net-b”的CNI调用中传递给每个插件：  

```
{
  …
  "args":{
    "cni":{
      "ips": ["10.2.2.42", "2001:db8::5"]
    }
  }
  …
}

```
##### 4.1.2.1.4 "mac"
具有值类型字符串的此可选键需要处理此网络附件的插件将给定的MAC地址分配给pod。 此密钥的值必须包含有效的6字节以太网MAC地址或有效的20字节IP-over-InfiniBand硬件地址（如RFC4391第9.1.1节中所述）。 如果该值无效，则网络附件选择注释应无效并由实现忽略。    


###### 4.1.2.1.4.1 "mac" Example  

```
  annotations:
     k8s.v1.cni.cncf.io/networks: |
      [
        {
          "name":"net-b",
          "mac": "02:23:45:67:89:01"
        }
      ]

```
###### 4.1.2.1.4.2 CNI Delegating Plugin Requirements  
实现必须向CNI“args”映射添加“mac”密钥（如CNI的CONVENTIONS.md中所述），并将其值设置为“mac”密钥值的转换。 由于此规范要求实现遵守“mac”，但插件可能会忽略CNI“args”，因此实现必须确保在返回的CNI请求结构中将请求的MAC地址分配给网络附件的接口; 如果尚未分配，则实施必须使网络附件失败。

鉴于紧接在前的网络附件选择注释示例，CNI委托插件会将数据转换为以下CNI JSON配置片段，该片段在“net-b”的CNI调用中传递给每个插件：  
```
{
  …
  "args":{
    "cni":{
      "mac": "02:23:45:67:89:01"
    }
  }
  …
}

```
##### 4.1.2.1.5 "interface"
此值为string类型的可选键要求实现使用此网络附件生成的pod接口的给定名称。 此键的值必须是有效的Linux内核接口名称。 如果该值无效，则网络附件选择注释应无效并由实现忽略。  
如果先前的网络附件已使用所请求的接口名称，则实施必须使当前网络附件失败。  
###### 4.1.2.1.5.1 CNI Delegating Plugin Requirements 
“接口”键需要CNI委托插件在调用此网络附件的CNI插件时将CNI_IFNAME环境变量设置为给定值。  
## 4.2 Multiple Attachments to the Same Network
Pod可以通过网络附件选择注释多次请求连接到同一网络。 这些请求中的每一个都被视为单独的附件，必须由实现作为单独的操作进行处理，并且必须生成单独的“网络附件状态注释”条目。  
### 4.2.1 CNI Delegating Plugin Requirements
网络附件选择注释中的每个网络引用对应于NetworkAttachmentDefinition对象描述的CNI插件/配置列表调用。 由于CNI规范将网络附件定义为[容器ID，网络名称，CNI_IFNAME]的唯一元组，因此CNI委托插件必须确保给定网络附件的所有CNI操作（例如ADD或DEL）使用相同的唯一元组， 应该创建如下：  

1. 容器ID：由运行时给出
2. 网络名称：存在于（或通过）NetworkAttachmentDefinition对象中
3. CNI_IFNAME：如果没有由网络附件选择注释另外指定，由CNI委托插件为给定附件生成，并且在给定[容器ID，网络名称]元组的所有附件中必须是唯一的。

### 4.2.2 Example  
```
  annotations:
    k8s.v1.cni.cncf.io/networks: net-a,net-a
```
在此示例中，实现必须附加“net-a”两次，并且每个附件将导致“网络附件状态注释”列表中的单独条目。  
# 5. Network Attachment Status Annotation
为了确保通过Kubernetes API可以获得网络附件的结果，实现可以将附件操作的结果发布到请求附件的pod对象上的注释。   
注释的名称应为“k8s.v1.cni.cncf.io/network-status”，其值应为JSON编码的地图列表。 列表中的每个元素应该是由如下所述的网络附加操作的结果组成的映射。  
## 5.1 Source of Status Information
网络附件操作的状态图应包含从给定网络的附件操作结果中获取的信息。 
仅在pod本身内部有用的状态信息（如IP路由）不需要是状态映射的一部分，因为此信息通常与Kubernetes API客户端无关。   
### 5.1.1 CNI Delegating Plugin Requirements
附件的状态应取自CNI Result对象的第一个沙箱接口，用于该附件的CNI ADD或GET调用。 如果ADD或GET调用未返回结果（CNI规范当前允许），则实现应添加最小状态映射，如下所述。  

实现应尽可能使用CNI Result对象中提供的尽可能多的信息来构建网络附件状态注释。 如果操作返回版本0.3.0或更高版本的CNI Result对象，则可以使用“interfaces”，“ips”和“dns”字段轻松构建网络附件状态注释。 如果操作返回的CNI Result对象小于0.3.0版，则实现应该从CNI Result中构建尽可能多的Network Attachment Status Annotation，并且可以填充剩余的Annotation字段（例如'interface'和'mac '）通过它希望的任何方式。

## 5.2 Cluster-Wide Default Network Entry
由于需要实现将pod连接到群集范围的默认网络，因此即使未在“网络附件选择注释”中指定，该网络也必须在状态映射中具有条目，并且可能没有相应的NetworkAttachmentDefinition对象。 该条目应将其“默认”键设置为“true”。 默认条目可以位于状态列表中的任何位置。
## 5.3 Status Map Key Definitions
为每个网络附件的状态图定义了以下键。 保留所有不包含句点字符的键名称，以确保将来可以扩展此规范。 写入除本规范中定义的密钥之外的密钥的实现必须使用反向域名表示法（例如“org.foo.bar.key-name”）来命名非标准密钥。  
### 5.3.1 "name"
此必需键的值（类型字符串）应包含来自窗格的网络附件选择批注的NetworkAttachmentDefinition对象名称，或者包含群集范围的默认网络的名称。 “name”可以包含第4.1.1节中定义的命名空间引用。 [参考2018-02-01会议@ 22:30]   
### 5.3.2 "interface"
此可选键的值（类型字符串）应包含与网络附件对应的pod的网络命名空间中的网络接口名称。  
#### 5.3.2.1 CNI Delegating Plugin Requirements  
对于版本0.3.0或更高版本的CNI结果，“interface”键应来自CNI Result的“interfaces”属性的第一个元素，该属性具有有效的“沙盒”属性。

对于早于0.3.0的CNI结果版本，实现可以从为网络附件操作设置的CNI_IFNAME环境变量填充该字段，或将该字段留空。  
### 5.3.3 "ips"
此可选键的值（类型字符串数组）应包含由于附件操作而分配给pod的IPv4和/或IPv6地址的数组。  

#### 5.3.3.1 CNI Delegating Plugin Requirements
对于版本0.3.0或更高版本的CNI结果，如果满足以下任一条件，则“ips”密钥应来自CNI结果的“ips”属性; 否则它不应该包含在状态图中。   

1. “ips”键应取自CNI Result对象的“ips”列表的元素，其中“interface”索引引用CNI Result对象的“interfaces”列表中带有有效“sandbox”键的第一个元素。   
2. 如果CNI Result对象的“interfaces”列表中没有元素，或者CNI Result对象的“interfaces”列表中没有任何接口具有有效的“sandbox”属性，则应从第一个元素中获取“ips”键。 CNI结果对象的“ips”列表，它没有“interface”属性或“interface”属性小于零。   

这些要求的目的是确保状态图中包含的IP地址与分配给pod中附件接口的IP地址相同，而不是CNI插件有时报告的非沙箱接口的地址。    

对于早于0.3.0的版本的CNI结果，“ips”密钥应来自CNI结果的“ip4”和“ip6”属性的组合。  
### 5.3.4 "mac"
此可选键的值（类型字符串）应包含由“interface”键指定的网络接口的硬件地址。 如果存在“mac”键，则还必须存在“interface”。  
#### 5.3.4.1 CNI Delegating Plugin Requirements
对于版本0.3.0或更高版本的CNI结果，“mac”键应来自CNI Result的“interfaces”属性的第一个元素，该属性具有有效的“沙盒”属性。

对于早于0.3.0的CNI结果版本，实现可以通过特定于实现的机制填充该字段，或将该字段留空。  

### 5.3.5 "default"
此必需键的值（类型为boolean）应指示此附件是群集范围的默认网络的结果。 “网络附件状态注释”列表中只有一个元素可以将“默认”键设置为true。  

### 5.3.6 "dns"
此可选键的值（类型映射）应包含由于网络附件而收集的DNS信息。 地图可能包含以下键。  

#### 5.3.6.1 "nameservers"
此可选键的值（类型字符串数组）应包含DNS服务器的IPv4和/或IPv6地址数组。  
#### 5.3.6.2 "domain"  
此可选键的值（类型字符串）应包含网络附件的本地域名。   

#### 5.3.6.3 "search"
此可选键的值（类型字符串数组）应包含网络附件的DNS搜索名称数组。   
## 5.4 Example  
```
kind: Pod
metadata:
  name: my-pod
  namespace: my-namespace
  annotations:
    k8s.v1.cni.cncf.io/network-status: |
      [
        {
          "name": "cluster-wide-default",
          "interface": "eth5",
          "ips": [ "1.2.3.1/24", "2001:abba::2230/64" ],
          "mac": "02:11:22:33:44:54",
          "default": true
        },
        {
          "name": "some-network",
          "interface": "eth1",
          "ips": [ "1.2.3.4/24", "2001:abba::2234/64" ],
          "mac": "02:11:22:33:44:55",
          "dns": {
            "nameservers": [ "4.2.2.1", "2001:4860:4860::8888" ],
            "search": [ "eng.foobar.com", "foobar.com" ]
          },
          "default": false
        },
        {
          "name": "other-ns/an-ip-over-infiniband-network",
          "interface": "ib0",
          "ips": [ "5.4.3.2/16" ],
          "mac": "80:00:11:22:33:44:55:66:77:88:99:aa:bb:cc:dd:ee:ff:00:11:22",
          "default": false
        }
      ]

```
# 6. Cluster-Wide Default Network  
实现必须将每个pod连接到群集范围的默认网络，从而保持现有的Kubernetes pod网络行为。 [ref 2018-01-18 @ 31:30]实现可以自由定义如何配置和引用群集范围的默认网络（例如，作为NetworkAttachmentDefinition对象，磁盘上的CNI JSON配置文件，或其他一些 如果所有pod都首先连接到该网络，则意味着）。   
## 6.1 Network Plugin Readiness  
在群集范围的默认网络准备就绪之前，实现不应将其CNI JSON配置文件写入Kubernetes CNI配置目录。 这可以防止Kubernetes在节点上安排将立即失败的pod，因为该实现已表示准备就绪，但群集范围的默认网络没有。    

该实现可以自由定义如何确定群集范围的默认网络就绪性，前提是当确定群集范围的默认网络准备就绪时，可以立即将pod连接到该网络并具有合理的网络连接期望。  
### 6.1.1 CNI Delegating Plugin Recommendations
为了防止群集范围的默认网络，实现和kubelet的CNI插件加载器之间的竞争条件，建议通过--cni-conf-dir命令行选项为kubelet配置特定于实现的CNI配置目录。  
 实现应该等到群集范围的默认网络插件的CNI JSON配置写入/etc/cni/net.d，然后将其自己的CNI JSON配置文件写入给予kubelet的特定于实现的CNI配置目录。  
### 6.1.2 Alternate Readiness Method
如果在指示实现准备好kubelet之前无法等待群集范围的默认网络准备就绪，则实现可以立即将其自己的CNI配置文件安装到kubelet CNI配置目录，确保其配置优先于群集 全范围的默认网络（如果有的话）。 然后，实施必须阻止任何pod网络连接/分离操作，直到群集范围的默认网络准备就绪。  

## 6.2 Cluster-wide Default Network Attachment Ordering
在附加由“网络附件选择注释”指定的任何网络之前，实施必须附加群集范围的默认网络。  

# 7. Runtime and Implementation Considerations
## 7.1 CNI Delegating Plugin Requirements for CNI Configuration and Result Versioning
CNI委托插件必须符合CNI规范在CNI配置列表中调用插件的要求。 如果CNI委托插件使用CNI项目的引用“libcni”库，则会自动处理这些问题。 如果没有，CNI规范要求CNI委托插件将配置列表中的cniVersion和name字段注入每个插件调用的配置JSON。 这可确保configlist中的每个插件都能够理解上一个插件的结果，并确保运行时接收到正确版本的最终结果。   
## 7.2 Attachment/Detachment Failure Handling
在pod网络设置中，无法连接pod的网络附件选择注释所引用的任何网络，或者无法连接群集范围的默认网络，应立即使pod网络设置操作失败。 不应尝试尚未执行的附件。  
在pod网络拆卸时，必须拆除在pod网络设置期间尝试的所有网络附件，并且一个网络附件的故障不能防止后续附件的拆除。 但是，如果任何分离失败，则应将最终错误传递给运行时以指示整体拆卸操作失败。  
## 7.3 Serialization of Network Attachment Operations
CNI规范0.4.0规定“容器运行时不能为同一容器调用并行操作，但允许为不同容器调用并行操作。” 实现必须遵循此要求，并且不得并行化pod网络附件操作。 在本规范的未来版本中可能会取消此要求。  
## 7.4 Restrictions on Selection of Network Attachment Definitions by Pods
实现可以自由地对给定Pod可以选择哪些网络附件定义施加限制。 这可以通过RBAC，准入控制或任何其他特定于实现的方法来完成。 如果实现确定Pod不允许为Pod选择的给定网络附件定义，则它必须使pod网络操作失败。  
