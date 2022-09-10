- [Secret](#secret)
  - [介绍](#介绍)
    - [注意](#注意)
  - [应用场景](#应用场景)
  - [使用](#使用)
    - [创建secret加密数据](#创建secret加密数据)
    - [以变量的形式挂载到pod容器中](#以变量的形式挂载到pod容器中)
    - [以volume的形式挂载到pod容器中](#以volume的形式挂载到pod容器中)
# Secret

## 介绍
* Secret 是一种包含少量敏感信息例如密码、令牌或密钥的对象。 这样的信息可能会被放在 Pod 规约中或者镜像中。 使用 Secret 意味着你不需要在应用程序代码中包含机密数据。
* 就是将加密数据保存在etc 中，让 pod 以挂载 Volume 的方式进行访问
### 注意
* 默认情况下，Kubernetes Secret 未加密地存储在 API 服务器的底层数据存储（etcd）中。 任何拥有 API 访问权限的人都可以检索或修改 Secret，任何有权访问 etcd 的人也可以。 此外，任何有权限在命名空间中创建 Pod 的人都可以使用该访问权限读取该命名空间中的任何 Secret； 这包括间接访问，例如创建 Deployment 的能力。
  
## 应用场景
* 作为挂载到一个或多个容器上的卷 中的文件。
* 作为容器的环境变量。
* 由 kubelet 在为 Pod 拉取镜像时使用。

## 使用
### 创建secret加密数据
* 创建 secret.yaml 
```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: MWYyZDFlMmU2N2Rm
  username: YWRtaW4=
```
* 拉起secret
```
[root@master ~]# kubectl create -f secret.yaml 
secret/mysecret created
[root@master ~]# kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-8lf22   kubernetes.io/service-account-token   3      10d
mysecret              Opaque                                2      22s
[root@master ~]# 
```
### 以变量的形式挂载到pod容器中
* 创建secret-pod.yaml文件
```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-nginx
    image: nginx
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
            optional: false # 此值为默认值；意味着 "mysecret"
                            # 必须存在且包含名为 "username" 的主键
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
            optional: false # 此值为默认值；意味着 "mysecret"
                            # 必须存在且包含名为 "password" 的主键
  restartPolicy: Never
```
* 拉起 pod
```
[root@master ~]# vim secret-pod.yaml
[root@master ~]# kubectl apply -f secret-pod.yaml 
pod/my-pod created
[root@master ~]# kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
ds-test-7wcll            1/1     Running   0          10d
my-pod                   1/1     Running   0          23s
nginx-6799fc88d8-xpv4q   1/1     Running   0          10d
[root@master ~]# 
```
* 进入容器，查看环境变量值
```
[root@master ~]# kubectl exec -it my-pod bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@my-pod:/# echo $SECRET_USERNAME
admin
root@my-pod:/# echo $SECRET_PASSWORD
1f2d1e2e67df
root@my-pod:/# 
```
### 以volume的形式挂载到pod容器中
* 创建secret-vol.yam 文件
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod-secret-volumes
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```
* 拉起pod
```
[root@master ~]# vim secret-vol.yaml
[root@master ~]# kubectl apply -f secret-vol.yaml 
pod/mypod-secret-volumes created
[root@master ~]# kubectl get pod
NAME                     READY   STATUS              RESTARTS   AGE
ds-test-7wcll            1/1     Running             0          10d
my-pod                   1/1     Running             0          12h
mypod-secret-volumes     0/1     ContainerCreating   0          13s
nginx-6799fc88d8-xpv4q   1/1     Running             0          10d
[root@master ~]# kubectl exec mypod-secret-volumes  -it bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@mypod-secret-volumes:/# cd /etc/fo
fonts/ foo/   
root@mypod-secret-volumes:/# cd /etc/foo/
root@mypod-secret-volumes:/etc/foo# ls     
password  username
root@mypod-secret-volumes:/etc/foo# cat username
adminroot@mypod-secret-volumes:/etc/foo# 
```