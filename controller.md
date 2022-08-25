- [Controller](#controller)
  - [什么是控制器](#什么是控制器)
  - [控制器类型](#控制器类型)
  - [Replication Controller](#replication-controller)
    - [工作原理](#工作原理)
    - [RC 替代方法](#rc-替代方法)
      - [ReplicaSet](#replicaset)
      - [Deployment（推荐）](#deployment推荐)
  - [ReplicaSet](#replicaset-1)
  - [Deployment](#deployment)
  - [DaemonSet](#daemonset)
  - [Job](#job)
  - [CronJob](#cronjob)

# Controller
## 什么是控制器
* k8s 中内建了很多 controller（控制器），这相当于一个状态机，用来控制pod 的具体状态和行为
## 控制器类型
* Replication Controller
* ReplicaSet
* DaemonSet
* StateFulSet
* Job
* CronJob
## Replication Controller
* ReplicationController（简称RC）是确保用户定义的Pod副本数保持不变
### 工作原理
* 在用户定义范围内，如果pod增多，则ReplicationController会终止额外的pod，如果减少，RC会创建新的pod，始终保持在定义范围。例如，RC会在Pod维护（例如内核升级）后在节点上重新创建新Pod。
* ReplicationController会替换由于某些原因而被删除或终止的pod，例如在节点故障或中断节点维护（例如内核升级）的情况下。因此，即使应用只需要一个pod，我们也建议使用ReplicationController。
### RC 替代方法
#### ReplicaSet
* ReplicaSet是支持新的set-based选择器要求的下一代ReplicationController 。它主要用作Deployment协调pod创建、删除和更新。
#### Deployment（推荐）
* Deployment是一个高级的API对象，以类似的方式更新其底层的副本集和它们的Pods kubectl rolling-update。如果您希望使用这种滚动更新功能，建议您进行部署，因为kubectl rolling-update它们是声明式的，服务器端的，并具有其他功能

## ReplicaSet
* ReplicaSet 的目的是维护一组在任何时候都处于运行状态的 Pod 副本的稳定集合。 因此，它通常用来保证给定数量的、完全相同的 Pod 的可用性
* ReplicaSet（RS）是Replication Controller（RC）的升级版本。ReplicaSet 和  Replication Controller之间的唯一区别是对选择器的支持。ReplicaSet支持labels user guide中描述的set-based选择器要求， 而Replication Controller仅支持equality-based的选择器要求
* ReplicaSet 确保任何时间都有指定数量的 Pod 副本在运行。 然而，Deployment 是一个更高级的概念，它管理 ReplicaSet，并向 Pod 提供声明式的更新以及许多其他有用的功能。 因此，我们建议使用 Deployment 而不是直接使用 ReplicaSet， 除非你需要自定义更新业务流程或根本不需要更新。

## Deployment
* Deployment为Pod和Replica Set（升级版的 Replication Controller）提供声明式更新
* 典型的应用场景如下
  * 定义Deployment 来创建Pod和ReplicaSet
  * 滚动升级和回滚应用
  * 扩容和缩容
  * 暂停和继续Deployment 
## DaemonSet
DaemonSet 确保全部（或者某些）节点上运行一个 Pod 的副本。 当有节点加入集群时， 也会为他们新增一个 Pod 。 当有节点从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

DaemonSet 的一些典型用法：
* 在每个节点上运行集群守护进程
* 在每个节点上运行日志收集守护进程
* 在每个节点上运行监控守护进程

## Job
* 负责批处理任务，即仅执行一次的任务，它保证批处理任务的一个或者多个pod成功结束
## CronJob
* 用于执行周期性的动作，例如备份、报告生成等。 这些任务中的每一个都应该配置为周期性重复的（例如：每天/每周/每月一次）； 你可以定义任务开始执行的时间间隔。