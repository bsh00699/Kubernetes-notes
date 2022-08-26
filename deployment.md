- [Deployment](#deployment)
  - [介绍](#介绍)
  - [pod与controller的关系](#pod与controller的关系)
  - [应用场景](#应用场景)
  - [使用Deployment 部署应用](#使用deployment-部署应用)
    - [创建deployment](#创建deployment)
      - [命令](#命令)
      - [手动编写yaml文件](#手动编写yaml文件)
    - [应用升级](#应用升级)
      - [命令](#命令-1)
      - [手动更改应用的镜像版本](#手动更改应用的镜像版本)
    - [应用回滚](#应用回滚)
      - [查看升级版本历史](#查看升级版本历史)
      - [回滚到上一个版本](#回滚到上一个版本)
      - [回滚到指定的版本](#回滚到指定的版本)
    - [弹性伸缩](#弹性伸缩)

# Deployment
## 介绍
* Deployment为Pod和Replica Set（升级版的 Replication Controller）提供声明式更新
## pod与controller的关系
* pod 是通过controller实现应用的运维，比如伸缩、滚动升级
* pod 与 controller 之间通过 selector label 标签简历关系

## 应用场景
* 定义Deployment 来创建Pod和ReplicaSet
* 滚动升级和回滚应用
* 扩容和缩容
* 暂停和继续Deployment 

## 使用Deployment 部署应用
### 创建deployment
#### 命令
* 导出yaml文件
```
kubectl create deployment web --image=nginx --dry-run -o yaml > web.yaml
```
* 使用 yam 部署应用
```
kubectl apply -f web.yaml
```
* 对外发布服务(生成yaml文件直接拉起)
```
kubectl expose deployment web --port=80 --type=NodePort --target-port=80 --name=web1 -o yaml > web1.yaml
# --port=80 Nginx 端口
# --type=NodePort NodePort形式暴露服务
# --target-port=80 对外端口
```
* 拉起服务
```
kubectl apply -f web1.yaml
```
* 查看服务以及端口
```
[root@master ~]# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE     SELECTOR
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        4d14h   <none>
nginx        NodePort    10.110.185.165   <none>        80:30399/TCP   4d5h    app=nginx
web1         NodePort    10.100.138.228   <none>        80:30184/TCP   5m15s   app=web
```
注意web1，对外端口是30184,浏览器直接访问 [部署该pod节点的公IP]:30184即可,结果如下
```
Welcome to nginx!
If you see this page, the nginx web server is successfully installed and working. Further configuration is required.

For online documentation and support please refer to nginx.org.
Commercial support is available at nginx.com.

Thank you for using nginx.
```
#### 手动编写yaml文件
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
然后拉起服务即可
```
kubectl apply -f [编写的yaml文件]
```
### 应用升级
#### 命令
* 先来更新 nginx Pod 以使用 nginx:1.16.1 镜像，而不是 nginx:1.14.2 镜像
```
kubectl set image deployment [deployment name] nginx=nginx:1.16.1
```
* 查看升级是否成功
```
kubectl rollout status deployment [deployment name]
```
#### 手动更改应用的镜像版本
* 手动编写yaml文件，更改应用的镜像版本
```
kubectl edit deployment [deployment name]
```
### 应用回滚
#### 查看升级版本历史
```
kubectl rollout history deployment [deployment name]
```
#### 回滚到上一个版本
* 新版本上线后，不是很稳定，需要回滚到上一个版本
```
kubectl rollout undo deployment [deployment name]
```
#### 回滚到指定的版本
```
kubectl rollout undo deployment [deployment name] --to-version=[版本号]
```
### 弹性伸缩
```
kubectl scale deployment [deployment name]  --replicas=2
```
假设集群启用了Pod 的水平自动缩放， 你可以为 Deployment 设置自动缩放器，并基于现有 Pod 的 CPU 利用率选择要运行的 Pod 个数下限和上限。
```
kubectl autoscale deployment [deployment name] --min=10 --max=15 --cpu-percent=80
```