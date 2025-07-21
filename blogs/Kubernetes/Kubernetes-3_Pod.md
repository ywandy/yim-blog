---
title: K8S-3_容器（Pod）
tags: 
  - Kubernetes
date: 2023-10-26 09:00:00
categories:	
  - Kubernetes
---

## 理论基础

Pod 是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元。

Pod （就像在鲸鱼荚或者豌豆荚中）是一组（一个或多个） [容器](https://kubernetes.io/zh/docs/concepts/overview/what-is-kubernetes/#why-containers)； 这些容器共享存储、网络、以及怎样运行这些容器的声明。 Pod 中的内容总是并置（colocated）的并且一同调度，在共享的上下文中运行。 Pod 所建模的是特定于应用的“逻辑主机”，其中包含一个或多个应用容器， 这些容器是相对紧密的耦合在一起的。 在非云环境中，在相同的物理机或虚拟机上运行的应用类似于 在同一逻辑主机上运行的云应用。

除了应用容器，Pod 还可以包含在 Pod 启动期间运行的[Init 容器](https://kubernetes.io/zh/docs/concepts/workloads/pods/init-containers/)。 你也可以在集群中支持[临时性容器](https://kubernetes.io/zh/docs/concepts/workloads/pods/ephemeral-containers/)的情况外，为调试的目的注入临时性容器。

Pod 的共享上下文包括一组 Linux 名字空间、控制组（cgroup）和可能一些其他的隔离 方面，即用来隔离 Docker 容器的技术。 在 Pod 的上下文中，每个独立的应用可能会进一步实施隔离。

就 Docker 概念的术语而言，Pod 类似于共享名字空间和文件系统卷的一组 Docker 容器。

## 创建Pod

```shell
#使用命令行创建Pod
kubectl run busybox --image=busybox -n mytest -- sleep 10000

#使用yaml创建Pod
kubectl run busybox2 --image=busybox -n mytest --dry-run=client -o yaml -- sleep 10000

vim busybox2.yaml
```

## 描述Pod的YAML组成

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: busybox2
  name: busybox2
  namespace: mytest
spec:
  containers:
  - args:
    - sleep
    - "10000"
    image: busybox
    name: busybox2
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

## 修改Pod

```shell
#命令行修改
kubectl edit pod -n mytest busybox2
##增加一个标签
  labels:
    run: busybox2
    app: web

#修改yaml文件
vim busybox2.yaml

......
metadata:
  labels:
    run: busybox2
    app: web1
    
kubectl apply -f busybox2.yaml


kubectl get -n mytest pod -l app=web
No resources found in mytest namespace.

kubectl get -n mytest pod -l app=web1

#通过命令行patch修改
kubectl patch -n mytest pod busybox2 -p '{"metadata": {"labels": {"app": "web2"}}}'
```

## 进入Pod中的容器

```shell
kubectl exec -n mytest -it busybox2 -- /bin/sh
```

## 查看容器运行的主机

```shell
kubectl get -n mytest pod -o wide

ssh xxx
```

## 创建多容器Pod

```shell
[root@clientvm ~]# cat multi-containers.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: multi-container
    app: web1
  name: multi-container
  namespace: mytest
spec:
  containers:
  - name: nginx
    image: nginx
  - name: busybox
    image: busybox
    args:
    - sleep
    - "10000"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  
 [root@clientvm ~]# kubectl apply -f multi-containers.yaml
pod/multi-container created
 
 [root@clientvm ~]# kubectl get -n mytest pod
NAME              READY   STATUS    RESTARTS   AGE
busybox           1/1     Running   0          45m
busybox2          1/1     Running   0          35m
labelpod          1/1     Running   0          80m
labelpod-yaml     1/1     Running   0          66m
multi-container   2/2     Running   0          99s

##进入多容器Pod中的某一个容器加-c选项
[root@clientvm ~]# kubectl exec -n mytest  -it multi-container -c nginx -- /bin/bash
root@multi-container:/# ls
bin  boot  dev  docker-entrypoint.d  docker-entrypoint.sh  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

## Init容器

init 容器是一种特殊容器，在 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 内的应用容器启动之前运行。Init 容器可以包括一些应用镜像中不存在的实用工具和安装脚本。

每个 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 中可以包含多个容器， 应用运行在这些容器里面，同时 Pod 也可以有一个或多个先于应用容器启动的 Init 容器。

Init 容器与普通的容器非常像，除了如下两点：

- 它们总是运行到完成。
- 每个都必须在下一个启动之前成功完成。

如果 Pod 的 Init 容器失败，kubelet 会不断地重启该 Init 容器直到该容器成功为止。如果为一个 Pod 指定了多个 Init 容器，这些容器会按顺序逐个运行。 每个 Init 容器必须运行成功，下一个才能够运行。当所有的 Init 容器运行完成时， Kubernetes 才会为 Pod 初始化应用容器并像平常一样运行。

```txt
[root@clientvm ~]# cat init-container.yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
  namespace: mytest
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 36000']
  initContainers:
  - name: init-container
    image: busybox:1.28
    command: ['sh', '-c', "echo hello"]
```

## Sidecar Pod

```shell
cat sidecar-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: sidecar-container-demo
spec:
  containers:
  - image: busybox
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo echo $(date -u) 'Hi I am from Sidecar container' >> /var/log/index.html; sleep 5;done"]
    name: sidecar-container
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "sleep 20"]
    resources: {}
    volumeMounts:
    - name: var-logs
      mountPath: /var/log
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: main-container
    resources: {}
    ports:
      - containerPort: 80
    volumeMounts:
    - name: var-logs
      mountPath: /usr/share/nginx/html
  dnsPolicy: Default
  volumes:
  - name: var-logs
    emptyDir: {}

kubectl apply -f sidecar-pod.yaml -n mytest
```
