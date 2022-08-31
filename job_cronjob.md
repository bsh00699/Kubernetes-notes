
# Job/CronJob
## Job

### 介绍
* 负责批量处理短暂的一次性任务 (short lived one-off tasks)，即仅执行一次的任务， 它保证批处理任务的一个或多个 Pod 成功结束 
### job类型

* 非并行 Job：
  * 通常只启动一个 Pod，除非该 Pod 失败。
  * 当 Pod 成功终止时，立即视 Job 为完成状态。
* 具有确定完成计数的并行 Job：
  * .spec.completions 字段设置为非 0 的正数值。
  * Job 用来代表整个任务，当成功的 Pod 个数达到 .spec.completions 时，Job 被视为完成。
  * 当使用 .spec.completionMode="Indexed" 时，每个 Pod 都会获得一个不同的 索引值，介于 0 和 .spec.completions-1 之间。
* 带工作队列的并行 Job：
  * 不设置 spec.completions，默认值为 .spec.parallelism。
  * 多个 Pod 之间必须相互协调，或者借助外部服务确定每个 Pod 要处理哪个工作条目。 例如，任一 Pod 都可以从工作队列中取走最多 N 个工作条目。
  * 每个 Pod 都可以独立确定是否其它 Pod 都已完成，进而确定 Job 是否完成。
  * 当 Job 中任何 Pod 成功终止，不再创建新 Pod。
  * 一旦至少 1 个 Pod 成功完成，并且所有 Pod 都已终止，即可宣告 Job 成功完成。
  * 一旦任何 Pod 成功退出，任何其它 Pod 都不应再对此任务执行任何操作或生成任何输出。 所有 Pod 都应启动退出过程

### 模板示例
* 下面是一个 Job 配置示例。它负责计算 π 到小数点后 2000 位，并将结果打印出来。 此计算大约需要 10 秒钟完成。
```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```
```
kubectl apply -f job.yaml
```

## CronJob