
# 如何测试kubernetes的dns服务是否可以使用呢？  
1. 新建一个yaml，pod-for-dns.yaml 
    ```
    apiVersion: v1
    kind: Pod
    metadata:
    name: bb
    namespace: default
    spec:
    containers:
    - image: busybox
      command:
        - sleep
        - "3600"
      imagePullPolicy: IfNotPresent
      name: bb
    restartPolicy: Always

    ```
2. 创建pod 
    ```
    kubectl  apply -f pod-for-dns.yaml
    ```
3. 查看pod的状态 
    ```
    kubectl get pods bb
    ```
4. 测试dns服务  
    ```
    kubectl exec -it bb -- nslookup kubernetes.default
    ```