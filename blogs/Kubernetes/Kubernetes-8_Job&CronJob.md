---
title: K8S-8_计划任务（Job&CronJob）
tags: 
  - Kubernetes
date: 2023-10-26 09:00:00
categories:	
  - Kubernetes
---

## Job

### 理论基础

Job是马上运行的一次性任务， job会创建一个或者多个 Pods，并确保指定数量的 Pods 成功终止。 随着 Pods 成功结束，Job 跟踪记录成功完成的 Pods 个数。 当数量达到指定的成功个数阈值时，任务（即 Job）结束。 删除 Job 的操作会清除所创建的全部 Pods。所以Job类型Pod的restartPolicy通常是Never或者OnFailure。

一种简单的使用场景下，你会创建一个 Job 对象以便以一种可靠的方式运行某 Pod 直到完成。 当第一个 Pod 失败或者被删除（比如因为节点硬件失效或者重启）时，Job 对象会启动一个新的 Pod。

### 命令行创建Job

```shell
[root@clientvm ~]# kubectl create job -h
Create a job with the specified name.
Usage:
  kubectl create job NAME --image=image [--from=cronjob/name] -- [COMMAND] [args...] [options]
  
Examples:
  # Create a job
  kubectl create job my-job --image=busybox

  # Create a job with command
  kubectl create job my-job --image=busybox -- date

  # Create a job from a CronJob named "a-cronjob"
  kubectl create job test-job --from=cronjob/a-cronjob
  
#examples
[root@clientvm ~]# kubectl create job my-job -n mytest --image=busybox -- echo hello
job.batch/my-job created

[root@clientvm ~]# kubectl get pod -n mytest
NAME           READY   STATUS              RESTARTS   AGE
my-job-4twr8   0/1     ContainerCreating   0          3s

[root@clientvm ~]# kubectl get pod -n mytest
NAME           READY   STATUS      RESTARTS   AGE
my-job-4twr8   0/1     Completed   0          5s

[root@clientvm ~]# kubectl logs my-job-4twr8 -n mytest
hello

[root@clientvm ~]# kubectl get jobs.batch -n mytest
NAME     COMPLETIONS   DURATION   AGE
my-job   1/1           4s         70s

##删除Pod会自动创建
[root@clientvm ~]# kubectl delete pod my-job-jp794 -n mytest
pod "my-job-jp794" deleted

[root@clientvm ~]#
[root@clientvm ~]# kubectl get pod -n mytest
NAME           READY   STATUS    RESTARTS   AGE
my-job-c72bq   1/1     Running   0          46s

```

### 使用yaml创建job

```shell
#生成yaml文件
[root@clientvm ~]# kubectl create job my-job -n mytest --image=busybox --dry-run=client -o yaml -- sleep 200
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: my-job
  namespace: mytest
spec:
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command:
        - sleep
        - "200"
        image: busybox
        name: my-job
        resources: {}
      restartPolicy: Never
status: {}

#创建job
[root@clientvm ~]# kubectl apply -f job.yaml
job.batch/my-job created

[root@clientvm ~]# kubectl get jobs.batch -n mytest
NAME     COMPLETIONS   DURATION   AGE
my-job   0/1           38s        38s

[root@clientvm ~]# kubectl get pod -n mytest
NAME           READY   STATUS    RESTARTS   AGE
my-job-n2z45   1/1     Running   0          49s

#设置并行与完成次数
#completions：设置完成次数
#parallelism： 设置并行
[root@clientvm ~]# kubectl explain job.spec
KIND:     Job
VERSION:  batch/v1
......
   completions  <integer>
     Specifies the desired number of successfully finished pods the job should
     be run with. 
   parallelism  <integer>
     Specifies the maximum desired number of pods the job should run at any
     given time. 
```

---

## CronJob

### 理论基础

Cron Job 创建基于时间调度的 [Jobs](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/)。

一个 CronJob 对象就像 *crontab* (cron table) 文件中的一行。 它用 [Cron](https://en.wikipedia.org/wiki/Cron) 格式进行编写， 并周期性地在给定的调度时间执行 Job。

CronJobs 对于创建周期性的、反复重复的任务很有用，例如执行数据备份或者发送邮件。 CronJobs 也可以用来计划在指定时间来执行的独立任务，例如计划当集群看起来很空闲时 执行某个 Job。

### 命令行创建CronJob

```shell
[root@clientvm ~]# kubectl create cronjob -h
Create a cronjob with the specified name.
Usage:
  kubectl create cronjob NAME --image=image --schedule='0/5 * * * ?' -- [COMMAND] [args...] [flags] [options]
  
Aliases:
cronjob, cj

Examples:
  # Create a cronjob
  kubectl create cronjob my-job --image=busybox --schedule="*/1 * * * *"

  # Create a cronjob with command
  kubectl create cronjob my-job --image=busybox --schedule="*/1 * * * *" -- date

#examples
[root@clientvm ~]# kubectl create cronjob my-job -n mytest --image=busybox --schedule="*/1 * * * *" -- date
cronjob.batch/my-job created

[root@clientvm ~]# kubectl get cronjobs.batch -n mytest
NAME     SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
my-job   */1 * * * *   False     0        <none>          12s
[root@clientvm ~]# kubectl get cronjobs.batch -n mytest
NAME     SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
my-job   */1 * * * *   False     1        10s             63s
[root@clientvm ~]#
[root@clientvm ~]#
[root@clientvm ~]# kubectl get pod -n mytest
NAME                      READY   STATUS              RESTARTS   AGE
my-job-1607481720-5dvr2   0/1     ContainerCreating   0          17s

[root@clientvm ~]# kubectl get pod -n mytest
NAME                      READY   STATUS      RESTARTS   AGE
my-job-1607481720-5dvr2   0/1     Completed   0          35s

[root@clientvm ~]# kubectl get pod -n mytest
NAME                      READY   STATUS      RESTARTS   AGE
my-job-1607481720-5dvr2   0/1     Completed   0          80s
my-job-1607481780-sszc5   0/1     Completed   0          20s

[root@clientvm ~]# kubectl logs my-job-1607481720-5dvr2 -n mytest
Wed Dec  9 02:42:26 UTC 2020

[root@clientvm ~]# kubectl describe cronjobs.batch -n mytest  my-job
Name:                          my-job
Namespace:                     mytest
Labels:                        <none>
Annotations:                   <none>
Schedule:                      */1 * * * *
......
Events:
  Type    Reason            Age    From                Message
  ----    ------            ----   ----                -------
  Normal  SuccessfulCreate  2m12s  cronjob-controller  Created job my-job-1607481720
  Normal  SawCompletedJob   111s   cronjob-controller  Saw completed job: my-job-1607481720, status: Complete
  Normal  SuccessfulCreate  71s    cronjob-controller  Created job my-job-1607481780
  Normal  SawCompletedJob   51s    cronjob-controller  Saw completed job: my-job-1607481780, status: Complete
  Normal  SuccessfulCreate  11s    cronjob-controller  Created job my-job-1607481840
  
## 删除
[root@clientvm ~]# kubectl delete cronjobs.batch -n mytest  my-job
cronjob.batch "my-job" deleted

```

### 使用yaml创建CronJob

```shell
[root@clientvm ~]# kubectl create cronjob my-job -n mytest --image=busybox --dry-run=client -o yaml --schedule="*/1 * * * *"  -- date
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  creationTimestamp: null
  name: my-job
  namespace: mytest
spec:
  jobTemplate:
    metadata:
      creationTimestamp: null
      name: my-job
    spec:
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - command:
            - date
            image: busybox
            name: my-job
            resources: {}
          restartPolicy: OnFailure
  schedule: '*/1 * * * *'
status: {}

#创建CronJob
[root@clientvm ~]# kubectl apply -f cronjob.yaml
cronjob.batch/my-job created

[root@clientvm ~]# kubectl get cronjobs.batch -n mytest
NAME     SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
my-job   */1 * * * *   False     0        <none>          7s
```

