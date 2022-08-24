- [Pod](#pod)
  - [pod基本概念](#pod基本概念)
  - [pod存在意义](#pod存在意义)
  - [pod实现机制](#pod实现机制)
  - [pod中镜像拉取策略](#pod中镜像拉取策略)
  - [pod资源限制](#pod资源限制)
  - [pod重启策略（restartPolicy）](#pod重启策略restartpolicy)
  - [pod健康检查](#pod健康检查)
  - [影响pod调度的因素](#影响pod调度的因素)
    - [资源限制](#资源限制)
    - [节点选择器标签](#节点选择器标签)
    - [节点亲和性](#节点亲和性)
      - [软亲和性](#软亲和性)
      - [硬亲和性](#硬亲和性)
      - [示例](#示例)
      - [反亲和性](#反亲和性)
    - [污点Taints（节点属性）](#污点taints节点属性)
      - [污点介绍](#污点介绍)
      - [污点设置、查看、去除](#污点设置查看去除)
      - [污点使用场景](#污点使用场景)
    - [污点容忍Tolerations](#污点容忍tolerations)
      - [介绍](#介绍)
  - [重启pod的几种方式](#重启pod的几种方式)
# Pod
## pod基本概念
* 最小部署单元
* 包含多个容器（一组容器的集合）
* 一个pod 中的容器共享网络命名空间
* pod 是短暂的

## pod存在意义
* 创建容器使用docker，一个docker对应一个容器，一个容器就是一个进程，也就是说一个容器只能运行一个应用程序
* pod 是多进程设计，运行多个应用程序，一个pod有多个容器，一个容器里面运行一个应用程序
* pod 为了亲密性应用   
  * 两个应用之间需要进行交互  
  * 网络之间的调用  
  * 两个应用需要频繁调用  
## pod实现机制
* 共享网络 
  * 通过pause 容器，把其他业务容器加到pause 容器里，让所有业务容器在同一个名称空间里，实现网络共享
* 共享存储
  * 持久化数据
    * 日志数据
    * 业务数据
  * 引入数据卷volume，使用数据卷，实现持久化存储
## pod中镜像拉取策略
* imagePullPolicy有三个值如下所示
  * IfNotPresent  默认值，镜像在宿主机上不存在时才拉取
  * Always 每次创建 pod 时，都会重新拉取一次镜像
  * Never pod 永远不会主动拉取这个镜像
## pod资源限制
* request 代表调度节点的资源
* limits 代表调度节点的资源的最大限制
```
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      request:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```
## pod重启策略（restartPolicy）
* Always: 当容器中止退出后，总是重启容器（默认策略）
* OnFailure: 当容器异常退出（退出状态码非0）时，才重启容器
* Never: 当容器中止退出后,从不重启容器

## pod健康检查
* 自我修复，如果健康检查过不去，pod就会重启，直到 status 为 runing
  * livenessProbe（存活检查）
    * 如果检查失败，将杀死容器，根据pod的restartPolicy 来操作
  * readinessProbe(就绪检查)
    * 如果检查失败，k8s 就会把 pod 从service end points 中剔除
* Probe 支持三种检查的方法
  1. httpGet 发送http 请求，返回 200-400 范围的状态码为成功
  2. exec 执行shell 命令 返回状态码是0 为成功
  3. tcpSocket 发起 tcp socket 建立成功

## 影响pod调度的因素
### 资源限制
* pod资源限制，根据request 找到足够的node节点，进行调度
```
resources:
  request:
    memory: "64Mi"
    cpu: "250m"
```
### 节点选择器标签
```
# 给节点打标签
kubectl label node node1 env_role=dev

# 在控制器上添加nodeSelector
spec:
  nodeSelector:
    env_role: dev
```
### 节点亲和性
* 节点亲和性分为软亲和性以及硬亲和性
  * preferredDuringSchedulingIgnoredDuringExecution：软策略 可以不再最好在
  * requiredDuringSchedulingIgnoredDuringExecution：硬策略 必须在 
#### 软亲和性
* preferredDuringSchedulingIgnoredDuringExecution：表示优先部署到满足条件的节点上，如果没有满足条件的节点，就忽略这些条件，按照正常逻辑部署
#### 硬亲和性
* requiredDuringSchedulingIgnoredDuringExecution：表示pod必须部署到满足条件的节点上，如果没有满足条件的节点，就不停重试
#### 示例
```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: gcr.io/google_containers/pause:2.0
```
* podAntiAffinity亲和性策略表示尽量把有label app:nginx的pod部署在同一节点
* matchExpressions 表示符合label条件的pod  
* 键值运算符operator关系
  * In：label 的值在某个列表中
  * NotIn：label 的值不在某个列表中
  * Gt：label 的值大于某个值
  * Lt：label 的值小于某个值
  * Exists：某个 label 存在
  * DoesNotExist：某个 label 不存在
#### 反亲和性
* pod的非亲和性表示pod不部署到满足某些label的pod所在的node上
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: nginx-server
        image: nginx:latest
```
其中 podAntiAffinity 表示pod的非亲和性策略
### 污点Taints（节点属性）
#### 污点介绍
使用 kubectl taint 命令可以给某个 Node 节点设置污点，Node 被设置上污点之后就和 Pod 之间存在了一种互斥的关系，可以让 Node 拒绝 Pod 的调度执行，甚至将 Node 已经存在的 Pod 驱逐出去。每个污点的组成如下
```
key=value:effect
```
每个污点有一个 key 和 value 作为污点的标签，其中 value 可以为空，eﬀect 描述污点的作用。当前effect字段支持如下三个选项：
1. NoSchedule ：表示 k8s 将不会将 Pod 调度到具有该污点的 Node 上
2. PreferNoSchedule ：表示 k8s 将尽量避免将 Pod 调度到具有该污点的 Node 上
3. NoExecute ：表示 k8s 将不会将 Pod 调度到具有该污点的 Node 上，同时会将 Node 上已经存在的 Pod 驱逐出去  
示例
```
apiVersion: v1
kind: Pod
metadata:
	name: affinity
	labels:
		app: node-affinity-pod
spec:
	containers:
	- name: with-node-affinity
		image: hub.atguigu.com/library/myapp:v1
		tolerations: #容忍污点
		- key: “key1” #污点key
	operator: “Equal”
		value: “value1”
		effect: “NoSchedule”
		tolerationSeconds: 3600 #容忍时间
```
#### 污点设置、查看、去除
* 设置污点
```
kubectl taint nodes node1 key1=value1:NoSchedule
```
* 查看污点
```
kubectl describe node [node-name] | grep Taints
```
* 去除污点
```
kubectl taint nodes node1 key1:NoSchedule-
```
#### 污点使用场景
* 专用节点
* 配置专用硬件节点
* 基于Taint驱除

### 污点容忍Tolerations
#### 介绍
设置了污点的 Node 将根据 taint 的 eﬀect：NoSchedule、PreferNoSchedule、NoExecute 和 Pod 之间产生互斥的关系，Pod 将在一定程度上不会被调度到 Node 上。 但我们可以在 Pod 上设置容忍 ( Toleration ) ，意思是设置了容忍的 Pod 将可以容忍污点的存在，可以被调度到存在污点的 Node 上  
示例如下
```
spec:
  tolerations:
  - key: "key1"
    operator: "Equal"
	  value: "value1"
	  effect: "NoSchedule"
tolerationSeconds: 3600
```
其中
* key, vaule, effect 要与 Node 上设置的 Taints的设置值 保持一致。
* operator 的值为 Exists 将会忽略 value 值
* tolerationSeconds 用于描述当 Pod 需要被驱逐时可以在 Pod 上继续保留运行的时间，类似限期驱离   

当不指定 key 值时，表示容忍所有的污点 key：
```
tolerations:
- operator: "Exists"
```
当不指定 eﬀect 值时，表示容忍所有的污点作用
```
tolerations:
- key: "key"
  operator: "Exists"
```
有多个 Master 存在时，防止资源浪费，可以如下设置，让master也启动pod
```
kubectl taint nodes Node-Name node-role.kubernetes.io/master=:PreferNoSchedule
```
## 重启pod的几种方式
* 控制副本数，从 0 到 1
```
kubectl scale deployment XXXX --replicas=0 -n {namespace} 
kubectl scale deployment XXXX --replicas=1 -n {namespace} 
```
* 删除原有的pod, 让 控制器自动拉起
```
kubectl delete pod {podname} -n {namespace} 
```
* 生成yaml 文件，在重新拉起pod
```
kubectl get pod {podname} -n {namespace} -o yaml | kubectl replace --force -f
```