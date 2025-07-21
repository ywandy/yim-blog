---
title: K8S-11_部署有状态服务（StatefulSet）
tags: 
  - Kubernetes
date: 2023-11-24 09:00:00
categories:	
  - Kubernetes
---

## 理论基础

StatefulSet 是用来管理有状态应用的工作负载 API 对象。

StatefulSet 用来管理某 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 集合的部署和扩缩， 并为这些 Pod 提供持久存储和持久标识符。

和 [Deployment](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/) 类似， StatefulSet 管理基于相同容器规约的一组 Pod。但和 Deployment 不同的是， StatefulSet 为它们的每个 Pod 维护了一个有粘性的 ID。这些 Pod 是基于相同的规约来创建的， 但是不能相互替换：无论怎么调度，每个 Pod 都有一个永久不变的 ID。

如果希望使用存储卷为工作负载提供持久存储，可以使用 StatefulSet 作为解决方案的一部分。 尽管 StatefulSet 中的单个 Pod 仍可能出现故障， 但持久的 Pod 标识符使得将现有卷与替换已失败 Pod 的新 Pod 相匹配变得更加容易。

StatefulSets 对于需要满足以下一个或多个需求的应用程序很有价值：

- 稳定的、唯一的网络标识符。
- 稳定的、持久的存储。
- 有序的、优雅的部署和缩放。
- 有序的、自动的滚动更新。

在上面描述中，“稳定的”意味着 Pod 调度或重调度的整个过程是有持久性的。 如果应用程序不需要任何稳定的标识符或有序的部署、删除或伸缩，则应该使用 由一组无状态的副本控制器提供的工作负载来部署应用程序，比如 [Deployment](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/) 或者 [ReplicaSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicaset/) 可能更适用于你的无状态应用部署需要。

## 有状态与无状态

有人将web应用中有无状态的情况，比着顾客逛商店的情景。

顾客：浏览器访问方；

商店：web服务器；

一次购买：一次http访问

我们知道，上一次顾客购买，并不代表顾客下一个小时一定会买（当然也不能代表不会）。 也就是说同一个顾客的不同购买之间的关系是不定的。所以说实在的，这种情况下，让商店保存所有的顾客购买的信息，等到下一次购买可以知道这个顾客以前购买 的内容代价非常大的。所以商店为了避免这个代价，索性就认为每次的购买都是一次独立的新的购买。浅台词：商店不区分对待老顾客和新过客。这就是无状态的。

但是，商店为了提高收益。她是想鼓励顾客购买的。所以告诉你，只要你在一个月内购买了５瓶以上的啤酒，就送你一个酒杯。

这种情况下有两种办法实现：

- A, 给顾客发放一个磁卡，里面放有顾客过去的购买信息。这样商店就可以知道了。这就是cookie.
- B, 给顾客发放一个唯一号码，号码制定的顾客的消费信息，存储在商店的服务器中。这就是session。

最后，商店可以全局的决定，是５瓶为送酒杯还是6瓶。这就是application。

其实，这些机制都是在无状态的传统购买过程中加入了一点东西，使整个过程变得有状态。Web应用就是这样的

## 创建StatefulSet

### 准备yaml

参考文档： https://kubernetes.io/zh/docs/concepts/workloads/controllers/statefulset/

```shell
[root@clientvm ~]# cat statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
  namespace: mytest
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: nginx
  name: mystatefulset
  namespace: mytest
spec:
  replicas: 3
  serviceName: nginx
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        imagePullPolicy: IfNotPresent
```

### 创建与查看

```shell
[root@clientvm ~]# kubectl apply -f statefulset.yaml
service/nginx created
statefulset.apps/mystatefulset created


[root@clientvm ~]# kubectl get all -n mytest
NAME                  READY   STATUS    RESTARTS   AGE
pod/mystatefulset-0   1/1     Running   0          94s
pod/mystatefulset-1   1/1     Running   0          92s
pod/mystatefulset-2   1/1     Running   0          90s

NAME                             READY   AGE
statefulset.apps/mystatefulset   3/3     94s
```

### 删除Pod，自动重建，Pod名字不变

```shell
[root@clientvm ~]# kubectl delete  pod -n mytest  mystatefulset-1
pod "mystatefulset-1" deleted
[root@clientvm ~]# kubectl get pod -n mytest
NAME              READY   STATUS    RESTARTS   AGE
mystatefulset-0   1/1     Running   0          2m18s
mystatefulset-1   1/1     Running   0          8s
mystatefulset-2   1/1     Running   0          2m14s
```

### 减少Pod

```shell
[root@clientvm ~]# kubectl scale statefulset -n mytest mystatefulset --replicas=2
statefulset.apps/mystatefulset scaled
[root@clientvm ~]# kubectl get pod -n mytest
NAME              READY   STATUS    RESTARTS   AGE
mystatefulset-0   1/1     Running   0          6m29s
mystatefulset-1   1/1     Running   0          4m19s
[root@clientvm ~]#
[root@clientvm ~]#
[root@clientvm ~]# kubectl scale statefulset -n mytest mystatefulset --replicas=1
statefulset.apps/mystatefulset scaled
[root@clientvm ~]# kubectl get pod -n mytest
NAME              READY   STATUS    RESTARTS   AGE
mystatefulset-0   1/1     Running   0          6m36s
```

### Service

statefulset通常需要与无头服务合用。无头服务无虚拟IP，任何访问请求都使用DNS方式指向后端Pod。

```shell
[root@clientvm yaml]# kubectl run busyboxcc --image=tutum/dnsutils -n mytest --image-pull-policy=IfNotPresent -- sleep 1000

[root@clientvm yaml]# kubectl exec -n mytest busyboxcc -it -- bash
root@busyboxcc:/# nslookup nginx.mytest.svc.example.com
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   nginx.mytest.svc.example.com
Address: 10.244.102.132
Name:   nginx.mytest.svc.example.com
Address: 10.244.71.199
Name:   nginx.mytest.svc.example.com
Address: 10.244.71.198

root@busyboxcc:/# nslookup mystatefulset-0.nginx.mytest.svc.example.com
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   mystatefulset-0.nginx.mytest.svc.example.com
Address: 10.244.71.198

root@busyboxcc:/# nslookup mystatefulset-1.nginx.mytest.svc.example.com
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   mystatefulset-1.nginx.mytest.svc.example.com
Address: 10.244.102.132


root@busyboxcc:/# nslookup mystatefulset-2.nginx.mytest.svc.example.com
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   mystatefulset-2.nginx.mytest.svc.example.com
Address: 10.244.71.199


root@busyboxcc:/# dig @10.96.0.10 -t a mystatefulset-0.nginx.mytest.svc.example.com
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;mystatefulset-0.nginx.mytest.svc.example.com. IN A

;; ANSWER SECTION:
mystatefulset-0.nginx.mytest.svc.example.com. 30 IN A 10.244.71.198
```

### 删除StatefulSet

```shell
[root@clientvm ~]# kubectl delete -f statefulset.yaml
service "nginx" deleted
statefulset.apps "mystatefulset" deleted
```

### 在StateFulset中使用StorageClass自动创建PV/PVC

```shell
[root@clientvm ~]# cat statefulset-sc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
  namespace: mytest
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: nginx
  name: mystatefulset
  namespace: mytest
spec:
  replicas: 3
  serviceName: nginx
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        imagePullPolicy: IfNotPresent
## volume
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
## StorageClass
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "managed-nfs-storage"
      resources:
        requests:
          storage: 100Mi
          
[root@clientvm ~]# kubectl get pod -n mytest
NAME              READY   STATUS    RESTARTS   AGE
......
mystatefulset-0   1/1     Running   0          15s
mystatefulset-1   1/1     Running   0          12s
mystatefulset-2   1/1     Running   0          7s
[root@clientvm ~]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                        STORAGECLASS          REASON   AGE
nfs-pv1                                    100Mi      RWX            Recycle          Available                                                               102m
pvc-85799af8-a9de-46fd-a268-b15edefb9eb5   100Mi      RWO            Delete           Bound       mytest/www-mystatefulset-2   managed-nfs-storage            13s
pvc-b3529c5a-cb07-4823-87be-9c4961988c2b   100Mi      RWO            Delete           Bound       mytest/www-mystatefulset-1   managed-nfs-storage            18s
pvc-c8faafe4-50a7-4ff4-acd7-9c45f3b5d4de   100Mi      RWO            Delete           Bound       mytest/www-mystatefulset-0   managed-nfs-storage            21s
```

