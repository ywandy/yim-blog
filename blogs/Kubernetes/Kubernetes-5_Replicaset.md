---
title: K8S-5_副本控制器（ReplicaSet）
tags: 
  - Kubernetes
date: 2023-10-26 09:00:00
categories:	
  - Kubernetes
---

## 使用探针保持Pod健康

针对运行中的容器，`kubelet` 可以选择是否执行以下三种探针，以及如何针对探测结果作出反应：

- `livenessProbe`：指示容器是否正在运行。如果存活态探测失败，则 kubelet 会杀死容器， 并且容器将根据其[重启策略](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy)决定未来。如果容器不提供存活探针， 则默认状态为 `Success`。如果容器中的进程能够在遇到问题或不健康的情况下自行崩溃，则不一定需要存活态探针; `kubelet` 将根据 Pod 的`restartPolicy` 自动执行修复操作。如果你希望容器在探测失败时被杀死并重新启动，那么请指定一个存活态探针， 并指定`restartPolicy` 为 "`Always`" 或 "`OnFailure`"。

- `readinessProbe`：指示容器是否准备好为请求提供服务。如果就绪态探测失败， 端点控制器将从与 Pod 匹配的所有服务的端点列表中删除该 Pod 的 IP 地址。 初始延迟之前的就绪态的状态值默认为 `Failure`。 如果容器不提供就绪态探针，则默认状态为 `Success`。如果要仅在探测成功时才开始向 Pod 发送请求流量，请指定就绪态探针。 在这种情况下，就绪态探针可能与存活态探针相同，但是规约中的就绪态探针的存在意味着 Pod 将在启动阶段不接收任何数据，并且只有在探针探测成功后才开始接收数据。

- `startupProbe`: 指示容器中的应用是否已经启动。如果提供了启动探针，则所有其他探针都会被 禁用，直到此探针成功为止。如果启动探测失败，`kubelet` 将杀死容器，而容器依其 [重启策略](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy)进行重启。 如果容器没有提供启动探测，则默认状态为 `Success`。对于所包含的容器需要较长时间才能启动就绪的 Pod 而言，启动探针是有用的。 你不再需要配置一个较长的存活态探测时间间隔，只需要设置另一个独立的配置选定， 对启动期间的容器执行探测，从而允许使用远远超出存活态时间间隔所允许的时长。

### 存活探针

```shell
[root@clientvm ~]# cat liveness-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    imagePullPolicy: IfNotPresent
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

`periodSeconds` 字段指定了 kubelet 应该每 5 秒执行一次存活探测。

`nitialDelaySeconds` 字段告诉 kubelet 在执行第一次探测前应该等待 5 秒。

kubelet 在容器内执行命令 `cat /tmp/healthy` 来进行探测。 如果命令执行成功并且返回值为 0，kubelet 就会认为这个容器是健康存活的。

这个容器生命的前 30 秒， `/tmp/healthy` 文件是存在的。 所以在这最开始的 30 秒内，执行命令 `cat /tmp/healthy` 会返回成功代码。 30 秒之后，执行命令 `cat /tmp/healthy` 就会返回失败代码。

### 创建Pod，观察状态

```shell
[root@clientvm ~]# kubectl apply -f liveness-pod.yaml -n mytest
[root@clientvm ~]# kubectl get pod -n mytest  --watch
NAME            READY   STATUS    RESTARTS   AGE
busybox         1/1     Running   43         4d23h
dns             1/1     Running   42         4d21h
liveness-exec   1/1     Running   0          19s
liveness-exec   1/1     Running   1          75s

[root@clientvm ~]# kubectl describe pod -n mytest liveness-exec
......
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  2m11s                default-scheduler  Successfully assigned mytest/liveness-exec to worker2.example.com
  Normal   Pulled     57s (x2 over 2m11s)  kubelet            Container image "busybox" already present on machine
  Normal   Created    56s (x2 over 2m11s)  kubelet            Created container liveness
  Normal   Started    56s (x2 over 2m10s)  kubelet            Started container liveness
  Warning  Unhealthy  12s (x6 over 97s)    kubelet            Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
  Normal   Killing    12s (x2 over 87s)    kubelet            Container liveness failed liveness probe, will be restarted
```

存活探针支持exec，httpGet，tcpSocket等多种不同的探测方式及其他相关参数配置：

```shell
[root@clientvm ~]# kubectl explain Pod.spec.containers.livenessProbe
KIND:     Pod
VERSION:  v1

RESOURCE: livenessProbe <Object>

DESCRIPTION:
     Periodic probe of container liveness. Container will be restarted if the
     probe fails. Cannot be updated. More info:
     https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes

     Probe describes a health check to be performed against a container to
     determine whether it is alive or ready to receive traffic.

FIELDS:
   exec <Object>
     One and only one of the following should be specified. Exec specifies the
     action to take.

   failureThreshold <integer>
     Minimum consecutive failures for the probe to be considered failed after
     having succeeded. Defaults to 3. Minimum value is 1.

   httpGet      <Object>
     HTTPGet specifies the http request to perform.

   initialDelaySeconds  <integer>
     Number of seconds after the container has started before liveness probes
     are initiated. More info:
     https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes

   periodSeconds    <integer>
     How often (in seconds) to perform the probe. Default to 10 seconds. Minimum
     value is 1.

   successThreshold <integer>
     Minimum consecutive successes for the probe to be considered successful
     after having failed. Defaults to 1. Must be 1 for liveness and startup.
     Minimum value is 1.

   tcpSocket    <Object>
     TCPSocket specifies an action involving a TCP port. TCP hooks not yet
     supported

   timeoutSeconds   <integer>
     Number of seconds after which the probe times out. Defaults to 1 second.
     Minimum value is 1. More info:
     https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes
```

### 就绪探针

```shell
[root@clientvm ~]# cat readiness-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: readiness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    imagePullPolicy: IfNotPresent
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

```shell
[root@clientvm ~]# kubectl get pod -w
NAME                                              READY   STATUS      RESTARTS   AGE
readiness-exec                                    1/1     Running     0          25s
readiness-exec                                    0/1     Running     0          43s
```

```shell
[root@clientvm ~]# kubectl describe pod readiness-exec
......
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  68s               default-scheduler  Successfully assigned default/readiness-exec to worker2.example.com
  Normal   Pulled     68s               kubelet            Container image "busybox" already present on machine
  Normal   Created    68s               kubelet            Created container liveness
  Normal   Started    67s               kubelet            Started container liveness
  Warning  Unhealthy  1s (x8 over 36s)  kubelet            Readiness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```

## ReplicaSet基础

ReplicaSet 的目的是维护一组在任何时候都处于运行状态的 Pod 副本的稳定集合。 因此，它通常用来保证给定数量的、完全相同的 Pod 的可用性。

RepicaSet 是通过一组字段来定义的，包括一个用来识别可获得的 Pod 的集合的选择算符、一个用来标明应该维护的副本个数的数值、一个用来指定应该创建新 Pod 以满足副本个数条件时要使用的 Pod 模板等等。 每个 ReplicaSet 都通过根据需要创建和 删除 Pod 以使得副本个数达到期望值， 进而实现其存在价值。当 ReplicaSet 需要创建新的 Pod 时，会使用所提供的 Pod 模板。

ReplicaSet 确保任何时间都有指定数量的 Pod 副本在运行。 然而，Deployment 是一个更高级的概念，它管理 ReplicaSet，并向 Pod 提供声明式的更新以及许多其他有用的功能。 因此，我们建议使用 Deployment 而不是直接使用 ReplicaSet，除非 你需要自定义更新业务流程或根本不需要更新。

这实际上意味着，你可能永远不需要操作 ReplicaSet 对象：而是使用 Deployment，并在 spec 部分定义你的应用。

### 创建ReplicaSet

```shell
[root@clientvm ~]# cat replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myweb
  namespace: mytest
  labels:
    app: guestbook
    tier: web
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: web
  template:
    metadata:
      labels:
        tier: web
    spec:
      containers:
      - name: nginx
        image: nginx
        imagePullPolicy: IfNotPresent
```

```shell
[root@clientvm ~]# kubectl apply -f replicaset.yaml
replicaset.apps/myweb created
[root@clientvm ~]#
[root@clientvm ~]# kubectl get pod -n mytest
NAME          READY   STATUS              RESTARTS   AGE
myweb-6tvp2   0/1     ContainerCreating   0          11s
myweb-krfmq   0/1     ContainerCreating   0          11s
myweb-l8qhv   0/1     ContainerCreating   0          11s
```

### 修改副本数

#### 命令行编辑

```shell
## 修改RS，设置副本数为1
[root@clientvm ~]# kubectl edit -n mytest rs myweb
replicaset.apps/myweb edited

[root@clientvm ~]# kubectl get pod -n mytest
NAME          READY   STATUS    RESTARTS   AGE
myweb-6tvp2   1/1     Running   0          10m
```

#### 修改yaml文件

```shell
## 修改yaml文件，设置副本数为2
[root@clientvm ~]# vim replicaset.yaml
......
spec:
  replicas: 2

[root@clientvm ~]# kubectl apply -f replicaset.yaml
replicaset.apps/myweb configured
[root@clientvm ~]# kubectl get pod -n mytest
NAME          READY   STATUS    RESTARTS   AGE
myweb-6tvp2   1/1     Running   0          21m
myweb-jtncn   1/1     Running   0          6s

```

#### 命令行

```shell
[root@clientvm ~]# kubectl scale --replicas=4 replicaset myweb -n mytest
replicaset.apps/myweb scaled
[root@clientvm ~]#
[root@clientvm ~]# kubectl get pod -n mytest
NAME          READY   STATUS    RESTARTS   AGE
myweb-6tvp2   1/1     Running   0          22m
myweb-bmbv8   1/1     Running   0          5s
myweb-glslz   1/1     Running   0          5s
myweb-jtncn   1/1     Running   0          116s
```

### 删除RS

```shell
[root@clientvm ~]# kubectl delete -n mytest rs myweb
replicaset.apps "myweb" deleted
```
