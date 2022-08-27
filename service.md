
- [Service](#service)
  - [介绍](#介绍)
    - [Service是什么](#service是什么)
    - [Service存在的意义](#service存在的意义)
    - [pod与service的关系](#pod与service的关系)
  - [Service 常见有三种类型](#service-常见有三种类型)

# Service

## 介绍
### Service是什么  
* 将运行在一组 Pods 上的应用程序公开为网络服务的抽象方法。
例如，假定有一组 Pod，它们对外暴露了 9376 端口，同时还被打上 app=MyApp 标签：
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```
上述配置创建一个名称为 "my-service" 的 Service 对象，它会将请求代理到使用 TCP 端口 9376，并且具有标签 app.kubernetes.io/name=MyApp 的 Pod 上    
说明： 需要注意的是，Service 能够将一个接收 port 映射到任意的 targetPort。 默认情况下，targetPort 将被设置为与 port 字段相同的值。  
Pod 中的端口定义是有名字的，你可以在 Service 的 targetPort 属性中引用这些名称。 例如，我们可以通过以下方式将 Service 的 targetPort 绑定到 Pod 端口：
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:stable
    ports:
      - containerPort: 80
        name: http-web-svc

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 80
    targetPort: http-web-svc
```
### Service存在的意义
* 服务发现（防止pod失联）
* 负载均衡（定义一组pod访问策略）

### pod与service的关系
* 根据label和selector 标签建立关系的
* 其中service层的ip 是虚拟IP
```
# service
selector: 
  app: nginx

# pod
labels:
  app: nginx
```
## Service 常见有三种类型

* ClusterIP：集群内部访问（默认）
  * 默认类型，自动分配一个仅 cluster 内部可以访问的虚拟 IP
* NodePort:  对外访问应用使用
  * 在 ClusterIP 基础上为 Service 在每台机器上绑定一个端口，这样就可 以通过 来访问该服务 
* LoadBalance：对外访问应用，公有云
  * 在 NodePort 的基础上，借助 cloud provider 创建一个外部的负载 均衡器，并将请求转发到 <NodeIP>:NodePort    

需要说明的是，node 内网部署应用，外网一般不能访问，我们需要
1. 找到一台可以进行外网访问的机器，安装Nginx，反向代理
2. 手动把可以访问节点添加到Nginx




