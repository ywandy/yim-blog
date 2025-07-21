---
title: K8S-6_声明式升级应用（DeploymentSet）
tags: 
  - Kubernetes
date: 2023-10-26 09:00:00
categories:	
  - Kubernetes
---

## 基础知识

一个 *Deployment* 控制器为 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 和 [ReplicaSets](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicaset/) 提供声明式的更新能力。

你负责描述 Deployment 中的 *目标状态*，而 Deployment 控制器以受控速率更改实际状态， 使其变为期望状态。你可以定义 Deployment 以创建新的 ReplicaSet，或删除现有 Deployment， 并通过新的 Deployment 收养其资源。

注意：不要管理 Deployment 所拥有的 ReplicaSet 。

以下是 Deployments 的典型用例：

- [创建 Deployment 以将 ReplicaSet 上线](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/#creating-a-deployment)。 ReplicaSet 在后台创建 Pods。 检查 ReplicaSet 的上线状态，查看其是否成功。
- 通过更新 Deployment 的 PodTemplateSpec，[声明 Pod 的新状态](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/#updating-a-deployment) 。 新的 ReplicaSet 会被创建，Deployment 以受控速率将 Pod 从旧 ReplicaSet 迁移到新 ReplicaSet。 每个新的 ReplicaSet 都会更新 Deployment 的修订版本。
- 如果 Deployment 的当前状态不稳定，[回滚到较早的 Deployment 版本](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment)。 每次回滚都会更新 Deployment 的修订版本。
- [扩大 Deployment 规模以承担更多负载](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/#scaling-a-deployment)。
- [暂停 Deployment](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/#pausing-and-resuming-a-deployment) 以应用对 PodTemplateSpec 所作的多项修改， 然后恢复其执行以启动新的上线版本。
- [使用 Deployment 状态](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/#deployment-status) 来判定上线过程是否出现停滞。
- [清理较旧的不再需要的 ReplicaSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/#clean-up-policy) 。

## 命令行运行一个Deployment

```shell
[root@clientvm ~]# kubectl create deployment -h
Usage:
  kubectl create deployment NAME --image=image -- [COMMAND] [args...] [options]

Examples:
  # Create a deployment named my-dep that runs the busybox image.
  kubectl create deployment my-dep --image=busybox

  # Create a deployment with command
  kubectl create deployment my-dep --image=busybox -- date

  # Create a deployment named my-dep that runs the nginx image with 3 replicas.
  kubectl create deployment my-dep --image=nginx --replicas=3

  # Create a deployment named my-dep that runs the busybox image and expose port 5701.
  kubectl create deployment my-dep --image=busybox --port=5701
  
#例:
## 创建
[root@clientvm ~]# kubectl create deployment testdep --image=nginx:1.8.1 -n mytest
deployment.apps/testdep created

## 查看
[root@clientvm k8s]# kubectl get deployments.apps -n mytest
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
testdep   1/1     1            1           44s
[root@clientvm k8s]# kubectl get pod -n mytest
NAME                       READY   STATUS    RESTARTS   AGE
testdep-64c98ff7d7-9ph5j   1/1     Running   0          50s

## 扩展
[root@clientvm ~]# kubectl get deployments.apps  -n mytest
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
testdep   2/2     2            2           4m6s
[root@clientvm ~]# kubectl get pod -n mytest
NAME                       READY   STATUS    RESTARTS   AGE
testdep-64c98ff7d7-6drmb   1/1     Running   0          64s
testdep-64c98ff7d7-9ph5j   1/1     Running   0          4m9s
```

## 创建Deployment

### 生成配置文件

```shell
[root@clientvm ~]# kubectl -n mytest create deployment dep-from-file --image=nginx --dry-run=client -o yaml >dep-from-file.yaml
```

### 根据需要修改配置

```shell
[root@clientvm ~]# vim dep-from-file.yaml
[root@clientvm ~]#
[root@clientvm ~]# cat dep-from-file.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: dep-from-file
  name: dep-from-file
  namespace: mytest
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dep-from-file
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: dep-from-file
    spec:
      containers:
      - image: nginx
        name: nginx
        imagePullPolicy: IfNotPresent
```

## 创建Deployment

```shell
[root@clientvm ~]# kubectl apply -f dep-from-file.yaml
deployment.apps/dep-from-file created
[root@clientvm ~]# kubectl get deployments.apps -n mytest
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
dep-from-file   2/2     2            2           15s
testdep         2/2     2            2           12m
```

## 查看RS

```shell
[root@clientvm ~]# kubectl get rs -n mytest
NAME                       DESIRED   CURRENT   READY   AGE
dep-from-file-7b78bc4877   2         2         2       72s
testdep-64c98ff7d7         2         2         2       13m
```

## 弹性伸缩

```shell
## 修改配置文件，修改replicas为4
[root@clientvm ~]# vim dep-from-file.yaml
[root@clientvm ~]# cat dep-from-file.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: dep-from-file
  name: dep-from-file
  namespace: mytest
spec:
  replicas: 4
  selector:
  ......

## 应用修改
[root@clientvm ~]# kubectl apply -f dep-from-file.yaml
deployment.apps/dep-from-file configured

## 查看
[root@clientvm ~]# kubectl get deployments.apps -n mytest
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
dep-from-file   4/4     4            4           3m41s
```

## 滚动更新

```shell
[root@clientvm ~]# kubectl set image -h
Update existing container image(s) of resources.
Usage:
  kubectl set image (-f FILENAME | TYPE NAME) CONTAINER_NAME_1=CONTAINER_IMAGE_1 ... CONTAINER_NAME_N=CONTAINER_IMAGE_N
[options]

Examples:
  # Set a deployment's nginx container image to 'nginx:1.9.1', and its busybox container image to 'busybox'.
  kubectl set image deployment/nginx busybox=busybox nginx=nginx:1.9.1

  # Update all deployments' and rc's nginx container's image to 'nginx:1.9.1'
  kubectl set image deployments,rc nginx=nginx:1.9.1 --all

  # Update image of all containers of daemonset abc to 'nginx:1.9.1'
  kubectl set image daemonset abc *=nginx:1.9.1

  # Print result (in yaml format) of updating nginx container image from local file, without hitting the server
  kubectl set image -f path/to/file.yaml nginx=nginx:1.9.1 --local -o yaml
  
#example1
[root@clientvm ~]# kubectl get deployments.apps -n mytest
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
dep-from-file   4/4     4            4           2m32s
testdep         2/2     2            2           24m
[root@clientvm ~]#
[root@clientvm ~]# kubectl set image deployment dep-from-file nginx=nginx:1.9.1 -n mytest
deployment.apps/dep-from-file image updated

## 查看
[root@clientvm ~]# kubectl rollout status deployment dep-from-file -n mytest
deployment "dep-from-file" successfully rolled out

[root@clientvm ~]# kubectl rollout status deployment dep-from-file -n mytest
deployment "dep-from-file" successfully rolled out
[root@clientvm ~]# kubectl describe deployments.apps -n mytest  dep-from-file
Name:                   dep-from-file
Namespace:              mytest
......
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  39m   deployment-controller  Scaled up replica set dep-from-file-7b78bc4877 to 4
  Normal  ScalingReplicaSet  34m   deployment-controller  Scaled up replica set dep-from-file-7dfb96d9dd to 1
  Normal  ScalingReplicaSet  34m   deployment-controller  Scaled down replica set dep-from-file-7b78bc4877 to 3
  Normal  ScalingReplicaSet  34m   deployment-controller  Scaled up replica set dep-from-file-7dfb96d9dd to 2
  Normal  ScalingReplicaSet  12m   deployment-controller  Scaled down replica set dep-from-file-7b78bc4877 to 2
  Normal  ScalingReplicaSet  12m   deployment-controller  Scaled up replica set dep-from-file-7dfb96d9dd to 3
  Normal  ScalingReplicaSet  11m   deployment-controller  Scaled down replica set dep-from-file-7b78bc4877 to 1
  Normal  ScalingReplicaSet  11m   deployment-controller  Scaled up replica set dep-from-file-7dfb96d9dd to 4
  Normal  ScalingReplicaSet  11m   deployment-controller  Scaled down replica set dep-from-file-7b78bc4877 to 0
  
#example2
[root@clientvm ~]# kubectl set image deployment dep-from-file nginx=nginx:1.8.1 -n mytest --record
deployment.apps/dep-from-file image updated

[root@master ~]# kubectl get pod -n mytest
NAME                             READY   STATUS              RESTARTS   AGE
dep-from-file-7478c844bc-6x8d4   0/1     ContainerCreating   0          2m
dep-from-file-7478c844bc-bf2rw   1/1     Running             0          66s
dep-from-file-7478c844bc-krjj4   1/1     Running             0          2m
dep-from-file-7478c844bc-tvsv4   0/1     ContainerCreating   0          64s
dep-from-file-7dfb96d9dd-9hzls   1/1     Running             0          39m

[root@clientvm ~]# kubectl rollout status deployment dep-from-file -n mytest
Waiting for deployment "dep-from-file" rollout to finish: 2 out of 4 new replicas have been updated...
Waiting for deployment "dep-from-file" rollout to finish: 2 out of 4 new replicas have been updated...
Waiting for deployment "dep-from-file" rollout to finish: 2 out of 4 new replicas have been updated...
Waiting for deployment "dep-from-file" rollout to finish: 3 out of 4 new replicas have been updated...
Waiting for deployment "dep-from-file" rollout to finish: 3 out of 4 new replicas have been updated...
Waiting for deployment "dep-from-file" rollout to finish: 3 out of 4 new replicas have been updated...
Waiting for deployment "dep-from-file" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "dep-from-file" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "dep-from-file" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "dep-from-file" rollout to finish: 3 of 4 updated replicas are available...
deployment "dep-from-file" successfully rolled out

## 查看历史记录
[root@clientvm ~]# kubectl rollout history deployment -n mytest dep-from-file
deployment.apps/dep-from-file
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         kubectl set image deployment dep-from-file nginx=nginx:1.8.1 --namespace=mytest --record=true

## 查看历史记录详细信息
[root@clientvm ~]# kubectl rollout history deployment -n mytest  dep-from-file --revision=3


## 回滚版本
[root@clientvm ~]# kubectl rollout undo deployment dep-from-file -n mytest --to-revision=2
deployment.apps/dep-from-file rolled back

[root@clientvm ~]# kubectl rollout status deployment dep-from-file -n mytest
deployment "dep-from-file" successfully rolled out


##验证Pod nginx版本
[root@clientvm ~]# kubectl edit pod dep-from-file-7dfb96d9dd-5hghh -n mytest
......
spec:
  containers:
  - image: nginx:1.9.1
    imagePullPolicy: IfNotPresent
......
```

## 升级策略

```shell
[root@clientvm ~]# kubectl explain deployment.spec.strategy.rollingUpdate
KIND:     Deployment
VERSION:  apps/v1

RESOURCE: rollingUpdate <Object>

DESCRIPTION:
     Rolling update config params. Present only if DeploymentStrategyType =
     RollingUpdate.

     Spec to control the desired behavior of rolling update.

FIELDS:
   maxSurge     <string>
     The maximum number of pods that can be scheduled above the desired number
     of pods. Value can be an absolute number (ex: 5) or a percentage of desired
     pods (ex: 10%). This can not be 0 if MaxUnavailable is 0. Absolute number
     is calculated from percentage by rounding up. Defaults to 25%. Example:
     when this is set to 30%, the new ReplicaSet can be scaled up immediately
     when the rolling update starts, such that the total number of old and new
     pods do not exceed 130% of desired pods. Once old pods have been killed,
     new ReplicaSet can be scaled up further, ensuring that total number of pods
     running at any time during the update is at most 130% of desired pods.

   maxUnavailable       <string>
     The maximum number of pods that can be unavailable during the update. Value
     can be an absolute number (ex: 5) or a percentage of desired pods (ex:
     10%). Absolute number is calculated from percentage by rounding down. This
     can not be 0 if MaxSurge is 0. Defaults to 25%. Example: when this is set
     to 30%, the old ReplicaSet can be scaled down to 70% of desired pods
     immediately when the rolling update starts. Once new pods are ready, old
     ReplicaSet can be scaled down further, followed by scaling up the new
     ReplicaSet, ensuring that the total number of pods available at all times
     during the update is at least 70% of desired pods.
     
# MaxUnavailable：滚动过程中最多有多少个 Pod 不可用；
# MaxSurge：滚动过程中最多存在多少个 Pod 超过预期 replicas 数量。
```

## 其他操作

```shell
[root@clientvm ~]# kubectl rollout -h
Manage the rollout of a resource.
Available Commands:
  history     View rollout history
  pause       Mark the provided resource as paused
  restart     Restart a resource
  resume      Resume a paused resource
  status      Show the status of the rollout
  undo        Undo a previous rollout
```

## 删除Deployment

```shell
[root@clientvm ~]# kubectl delete -f dep-from-file.yaml
deployment.apps "dep-from-file" deleted
[root@clientvm ~]# kubectl get deployments.apps -n mytest
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
testdep   2/2     2            2           129m
```

