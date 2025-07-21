---
title: K8S-14_资源调度（NodeSelector）
tags: 
  - Kubernetes
date: 2023-11-27 09:00:00
categories:	
  - Kubernetes
---

## nodeSelector

### 基础理论

`nodeSelector` 是节点选择约束的最简单推荐形式。`nodeSelector` 是 PodSpec 的一个字段。 它包含键值对的映射。为了使 pod 可以在某个节点上运行，该节点的标签中必须包含这里的每个键值对（它也可以具有其他标签）。最常见的用法的是一对键值对。

### 使用nodeSelector

```shell
#查看节点标签
[root@clientvm ~]# kubectl get nodes --show-labels
NAME                  STATUS   ROLES    AGE   VERSION   LABELS
master.example.com    Ready    master   17d   v1.19.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master.example.com,kubernetes.io/os=linux,node-role.kubernetes.io/master=
worker1.example.com   Ready    <none>   17d   v1.19.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker1.example.com,kubernetes.io/os=linux
worker2.example.com   Ready    <none>   17d   v1.19.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker2.example.com,kubernetes.io/os=linux

#给节点打标签
[root@clientvm ~]# kubectl label nodes worker1.example.com disk=ssd
node/worker1.example.com labeled

#使用nodeSelector调度Pod
[root@clientvm ~]# cat nodeSelector-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-nodeselector
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disk: ssd

[root@clientvm ~]# kubectl apply -f nodeSelector-pod.yaml
pod/nginx-nodeselector created

[root@clientvm ~]# kubectl get pod -o wide
NAME                                              READY   STATUS      RESTARTS   AGE    IP            NODE                  NOMINATED NODE   READINESS GATES
nginx-nodeselector                                1/1     Running     0          19s    10.244.1.57   worker1.example.com   <none>           <none>

```

## NodeName

### 基础理论

`nodeName` 是节点选择约束的最简单方法，但是由于其自身限制，通常不使用它。`nodeName` 是 PodSpec 的一个字段。如果它不为空，调度器将忽略 pod，并且运行在它指定节点上的 kubelet 进程尝试运行该 pod。因此，如果 `nodeName` 在 PodSpec 中指定了，则它优先于上面的节点选择方法。 如果指定的节点不存在，将调度失败。

### 使用NodeName

```shell
[root@clientvm ~]# cat nodeName.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-nodename
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeName: worker1.example.com

[root@clientvm ~]# kubectl apply -f nodeName.yaml
pod/nginx-nodename created

[root@clientvm ~]# kubectl get pod nginx-nodename -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP            NODE                  NOMINATED NODE   READINESS GATES
nginx-nodename   1/1     Running   0          37s   10.244.1.58   worker1.example.com   <none>           <none>

```

## 亲和度

### 基础理论

`nodeSelector` 提供了一种非常简单的方法来将 pod 约束到具有特定标签的节点上。亲和/反亲和功能极大地扩展了你可以表达约束的类型。关键的增强点是

1. 语言更具表现力（不仅仅是“完全匹配的 AND”）
2. 你可以发现规则是“偏好”，而不是硬性要求，因此，如果调度器无法满足该要求，仍然调度该 pod
3. 你可以使用节点上的 pod 的标签来约束，而不是使用节点本身的标签，来允许哪些 pod 可以或者不可以被放置在一起。

亲和功能包含两种类型的亲和，即“节点亲和”和“pod 间亲和/反亲和”。节点亲和就像现有的`nodeSelector`，然而 pod 间亲和/反亲和约束 pod 标签而不是节点标签。

目前有两种类型的节点亲和，分别为 `requiredDuringSchedulingIgnoredDuringExecution` 和 `preferredDuringSchedulingIgnoredDuringExecution`。你可以视它们为“硬”和“软”，意思是，前者指定了将 pod 调度到一个节点上*必须*满足的规则（就像 `nodeSelector` 但使用更具表现力的语法），后者指定调度器将尝试执行但不能保证的*偏好*。

### 使用节点亲和

```shell
[root@clientvm deploy]# kubectl label nodes worker1.example.com app=web
node/worker1.example.com labeled
[root@clientvm deploy]# kubectl label nodes worker2.example.com disk=hybrid
node/worker2.example.com labeled

[root@clientvm ~]# cat node-affinity.yaml
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
          - key: disk
            operator: In
            values:
            - ssd
            - hybrid
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: app
            operator: In
            values:
            - web
  containers:
  - name: with-node-affinity
    image: nginx
    imagePullPolicy: IfNotPresent

[root@clientvm ~]# kubectl apply -f node-affinity.yaml
pod/with-node-affinity created

[root@clientvm ~]# kubectl get pod with-node-affinity -o wide
NAME                 READY   STATUS    RESTARTS   AGE   IP            NODE                  NOMINATED NODE   READINESS GATES
with-node-affinity   1/1     Running   0          13s   10.244.1.59   worker1.example.com   <none>           <none>

#重新调度
[root@clientvm deploy]# kubectl label nodes worker1.example.com disk-
node/worker1.example.com labeled

[root@clientvm ~]# kubectl delete -f node-affinity.yaml
pod "with-node-affinity" deleted
[root@clientvm ~]#
[root@clientvm ~]# kubectl apply -f node-affinity.yaml
pod/with-node-affinity created

[root@clientvm ~]# kubectl get pod with-node-affinity -o wide
NAME                 READY   STATUS    RESTARTS   AGE   IP            NODE                  NOMINATED NODE   READINESS GATES
with-node-affinity   1/1     Running   0          4s    10.244.2.73   worker2.example.com   <none>           <none>
```

## pod 间亲和与反亲和

pod 间亲和与反亲和使你可以*基于已经在节点上运行的 pod 的标签*来约束 pod 可以调度到的节点，而不是基于节点上的标签。规则的格式为“如果 X 节点上已经运行了一个或多个 满足规则 Y 的pod，则这个 pod 应该（或者在非亲和的情况下不应该）运行在 X 节点”。

### 使用Pod亲和

```shell
[root@clientvm ~]# cat pod-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: run
            operator: In
            values:
            - nginx2
        topologyKey: disk
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: disk
  containers:
  - name: with-pod-affinity
    image: nginx
    imagePullPolicy: IfNotPresent

[root@clientvm ~]# kubectl apply -f pod-affinity.yaml
pod/with-pod-affinity created

[root@clientvm ~]# kubectl get pod with-pod-affinity -o wide
NAME                READY   STATUS    RESTARTS   AGE   IP            NODE                  NOMINATED NODE   READINESS GATES
with-pod-affinity   1/1     Running   0          14s   10.244.2.74   worker2.example.com   <none>           <none>
```

## 污点和容忍度

Taint（污点）与亲和性相反，它使节点能够排斥一类特定的 Pod。

容忍度（Tolerations）是应用于 Pod 上的，允许（但并不要求）Pod 调度到带有与之匹配的污点的节点上。

污点和容忍度（Toleration）相互配合，可以用来避免 Pod 被分配到不合适的节点上。 每个节点上都可以应用一个或多个污点，这表示对于那些不能容忍这些污点的 Pod，是不会被该节点接受的。

```shell
kubectl taint nodes node1 key1=value1:NoSchedule
```

- NoSchedule： 这表示只有拥有和这个污点相匹配的容忍度的 Pod 才能够被分配到 `node1` 这个节点.
- NoExecute ：任何不能忍受这个污点的 Pod 都会马上被驱逐。

用例：

- 专用节点：如果您想将某些节点专门分配给特定的一组用户使用，您可以给这些节点添加一个污点（即， `kubectl taint nodes nodename dedicated=groupName:NoSchedule`）， 然后给这组用户的 Pod 添加一个相对应的 toleration（通过编写一个自定义的 [准入控制器](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/)，很容易就能做到）。

- 配备了特殊硬件的节点：在部分节点配备了特殊硬件（比如 GPU）的集群中， 我们希望不需要这类硬件的 Pod 不要被分配到这些特殊节点，以便为后继需要这类硬件的 Pod 保留资源。 要达到这个目的，可以先给配备了特殊硬件的节点添加 taint （例如 `kubectl taint nodes nodename special=true:NoSchedule` 或 `kubectl taint nodes nodename special=true:PreferNoSchedule`)， 然后给使用了这类特殊硬件的 Pod 添加一个相匹配的 toleration。 

- 基于污点的驱逐: 这是在每个 Pod 中配置的在节点出现问题时的驱逐行为。

```shell
#examples
[root@clientvm ~]# cat taint.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-taint
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "GPU"
    operator: "Exists"
    effect: "NoSchedule"

[root@clientvm ~]# kubectl apply -f taint.yaml
pod/nginx-taint created

[root@clientvm ~]# kubectl get pod nginx-taint -o wide
NAME          READY   STATUS    RESTARTS   AGE   IP            NODE                  NOMINATED NODE   READINESS GATES
nginx-taint   1/1     Running   0          15s   10.244.1.60   worker1.example.com   <none>           <none>

[root@clientvm ~]# kubectl taint nodes worker1.example.com  GPU=yes:NoExecute
node/worker1.example.com tainted
[root@clientvm ~]# kubectl get pod nginx-taint -o wide
Error from server (NotFound): pods "nginx-taint" not found

[root@clientvm ~]# kubectl apply -f taint.yaml
pod/nginx-taint created
[root@clientvm ~]# kubectl get pod  -o wide
NAME                     READY   STATUS      RESTARTS   AGE    IP            NODE                  NOMINATED NODE   READINESS GATES
nginx-taint              1/1     Running     0          2s     10.244.2.76   worker2.example.com   <none>           <none>

```

operator 的默认值是 Equal。

一个容忍度和一个污点相“匹配”是指它们有一样的键名和效果，并且：

- 如果 operator 是 Exists （此时容忍度不能指定 value），或者
- 如果 operator 是 Equal ，则它们的 value 应该相等

**说明：**

存在两种特殊情况：

- 如果一个容忍度的 key 为空且 operator 为 Exists， 表示这个容忍度与任意的 key 、value 和 effect 都匹配，即这个容忍度能容忍任意 taint。
- 如果 effect 为空，则可以与所有键名 key1 的效果相匹配。

例如，假设您给一个节点添加了如下污点

```shell
kubectl taint nodes node1 key1=value1:NoSchedule 
kubectl taint nodes node1 key1=value1:NoExecute 
kubectl taint nodes node1 key2=value2:NoSchedule 
```

假定有一个 Pod，它有两个容忍度：

```shell
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

在这种情况下，上述 Pod 不会被分配到上述节点，因为其没有容忍度和第三个污点相匹配。 但是如果在给节点添加上述污点之前，该 Pod 已经在上述节点运行， 那么它还可以继续运行在该节点上，因为第三个污点是三个污点中唯一不能被这个 Pod 容忍的。

## 基于节点状态添加污点

控制平面使用节点[控制器](https://kubernetes.io/zh/docs/concepts/architecture/controller/)自动创建 与[节点状况](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/node-pressure-eviction/#node-conditions)对应的带有 NoSchedule 效应的污点。当某种条件为真时，节点控制器会自动给节点添加一个污点。当前内置的污点包括：

- node.kubernetes.io/not-ready：节点未准备好。这相当于节点状态 Ready 的值为 "False"。
- node.kubernetes.io/unreachable：节点控制器访问不到节点. 这相当于节点状态 Ready 的值为 "Unknown"。
- node.kubernetes.io/memory-pressure：节点存在内存压力。
- node.kubernetes.io/disk-pressure：节点存在磁盘压力。
- node.kubernetes.io/pid-pressure: 节点的 PID 压力。
- node.kubernetes.io/network-unavailable：节点网络不可用。
- node.kubernetes.io/unschedulable: 节点不可调度。

DaemonSet 控制器自动为所有守护进程添加如下 NoSchedule 容忍度以防 DaemonSet 崩溃：

- node.kubernetes.io/memory-pressure
- node.kubernetes.io/disk-pressure
- node.kubernetes.io/pid-pressure 
- node.kubernetes.io/unschedulable
