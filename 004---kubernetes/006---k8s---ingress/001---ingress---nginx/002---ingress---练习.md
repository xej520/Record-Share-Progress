## 验证Ingress的反向代理服务？  
### 准备被测试用的服务  
1. 准备一个文件test-nginx.yaml  
```
---
apiVersion: v1
kind: Service
metadata:
  name: test-ingress
  namespace: default
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: test-ingress

---

apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: test-ingress
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: test-ingress
    spec:
      containers:
      - image: nginx:latest
        imagePullPolicy: IfNotPresent
        name: test-nginx
        ports:
        - containerPort: 80
---
```

2. 创建nginx服务  
```
kubectl create -f test-nginx.yaml 
```
### 测试  
#### 未使用ingress-nginx前的测试  
1. 查看pod、svc情况
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/3A2614279048431ABBF5647FA843F42C/20526)   
2. 测试  
![](https://note.youdao.com/yws/public/resource/d8631b2801d11e53d570068af1c0bf0f/xmlnote/DDD324E0851C4881B00BA7AC21EB8C5C/20529)  

#### 使用ingress-nginx  
1. 准备一个ingress服务的yaml 

2. 使其生效  


3. 测试  