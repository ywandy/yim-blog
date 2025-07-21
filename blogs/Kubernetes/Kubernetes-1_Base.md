---
title: K8S-1_基本命令
tags: 
  - Kubernetes
date: 2023-10-26 09:00:00
categories:	
  - Kubernetes
---

```shell
#查看集群状态
kubectl cluster-info

#查看集群版本
kubectl version

#查看集群API信息
kubectl get node

#查看节点扩展信息
kubectl get node -o wide

#查看节点详细描述
kubectl describe nodes master.example.com

#将资源信息以yaml格式输出
kubectl get nodes master.example.com -o yaml

#将资源信息以json格式输出
kubectl get nodes master.example.com -o json

#explain命令
kubectl explain namespace
```
