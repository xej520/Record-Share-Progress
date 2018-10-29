
# 一、管理etcd集群  
1. 查看集群版本  
    ```
    etcdctl  --version
    etcdctl  --help
    ```

2. 查看集群健康状态  
    ```
    etcdctl cluster-health
    ```
3. 查看集群成员  
    ```
    etcdctl  member  list
    ```  
    ![](https://note.youdao.com/yws/public/resource/ca7c2468223e3c4a80c4e24b70ff9608/xmlnote/F756A0F11E254BB8BD68BE53DF66735C/21619)  


4. 更新节点  
    如果你想更新一个节点的IP(peerURLS)，首先你需要知道那个节点的ID  
    ```
    etcdctl member list
    etcdctl member update memberID http://ip:2380
    ```

5. 删除一个节点(ETCD集群成员的缩减)   
    ```  
    etcdctl  member  list
    etcdctl  member  remove  memberID
    etcdctl  member  list
    //登陆要删除的节点，停止etcd服务   
    systemctl stop etcd

    ```

6. 增加一个新的节点(ETCD集群成员的扩展)   
    注意:步骤很重要，不然会报集群ID不匹配
    - a. 将目标节点添加到集群  
        etcdctl member add slave1 http://172.16.91.176:2380
    - b. 查看新增成员列表，slave1状态为unstarted  
        etcdctl member list
    - c. 清空目标节点slave1的data-dir 
    节点删除后，集群中的成员信息会更新，新节点是作为一个全新的节点加入集群，  
    如果data-dir有数据，etcd启动时会读取己经存在的数据，仍然用老的memberID会造成无法加入集群，  
    所以一定要清空新节点的data-dir。   
    rm  -rf  /path/to/etcd/data
    - d. 修改新增节点的etcd.service文件里，     –initial-cluster-state改为existing
    - e. 重新启动etcd服务 

# 二、CRUD 操作  
1. 在etcd1上设置一个key/value对，这时就可以在集群任意节点上获取key/value  
```
etcd1# etcdctl set api_server  http://192.168.5.44:8080

http://192.168.5.44:8080


etcd0# etcdctl get api_server

http://192.168.5.44:8080

```
2. crud 
```
# etcdctl  set  foo  "bar"
# etcdctl  get  foo

# etcdctl  mkdir  hello
# etcdctl  ls

# etcdctl  --output  extended  get  foo
# etcdctl  --output  json  get  foo

# etcdctl  update  foo  "etcd cluster is ok"
# etcdctl  get foo

# etcdctl  import  --snap  /data/etcd/member/snap/db
```
3. REST API  
>curl  http://10.1.2.61:2379/v2/members   

查看集群成员，其中，id是集群成员的全局唯一的身份标识，name是成员的名字，peerURLs是成员之间通信的入口，clientURLs是成员跟用户通信的访问入口
```
# curl   http://10.1.2.61:2379/v2/keys

# curl   -fs  -X  PUT   http://10.1.2.61:2379/v2/keys/_test

# curl   -X  GET   http://10.1.2.61:2379/v2/keys/_test
```  