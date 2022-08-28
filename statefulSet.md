- [StatefulSet](#statefulset)
  - [无状态和有状态](#无状态和有状态)
    - [无状态](#无状态)
    - [有状态](#有状态)
  - [限制](#限制)
  - [部署有状态应用](#部署有状态应用)
    - [无头service（无头服务）](#无头service无头服务)
    - [StatefulSet部署有状态服务](#statefulset部署有状态服务)
    - [statefulState 与 deployment区别](#statefulstate-与-deployment区别)

# StatefulSet

## 无状态和有状态
### 无状态
* 认为Pod 都一样
* 没有顺序要求
* 不用考虑在那个node 运行
* 随意进行伸缩和扩展

### 有状态
* 有状态服务的因素都要考虑到
* 稳定的、持久的存储
* 有序的、优雅的部署和扩缩
* 有序的、自动的滚动更新

## 限制
* 给定 Pod 的存储必须由 PersistentVolume Provisioner 基于所请求的 storage class 来制备，或者由管理员预先制备。
* 删除或者扩缩 StatefulSet 并不会删除它关联的存储卷。 这样做是为了保证数据安全，它通常比自动清除 StatefulSet 所有相关的资源更有价值。
* StatefulSet 当前需要无头服务来负责 Pod 的网络标识。你需要负责创建此服务。
* 当删除一个 StatefulSet 时，该 StatefulSet 不提供任何终止 Pod 的保证。 为了实现 StatefulSet 中的 Pod 可以有序且体面地终止，可以在删除之前将 StatefulSet 缩容到 0。
* 在默认 Pod 管理策略(OrderedReady) 时使用滚动更新， 可能进入需要人工干预才能修复的损坏状态。
  
## 部署有状态应用
### 无头service（无头服务）
* ClusterIP: none
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```
### StatefulSet部署有状态服务
1. 创建无头服务
2. 通过 StatefulSet 控制器拉起pod 	
以一个简单的 nginx 服务 web.yaml 为例: 
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # 必须匹配
  serviceName: "nginx"
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx # 必须匹配
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
```
* 验证
```
[root@master ~]# kubectl apply -f statefulset.yaml
service/nginx created
statefulset.apps/web created

# 查看svc, 发现无头服务nginx 创建
[root@master ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   13m
nginx        ClusterIP   None         <none>        80/TCP    71s
[root@master ~]# kubectl get pod
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          2m37s
web-1   1/1     Running   0          2m20s
```
### statefulState 与 deployment区别
* 有身份的、唯一标识的
* 每个pod有唯一主机名
* 唯一域名
  * StatefulSet 中每个 Pod 的 DNS 格式为 
    * statefulState名称-{0...N}.service名称.名称空间.svc.cluster.local
    * {0...N}为 Pod 所在的序号，从 0 开始到 N-1 
