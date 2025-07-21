---
title: K8S-15_集群资源管理（ClusterManagement）
tags: 
  - Kubernetes
date: 2023-10-26 09:00:00
categories:	
  - Kubernetes
---

## NameSpace资源配额

### 基础理论

当多个用户或团队共享具有固定节点数目的集群时，人们会担心有人使用超过其基于公平原则所分配到的资源量。

资源配额，通过 ResourceQuota 对象来定义，对每个命名空间的资源消耗总量提供限制。 它可以限制命名空间中某种类型的对象的总数目上限，也可以限制命令空间中的 Pod 可以使用的计算资源的总上限。

在集群容量小于各命名空间配额总和的情况下，可能存在资源竞争。资源竞争时，Kubernetes 系统会遵循先到先得的原则。

不管是资源竞争还是配额的修改，都不会影响已经创建的资源使用对象。

对NameSpace定义ResourceQuota后，在该NameSpace中创建的所有Pod都需要定义相应的资源。

https://kubernetes.io/docs/concepts/policy/resource-quotas/

### 配置资源配额

Limit：最大允许使用资源量

request：最小需要资源量

```yaml
[root@clientvm ~]# cat compute-resources.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    configmaps: "10"
    persistentvolumeclaims: "4"
    pods: "4"
    replicationcontrollers: "20"
    secrets: "10"
    services: "10"
    
[root@clientvm ~]# kubectl apply -f compute-resources.yaml -n mytest
resourcequota/compute-resources created

[root@clientvm ~]# kubectl describe resourcequotas compute-resources -n mytest
Name:                   compute-resources
Namespace:              mytest
Resource                Used  Hard
--------                ----  ----
configmaps              0     10
limits.cpu              0     2
limits.memory           0     2Gi
persistentvolumeclaims  0     4
pods                    1     4
replicationcontrollers  0     20
requests.cpu            0     1
requests.memory         0     1Gi
secrets                 1     10
services                0     10
```

### 内存限制

```shell
[root@clientvm ~]# cat memory-limit-pod1.yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo1
  namespace: mytest
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        memory: "200Mi"
        cpu: "200m"
      requests:
        cpu: "200m"
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
    
[root@clientvm ~]# kubectl get pod -n mytest memory-demo1
NAME           READY   STATUS    RESTARTS   AGE
memory-demo1   1/1     Running   0          3m8s

#更改分配资源重新创建：
[root@clientvm ~]# cat memory-limit-pod2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo2
  namespace: mytest
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        memory: "100Mi"
        cpu: "200m"
      requests:
        cpu: "200m"
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
    
[root@clientvm ~]# kubectl apply -f memory-limit-pod2.yaml -n mytest
pod/memory-demo2 created

[root@clientvm ~]# kubectl get pod -n mytest
NAME                     READY   STATUS      RESTARTS   AGE
memory-demo1             1/1     Running     0          11m
memory-demo2             0/1     OOMKilled   2          27s
sidecar-container-demo   2/2     Running     0          3h42m
```

### CPU限制

CPU的分配以m为最小单位，称为毫，一个cpu=1000m

```shell
[root@clientvm ~]# cat cpu-limit-pod1.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo-1
  namespace: mytest
spec:
  containers:
  - name: cpu-demo-ctr
    image: vish/stress
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        cpu: "1"
      requests:
        cpu: "0.5"
    args:
    - -cpus
    - "2"
    
[root@clientvm ~]# kubectl apply -f cpu-limit-pod1.yaml
pod/cpu-demo-1 created

[root@clientvm ~]# kubectl get pod -n mytest
NAME                     READY   STATUS    RESTARTS   AGE
cpu-demo-1               1/1     Running   0          2m27s

#修改limit和request后重新部署
[root@clientvm ~]# cat cpu-limit-pod2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo-2
  namespace: mytest
spec:
  containers:
  - name: cpu-demo-ctr
    image: vish/stress
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        cpu: "16"
      requests:
        cpu: "16"
    args:
    - -cpus
    - "8"
    
[root@clientvm ~]# kubectl apply -f cpu-limit-pod2.yaml
pod/cpu-demo-2 created

[root@clientvm ~]# kubectl get pod -n mytest
NAME                     READY   STATUS    RESTARTS   AGE
cpu-demo-1               1/1     Running   0          19m
cpu-demo-2               0/1     Pending   0          6s
sidecar-container-demo   2/2     Running   0          4h6m
[root@clientvm ~]# kubectl describe pod -n mytest cpu-demo-2
......
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  24s (x2 over 24s)  default-scheduler  0/3 nodes are available: 3 Insufficient cpu.
```

### LimitRange

默认情况下， Kubernetes 集群上的容器运行使用的[计算资源](https://kubernetes.io/zh/docs/concepts/configuration/manage-resources-containers/)没有限制。 使用资源配额，集群管理员可以以[名字空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/)为单位，限制其资源的使用与创建。 在命名空间中，一个 Pod 或 Container 最多能够使用命名空间的资源配额所定义的 CPU 和内存用量。 有人担心，一个 Pod 或 Container 会垄断所有可用的资源。 LimitRange 是在命名空间内限制资源分配（给多个 Pod 或 Container）的策略对象。

一个 *LimitRange（限制范围）* 对象提供的限制能够做到：

- 在一个命名空间中实施对每个 Pod 或 Container 最小和最大的资源使用量的限制。
- 在一个命名空间中实施对每个 PersistentVolumeClaim 能申请的最小和最大的存储空间大小的限制。
- 在一个命名空间中实施对一种资源的申请值和限制值的比值的控制。
- 设置一个命名空间中对计算资源的默认申请/限制值，并且自动的在运行时注入到多个 Container 中

创建LimitRange

```shell
[root@clientvm ~]# kubectl explain limitrange.spec.limits
KIND:     LimitRange
VERSION:  v1

RESOURCE: limits <[]Object>

DESCRIPTION:
     Limits is the list of LimitRangeItem objects that are enforced.

     LimitRangeItem defines a min/max usage limit for any resource that matches
     on kind.

FIELDS:
   default      <map[string]string>
     Default resource requirement limit value by resource name if resource limit
     is omitted.

   defaultRequest       <map[string]string>
     DefaultRequest is the default resource requirement request value by
     resource name if resource request is omitted.

   max  <map[string]string>
     Max usage constraints on this kind by resource name.

   maxLimitRequestRatio <map[string]string>
     MaxLimitRequestRatio if specified, the named resource must have a request
     and limit that are both non-zero where limit divided by request is less
     than or equal to the enumerated value; this represents the max burst for
     the named resource.

   min  <map[string]string>
     Min usage constraints on this kind by resource name.

   type <string> -required-
     Type of resource that this limit applies to.

[root@clientvm ~]# cat limitRange-cpu-container.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-min-max-demo
spec:
  limits:
  - max:
      cpu: "800m"
    min:
      cpu: "200m"
    type: Container
    
[root@clientvm ~]# kubectl apply -f limitRange-cpu-container.yaml -n mytest
limitrange/cpu-min-max-demo created
[root@clientvm ~]# kubectl describe limitranges -n mytest
Name:       cpu-min-max-demo
Namespace:  mytest
Type        Resource  Min   Max   Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---   ---   ---------------  -------------  -----------------------
Container   cpu       200m  800m  800m             800m

#尝试创建超出CPU范围的Pod会报错
[root@clientvm ~]# cat limitRange-out-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo-2
spec:
  containers:
  - name: constraints-cpu-demo-2
    image: nginx
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        cpu: "1.5"
      requests:
        cpu: "500m"
 
[root@clientvm ~]# kubectl apply -f limitRange-out-pod.yaml -n mytest
Error from server (Forbidden): error when creating "limitRange-out-pod.yaml": pods "constraints-cpu-demo-2" is forbidden: maximum cpu usage per Container is 800m, but limit is 1500m

#尝试创建未指定CPU大小的Pod
[root@clientvm ~]# cat limitRange-in-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo-2
spec:
  containers:
  - name: constraints-cpu-demo-2
    image: nginx
    imagePullPolicy: IfNotPresent

[root@clientvm ~]# kubectl apply -f limitRange-in-pod.yaml -n mytest
pod/constraints-cpu-demo-2 created

[root@clientvm ~]# kubectl describe pod constraints-cpu-demo-2 -n mytest
    Limits:
      cpu:  800m
    Requests:
      cpu:        800m
```

