---
title: K8S-7_每个节点上运行一个Pod（Daemonset）
tags: 
  - Kubernetes
date: 2023-10-26 09:00:00
categories:	
  - Kubernetes
---

## 理论基础

*DaemonSet* 确保每个节点上运行一个 Pod 的副本。 当有节点加入集群时， 也会为他们新增一个 Pod 。 当有节点从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

DaemonSet 的一些典型用法：

- 在每个节点上运行集群守护进程
- 在每个节点上运行日志收集守护进程
- 在每个节点上运行监控守护进程

一种简单的用法是为每种类型的守护进程在所有的节点上都启动一个 DaemonSet。 

## 创建DaemonSet

可以使用Deployment的yaml文件作为范例，更改kind为DaemonSet，删除replicas行

```shell
[root@clientvm ~]# cat DS.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx
  namespace: mytest
  labels:
    k8s-app: nginx-web
spec:
  selector:
    matchLabels:
      name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        
[root@clientvm ~]# kubectl apply -f DS.yaml
daemonset.apps/nginx created

[root@clientvm ~]# kubectl get pod -n mytest -o wide
NAME          READY   STATUS    RESTARTS   AGE   IP            NODE                  NOMINATED NODE   READINESS GATES
nginx-4jspl   1/1     Running   0          50s   10.244.1.25   worker1.example.com   <none>           <none>
nginx-l965c   1/1     Running   0          50s   10.244.2.35   worker2.example.com   <none>           <none>

[root@clientvm ~]# kubectl get daemonsets.apps -n mytest
NAME    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
nginx   2         2         2       2            2           <none>          103s

## 测试删除Pod
[root@clientvm ~]# kubectl delete pod -n mytest nginx-4jspl
pod "nginx-4jspl" deleted
[root@clientvm ~]# kubectl get pod -n mytest
NAME          READY   STATUS              RESTARTS   AGE
nginx-hgjtj   0/1     ContainerCreating   0          6s
nginx-l965c   1/1     Running             0          2m57s
```

