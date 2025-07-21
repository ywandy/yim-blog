---
title: K8S-sh-base
tags: 
  - Kubernetes
date: 2024-03-18 09:00:00
categories:	
  - Kubernetes
---

## K8S控制基本命令

### 显示资源列表

```sh
# kubectl get 资源类型
# 获取所有的node信息
kubectl get nodes
# 获取所有的deployment信息
kubectl get deployments

# 获取所有的deamonset信息
kubectl get deamonsets
# 获取所有的pod更宽的信息
kubectl get pods -o

# 名称空间
# 在命令后增加 -A 或 --all-namespaces 可查看所有 名称空间中 的对象，使用参数 -n 可查看指定名称空间的对象，例如
# 查看所有名称空间的 Deployment
kubectl get deployments -A
# 查看 kube-system 名称空间的 Deployment
kubectl get deployments -n kube-system
```

### 显示有关资源的详细信息

```sh
# kubectl describe 资源类型 资源名称
# 分析某一个pod的信息
kubectl describe pod xxx
```

### 查看pod中的容器的打印日志

```sh
# kubectl logs Pod名称
# 查看名称为nginx-pod-XXXXXXX的Pod内的容器打印的日志
# 本案例中的 nginx-pod 没有输出日志，所以您看到的结果是空的
kubectl logs -f nginx-pod-XXXXXXX
```

### 在pod中的容器环境内执行命令

```sh
# kubectl exec Pod名称 操作命令
# 在名称为nginx-pod-xxxxxx的Pod中运行bash
kubectl exec -it nginx-pod-xxxxxx /bin/bash
```

### 创建一个service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service	#Service 的名称
  labels:     	#Service 自己的标签
    app: nginx	#为该 Service 设置 key 为 app，value 为 nginx 的标签
spec:	    #这是关于该 Service 的定义，描述了 Service 如何选择 Pod，如何被访问
  selector:	    #标签选择器
    app: nginx	#选择包含标签 app:nginx 的 Pod
  ports:
  - name: nginx-port	#端口的名字
    protocol: TCP	    #协议类型 TCP/UDP
    port: 80	        #集群内的其他容器组可通过 80 端口访问 Service
    nodePort: 32600   #通过任意节点的 32600 端口访问 Service
    targetPort: 80	#将请求转发到匹配 Pod 的 80 端口
  type: NodePort	#Serive的类型，ClusterIP/NodePort/LoaderBalancer
```

```sh
# 执行命令
kubectl apply -f nginx-service.yaml
# 查看结果
kubectl get services -o wide
```

### 修改副本数

```sh
vim nginx-deployment.yaml
```

==将replicas改成4==

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

```sh
kubectl apply -f nginx-deployment.yaml
```

### 滚动更新nginx-deployment

滚动更新是k8s的一个机制，只需要修改一下containers下的image版本号。
