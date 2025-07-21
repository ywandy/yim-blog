---
title: K8S-2_集群（Namespace）
tags: 
  - Kubernetes
date: 2023-10-26 09:00:00
categories:	
  - Kubernetes
---

## 理论基础

​		Kubernetes 支持多个虚拟集群，它们底层依赖于同一个物理集群。 这些虚拟集群被称为名字空间。名字空间适用于存在很多跨多个团队或项目的用户的场景。对于只有几到几十个用户的集群，根本不需要创建或考虑名字空间。当需要名称空间提供的功能时，请开始使用它们。名字空间为名称提供了一个范围。资源的名称需要在名字空间内是唯一的，但不能跨名字空间。 名字空间不能相互嵌套，每个 Kubernetes 资源只能在一个名字空间中。名字空间是在多个用户之间划分集群资源的一种方法（通过[资源配额](https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/)）。不需要使用多个名字空间来分隔轻微不同的资源，例如同一软件的不同版本： 使用[标签](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels)来区分同一名字空间中的不同资源。

## 查看信息

```shell
#获取NameSpace
kubectl get namespaces

#查看NameSpace下的资源
kubectl get all -n kube-system

#创建NameSpace
kubectl create ns my-web
kubectl get ns

#删除NameSpace
kubectl delete namespaces myweb
```

注意：删除NameSpace会删除NameSpace中的所有资源，如果其中有资源删不掉，这个NameSpace可能会处于Terminating状态，停止响应