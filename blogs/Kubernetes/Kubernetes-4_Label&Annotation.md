---
title: K8S-4_标签与标签选择器（Label&Annotation）
tags: 
  - Kubernetes
date: 2023-10-26 09:00:00
categories:	
  - Kubernetes
---

## 标签

### 基本理论

标签（Labels）是附加到 Kubernetes 对象（比如 Pods）上的键值对。 标签旨在用于指定对用户有意义且相关的对象的标识属性，但不直接对核心系统有语义含义。 标签可以用于组织和选择对象的子集。标签可以在创建时附加到对象，随后可以随时添加和修改。 每个对象都可以定义一组键/值标签。每个键对于给定对象必须是唯一的。

```json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

### 命令行创建带标签的POD

```shell
[root@clientvm ~]# kubectl create namespace mytest
namespace/mytest created
[root@clientvm ~]# kubectl run labelpod --image=nginx --labels="app=myweb,env=prod" -n mytest
pod/labelpod created
[root@clientvm ~]# kubectl get -n mytest pod
NAME       READY   STATUS    RESTARTS   AGE
labelpod   1/1     Running   0          2m2s
```

### 查看pod的标签

```shell
[root@clientvm ~]# kubectl get -n mytest pod --show-labels
NAME       READY   STATUS    RESTARTS   AGE     LABELS
labelpod   1/1     Running   0          2m28s   app=myweb,env=prod
```

### 为pod添加新标签

```shell
[root@clientvm ~]# kubectl label pod labelpod version=v1 -n mytest
pod/labelpod labeled
[root@clientvm ~]# kubectl get -n mytest pod --show-labels
NAME       READY   STATUS    RESTARTS   AGE     LABELS
labelpod   1/1     Running   0          2m50s   app=myweb,env=prod,version=v1
```

### 删除pod标签

```shell
[root@clientvm ~]# kubectl label pod labelpod version- -n mytest
pod/labelpod labeled
[root@clientvm ~]# kubectl get -n mytest pod --show-labels
NAME       READY   STATUS    RESTARTS   AGE     LABELS
labelpod   1/1     Running   0          3m27s   app=myweb,env=prod
```

### 使用yaml文件创建带标签的Pod

#### 创建yaml文件

```shell
[root@clientvm ~]# kubectl run labelpod-yaml --image=nginx --labels="app=myweb,env=prod" -n mytest --dry-run=client  -o yaml >labelpod.yaml

[root@clientvm ~]# vim labelpod.yaml
[root@clientvm ~]# cat labelpod.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: myweb
    env: prod
  name: labelpod-yaml
  namespace: mytest
spec:
  containers:
  - image: nginx
    name: labelpod-yaml
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

#### 创建Pod

```shell
[root@clientvm ~]# kubectl apply -f labelpod.yaml
pod/labelpod-yaml created
[root@clientvm ~]# kubectl get pod -n mytest --show-labels
NAME            READY   STATUS    RESTARTS   AGE   LABELS
labelpod        1/1     Running   0          15m   app=myweb,env=prod
labelpod-yaml   1/1     Running   0          89s   app=myweb,env=prod
```

## 标签选择器（selector）

-L 查看指定标签的值

-l 查看符合标签的对象

```shell
[root@clientvm ~]# kubectl label pod labelpod version=v1 -n mytest
pod/labelpod labeled
[root@clientvm ~]# kubectl get pod -l version=v1 -n mytest
NAME       READY   STATUS    RESTARTS   AGE
labelpod   1/1     Running   0          18m

[root@clientvm ~]# kubectl get pod -l version=v1 -n mytest -L app
NAME       READY   STATUS    RESTARTS   AGE   APP
labelpod   1/1     Running   0          19m   myweb

[root@clientvm ~]# kubectl get pod -l version!=v1 -n mytest -L app
NAME            READY   STATUS    RESTARTS   AGE     APP
labelpod-yaml   1/1     Running   0          7m34s   myweb
```

## Annotation

### 理论基础

你可以使用标签或注解将元数据附加到 Kubernetes 对象。 标签可以用来选择对象和查找满足某些条件的对象集合。 相反，注解不用于标识和选择对象。 注解中的元数据，可以很小，也可以很大，可以是结构化的，也可以是非结构化的，能够包含标签不允许的字符。

注解和标签一样，是键/值对:

```json
"metadata": {
  "annotations": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

以下是一些例子，用来说明哪些信息可以使用注解来记录:

- 声明性配置所管理的字段。 将这些字段附加为注解，能够将它们与客户端或服务端设置的默认值、 自动生成的字段以及通过自动调整大小或自动伸缩系统设置的字段区分开来。
- 构建、发布或镜像信息（如时间戳、发布 ID、Git 分支、PR 数量、镜像哈希、仓库地址）。
- 指向日志记录、监控、分析或审计仓库的指针。
- 可用于调试目的的客户端库或工具信息：例如，名称、版本和构建信息。
- 用户或者工具/系统的来源信息，例如来自其他生态系统组件的相关对象的 URL。
- 轻量级上线工具的元数据信息：例如，配置或检查点。
- 负责人员的电话或呼机号码，或指定在何处可以找到该信息的目录条目，如团队网站。
- 从用户到最终运行的指令，以修改行为或使用非标准功能。

### 查看Annotation

```shell
[root@clientvm ~]# kubectl describe nodes master.example.com | more
Name:               master.example.com
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=master.example.com
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/master=
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"5e:48:c6:73:ac:7c"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 192.168.241.129
                    kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
```

### 添加Annotation

```shell
[root@clientvm ~]# kubectl annotate nodes master.example.com disk=ssd
node/master.example.com annotated
[root@clientvm ~]# kubectl describe nodes master.example.com | grep disk
Annotations:        disk: ssd
```

### 删除Annotation

```shell
[root@clientvm ~]# kubectl annotate nodes master.example.com disk-
node/master.example.com annotated
```
