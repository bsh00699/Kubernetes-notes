- [DaemonSet](#daemonset)
  - [介绍](#介绍)
  - [使用场景](#使用场景)
  - [创建 DaemonSet](#创建-daemonset)

# DaemonSet

## 介绍
* DaemonSet 确保全部（或者某些）节点上运行一个 Pod 的副本。 当有节点加入集群时， 也会为他们新增一个 Pod 。 当有节点从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。
  
## 使用场景
* 在每个节点上运行集群守护进程
* 在每个节点上运行日志收集守护进程
* 在每个节点上运行监控守护进程
  
## 创建 DaemonSet
下面是一个简单示例
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: nginx
  name: ds-test
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: IfNotPresent
        name: logs
        ports:
        - containerPort: 80
        volumeMounts:
        - name: varlog
          mountPath: /tmp/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```
```
kubectl apply -f ds-test.yaml

# 查看pod 
[root@master ~]# kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
ds-test-7wcll            1/1     Running   0          22s
# 进入pod 查看 /tmp/log 下日志
kubectl exec -it ds-test-7wcll -- /bin/bash

[root@master ~]# kubectl exec -it ds-test-7wcll bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@ds-test-7wcll:/# ls /tmp/log
anaconda           chrony                 dnf.librepo.log               hawkey.log           maillog-20220731   private          spooler-20220725
audit              cloud-init-output.log  dnf.librepo.log-20211111      hawkey.log-20220725  maillog-20220821   sa               spooler-20220731
boot.log           cloud-init.log         dnf.librepo.log-20220725      hawkey.log-20220731  maillog-20220828   samba            spooler-20220821
boot.log-20211111  containers             dnf.librepo.log-20220731      hawkey.log-20220821  messages           secure           spooler-20220828
boot.log-20220725  cron                   dnf.librepo.log-20220821      hawkey.log-20220828  messages-20220725  secure-20220725  sssd
boot.log-20220821  cron-20220725          dnf.log                       journal              messages-20220731  secure-20220731  tuned
boot.log-20220822  cron-20220731          dnf.log.1                     lastlog              messages-20220821  secure-20220821  wtmp
btmp               cron-20220821          dnf.rpm.log                   maillog              messages-20220828  secure-20220828
btmp-20220801      cron-20220828          ecs_network_optimization.log  maillog-20220725     pods               spooler
root@ds-test-7wcll:/# 
```
