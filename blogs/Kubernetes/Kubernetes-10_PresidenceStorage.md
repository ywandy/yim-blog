---
title: K8S-10_为Pod使用持久性存储（Presidence&Storage）
tags: 
  - Kubernetes
date: 2023-10-26 09:00:00
categories:	
  - Kubernetes
---

## 卷

Container 中的文件在磁盘上是临时存放的，这给 Container 中运行的较重要的应用 程序带来一些问题。一是当容器崩溃时文件丢失，kubelet 会重新启动容器， 但容器会以干净的状态重启。 

二是在同一 `Pod` 中运行多个容器并共享文件时出现。 Kubernetes [卷（Volume）](https://kubernetes.io/zh/docs/concepts/storage/volumes/) 这一抽象概念能够解决这两个问题。

Docker 也有 [卷（Volume）](https://docs.docker.com/storage/) 的概念，但对它只有少量且松散的管理。 Docker 卷是磁盘上或者另外一个容器内的一个目录。 Docker 提供卷驱动程序，但是其功能非常有限。

Kubernetes 支持很多类型的卷。 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 可以同时使用任意数目的卷类型。 临时卷类型的生命周期与 Pod 相同，但持久卷可以比 Pod 的存活期长。 因此，卷的存在时间会超出 Pod 中运行的所有容器，并且在容器重新启动时数据也会得到保留。 当 Pod 不再存在时，卷也将不再存在。

卷的核心是包含一些数据的一个目录，Pod 中的容器可以访问该目录。 所采用的特定的卷类型将决定该目录如何形成的、使用何种介质保存数据以及目录中存放 的内容。

使用卷时, 在 `.spec.volumes` 字段中设置为 Pod 提供的卷，并在 `.spec.containers[*].volumeMounts` 字段中声明卷在容器中的挂载位置。

## Pod使用存储的几种方式

![pod存储的方式](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202311221516377.png)

## Pod直接使用存储

Kubernetes支持非常多的卷类型，参考官方文档：

https://kubernetes.io/zh/docs/concepts/storage/volumes/

本文中会介绍其中几种：

### emptyDir

当 Pod 分派到某个 Node 上时，`emptyDir` 卷会被创建，并且在 Pod 在该节点上运行期间，卷一直存在。 就像其名称表示的那样，卷最初是空的。 尽管 Pod 中的容器挂载 `emptyDir` 卷的路径可能相同也可能不同，这些容器都可以读写 `emptyDir` 卷中相同的文件。 当 Pod 因为某些原因被从节点上删除时，`emptyDir` 卷中的数据也会被永久删除。

说明：容器崩溃并不会导致 Pod 被从节点上移除，因此容器崩溃期间 `emptyDir` 卷中的数据是安全的。

`emptyDir` 的一些用途：

- 缓存空间，例如基于磁盘的归并排序。
- 为耗时较长的计算任务提供检查点，以便任务能方便地从崩溃前状态恢复执行。
- 在 Web 服务器容器服务数据时，保存内容管理器容器获取的文件。

`emptyDir` 卷存储在该节点所使用的介质上；这里的介质可以是磁盘或 SSD 或网络存储。但是，你可以将 `emptyDir.medium` 字段设置为 `"Memory"`，以告诉 Kubernetes 为你挂载 tmpfs（基于 RAM 的文件系统）。 虽然 tmpfs 速度非常快，但是要注意它与磁盘不同。 tmpfs 在节点重启时会被清除，并且你所写入的所有文件都会计入容器的内存消耗，受容器内存限制约束。

```shell
[root@clientvm ~]# cat emptyDir-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    name: test-container
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}


#pod已经在worker1上运行：
[root@clientvm ~]# kubectl get pod -n mytest -o wide
NAME      READY   STATUS    RESTARTS   AGE     IP            NODE                  NOMINATED NODE   READINESS GATES
busybox   1/1     Running   43         5d      10.244.2.55   worker2.example.com   <none>           <none>
dns       1/1     Running   42         4d22h   10.244.1.40   worker1.example.com   <none>           <none>
test-pd   1/1     Running   0          3m24s   10.244.1.46   worker1.example.com   <none>           <none>


[root@clientvm ~]# kubectl describe pod test-pd -n mytest
......
Volumes:
  cache-volume:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
    
#验证：
[root@clientvm ~]# kubectl exec test-pd -n mytest -it -- bash
root@test-pd:/# cd /cache/
root@test-pd:/cache# touch aaa bbb ccc ddd
root@test-pd:/cache# exit
exit

#worker1上，查找容器名字
[root@worker1 ~]# docker ps | grep test-container
8c61c355f7bb        bc9a0695f571                                        "/docker-entrypoint.…"   4 minutes ago       Up 4 minutes                            k8s_test-container_test-pd_mytest_a75956a3-a140-4582-b02c-03f25163d3d2_0

#查找dir
[root@worker1 ~]# docker inspect 8c61c355f7bb | grep -A3 -B3 '/cache'
......
        "Mounts": [
            {
                "Type": "bind",
                "Source": "/var/lib/kubelet/pods/a75956a3-a140-4582-b02c-03f25163d3d2/volumes/kubernetes.io~empty-dir/cache-volume",
                "Destination": "/cache",
                "Mode": "Z",
                "RW": true,
                "Propagation": "rprivate"
[root@worker1 ~]#
[root@worker1 ~]# ll "/var/lib/kubelet/pods/a75956a3-a140-4582-b02c-03f25163d3d2/volumes/kubernetes.io~empty-dir/cache-volume"
total 0
-rw-r--r--. 1 root root 0 Dec 14 12:24 aaa
-rw-r--r--. 1 root root 0 Dec 14 12:24 bbb
-rw-r--r--. 1 root root 0 Dec 14 12:24 ccc
-rw-r--r--. 1 root root 0 Dec 14 12:24 ddd

#删除Pod后，发现dir也随之删除
[root@clientvm ~]# kubectl delete -f emptyDir-pod.yaml -n mytest
pod "test-pd" deleted

[root@worker1 ~]# ll "/var/lib/kubelet/pods/a75956a3-a140-4582-b02c-03f25163d3d2/volumes/kubernetes.io~empty-dir/cache-volume"
ls: cannot access /var/lib/kubelet/pods/a75956a3-a140-4582-b02c-03f25163d3d2/volumes/kubernetes.io~empty-dir/cache-volume: No such file or directory

#限制emptyDir容量
[root@clientvm ~]# cat empty-Dir-limit-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    name: test-container
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir:
      sizeLimit: 2Gi
      
[root@clientvm ~]# kubectl apply -f empty-Dir-limit-pod.yaml -n mytest
pod/test-pd created

[root@clientvm ~]# kubectl describe pod -n mytest test-pd
......
Volumes:
  cache-volume:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  2Gi
```

### hostPath

`hostPath` 卷能将主机节点文件系统上的文件或目录挂载到你的 Pod 中。

`hostPath` 的一些用法有：

- 运行一个需要访问 Docker 内部机制的容器；可使用 `hostPath` 挂载 `/var/lib/docker` 路径。
- 允许 Pod 指定给定的 `hostPath` 在运行 Pod 之前是否应该存在，是否应该创建以及应该以什么方式存在。

除了必需的 `path` 属性之外，用户可以选择性地为 `hostPath` 卷指定 `type`。

| 取值              | 行为                                                         |
| ----------------- | ------------------------------------------------------------ |
|                   | 空字符串（默认）用于向后兼容，这意味着在安装 hostPath 卷之前不会执行任何检查。 |
| DirectoryOrCreate | 如果在给定路径上什么都不存在，那么将根据需要创建空目录，权限设置为 0755，具有与 kubelet 相同的组和属主信息。 |
| Directory         | 在给定路径上必须存在的目录。                                 |
| FileOrCreate      | 如果在给定路径上什么都不存在，那么将在那里根据需要创建空文件，权限设置为 0644，具有与 kubelet 相同的组和所有权。 |
| File              | 在给定路径上必须存在的文件。                                 |
| Socket            | 在给定路径上必须存在的 UNIX 套接字。                         |
| CharDevice        | 在给定路径上必须存在的字符设备。                             |
| BlockDevice       | 在给定路径上必须存在的块设备。                               |

```shell
#examples
[root@clientvm ~]# cat hostPath-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /data
      # Optional
      type: DirectoryOrCreate
      
[root@clientvm ~]# kubectl apply -f hostPath-pod.yaml
pod/test-pd created

[root@clientvm ~]# kubectl get pod -o wide
NAME                                              READY   STATUS      RESTARTS   AGE     IP            NODE                  NOMINATED NODE   READINESS GATES
test-pd                                           1/1     Running     0          99s     10.244.1.49   worker1.example.com   <none>           <none>

[root@clientvm ~]# kubectl exec test-pd -it -- bash
root@test-pd:/# cd /test-pd/
root@test-pd:/test-pd# touch aaa bbb ccc ddd
root@test-pd:/test-pd# exit
exit

#到worker1上验证
[root@worker1 ~]# cd /data/
[root@worker1 data]# ll
total 0
-rw-r--r--. 1 root root 0 Dec 14 12:47 aaa
-rw-r--r--. 1 root root 0 Dec 14 12:47 bbb
-rw-r--r--. 1 root root 0 Dec 14 12:47 ccc
-rw-r--r--. 1 root root 0 Dec 14 12:47 ddd

#删除Pod后再查看
[root@clientvm ~]# kubectl delete -f hostPath-pod.yaml
pod "test-pd" deleted

[root@worker1 /]# ll /data/
total 0
-rw-r--r--. 1 root root 0 Dec 14 12:47 aaa
-rw-r--r--. 1 root root 0 Dec 14 12:47 bbb
-rw-r--r--. 1 root root 0 Dec 14 12:47 ccc
-rw-r--r--. 1 root root 0 Dec 14 12:47 ddd
```

### NFS

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: test-container
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: test-volume
  volumes:
  - name: test-volume
    nfs:
      server: clientvm.example.com
      path: /data
```

```shell
[root@clientvm ~]# kubectl apply -f  pod-nfs.yaml -n mytest
[root@clientvm ~]# cd /data/
[root@clientvm data]# ls
[root@clientvm data]# echo AAAAAAAAAAAAAA >index.html

[root@master ~]# kubectl get pod -o wide -n mytest
NAME                               READY   STATUS    RESTARTS   AGE     IP               NODE                  NOMINATED NODE   READINESS GATES
......
test-pd                            1/1     Running   0          6m43s   10.244.71.214    worker2.example.com   <none>           <none>

[root@master ~]# curl 10.244.71.214
AAAAAAAAAAAAAA
```

## 使用持久卷 Persistent Volume

### 理论基础

持久卷（PersistentVolume，PV）是集群中的一块存储，可以由管理员事先供应，或者 使用[存储类（Storage Class）](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/)来动态供应。 持久卷是集群资源，就像节点也是集群资源一样。PV 持久卷和普通的 Volume 一样，也是使用 卷插件来实现的，只是它们拥有独立于任何使用 PV 的 Pod 的生命周期。 此 API 对象中记述了存储的实现细节，无论其背后是 NFS、iSCSI 还是特定于云平台的存储系统。

持久卷申明（PersistentVolumeClaim，PVC）表达的是用户对存储的请求。概念上与 Pod 类似。 Pod 会耗用节点资源，而 PVC 申领会耗用 PV 资源。Pod 可以请求特定数量的资源（CPU 和内存）；同样 PVC 申领也可以请求特定的大小和访问模式 （例如，可以要求 PV 卷能够以 ReadWriteOnce、ReadOnlyMany 或 ReadWriteMany 模式之一来挂载，参见[访问模式](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#access-modes)）。

### 回收策略

当用户不再使用其存储卷时，他们可以从 API 中将 PVC 对象删除，从而允许 该资源被回收再利用。PersistentVolume 对象的回收策略告诉集群，当其被 从申领中释放时如何处理该数据卷。 目前，数据卷可以被 Retained（保留）、 Deleted（删除）及Recycle(此策略已废弃)。

#### Retained（保留）

回收策略 `Retain` 使得用户可以手动回收资源。当 PersistentVolumeClaim 对象 被删除时，PersistentVolume 卷仍然存在，对应的数据卷被视为"已释放（released）"。 由于卷上仍然存在这前一申领人的数据，该卷还不能用于其他申领。 管理员可以通过下面的步骤来手动回收该卷：

1. 删除 PersistentVolume 对象。与之相关的、位于外部基础设施中的存储资产 （例如 AWS EBS、GCE PD、Azure Disk 或 Cinder 卷）在 PV 删除之后仍然存在。
2. 根据情况，手动清除所关联的存储资产上的数据。
3. 手动删除所关联的存储资产；如果你希望重用该存储资产，可以基于存储资产的 定义创建新的 PersistentVolume 卷对象。

#### 删除（Delete）

对于支持 `Delete` 回收策略的卷插件，删除动作会将 PersistentVolume 对象从 Kubernetes 中移除，同时也会从外部基础设施（如 AWS EBS、GCE PD、Azure Disk 或 Cinder 卷）中移除所关联的存储资产。 动态供应的卷会继承[其 StorageClass 中设置的回收策略](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#reclaim-policy)，该策略默认 为 `Delete`。

### 访问模式

PersistentVolume 卷可以用资源提供者所支持的任何方式挂载到宿主系统上。 如下表所示，提供者（驱动）的能力不同，每个 PV 卷的访问模式都会设置为 对应卷所支持的模式值。 例如，NFS 可以支持多个读写客户，但是某个特定的 NFS PV 卷可能在服务器 上以只读的方式导出。每个 PV 卷都会获得自身的访问模式集合，描述的是 特定 PV 卷的能力。

访问模式有：

- ReadWriteOnce -- 卷可以被一个节点以读写方式挂载；
- ReadOnlyMany -- 卷可以被多个节点以只读方式挂载；
- ReadWriteMany -- 卷可以被多个节点以读写方式挂载。

在命令行接口（CLI）中，访问模式也使用以下缩写形式：

- RWO - ReadWriteOnce
- ROX - ReadOnlyMany
- RWX - ReadWriteMany

### 绑定规则

当 PVC 绑定 PV 时，需考虑以下参数来筛选当前集群内是否存在满足条件的 PV。如果需要将特定的PV绑定到特定的PVC，则需要指定volumeName和claimRef

| 参数         | 描述                                                         |
| ------------ | ------------------------------------------------------------ |
| VolumeMode   | 主要定义 volume 是文件系统（FileSystem）类型还是块（Block）类型，PV 与 PVC 的 VolumeMode 标签必须相匹配。 |
| Storageclass | PV 与 PVC 的 storageclass 类名必须相同（或同时为空）。       |
| AccessMode   | 主要定义 volume 的访问模式，PV 与 PVC 的 AccessMode 必须相同。 |
| Size         | 主要定义 volume 的存储容量，PVC 中声明的容量必须小于等于 PV，如果存在多个满足条件的 PV，则选择最小的 PV 与 PVC 绑定。 |

### 范例

创建基于NFS的PV

```shell
[root@clientvm ~]# cat nfs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv1
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 192.168.241.128
    path: "/data"
    
[root@clientvm ~]# kubectl apply -f nfs-pv.yaml
persistentvolume/nfs-pv1 created
[root@clientvm ~]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
nfs-pv1   100Mi      RWX            Recycle           Available                                   6s

#创建PVC
[root@clientvm ~]# cat pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc1
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 100Mi

[root@clientvm ~]# kubectl apply -f pvc.yaml
persistentvolumeclaim/nfs1 created

[root@clientvm ~]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   REASON   AGE
nfs-pv1   100Mi      RWX            Recycle           Bound    default/nfs-pvc1                           15s

[root@clientvm ~]# kubectl get pvc
NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc1   Bound    nfs-pv1   100Mi      RWX                           110s

#创建Pod，使用PVC
[root@clientvm ~]# cat pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: nfs-pvc1
  containers:
    - name: task-pv-container
      image: nginx
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage

[root@clientvm ~]# kubectl apply -f pod-with-pvc.yaml
pod/task-pv-pod created

[root@clientvm ~]# kubectl get pod
NAME                                              READY   STATUS      RESTARTS   AGE
......
task-pv-pod                                       1/1     Running     0          13s

#验证读写数据
[root@clientvm ~]# kubectl exec task-pv-pod -it -- bash
root@task-pv-pod:/# cd "/usr/share/nginx/html"
root@task-pv-pod:/usr/share/nginx/html# ls
root@task-pv-pod:/usr/share/nginx/html# echo AAAAAA >index.html
root@task-pv-pod:/usr/share/nginx/html# exit
exit

[root@clientvm ~]# ls /data/
index.html
[root@clientvm ~]# cat /data/index.html
AAAAAA

#删除PVC与Pod，PV重新可用
[root@clientvm ~]# kubectl delete -f pod-with-pvc.yaml
pod "task-pv-pod" deleted

[root@clientvm ~]# kubectl delete -f nfs-pvc.yaml
persistentvolumeclaim "nfs-pvc1" deleted

[root@clientvm ~]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
nfs-pv1   100Mi      RWX            Recycle          Available                                   2m11s

```

## 使用StorageClass

### 创建基于NFS的StorageClass

参考： https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

```shell
[root@clientvm ~]# kubectl create namespace nfs-sc
[root@clientvm ~]# kubectl apply -f /resources/yaml/nfs-storage-class/rbac.yaml
[root@clientvm ~]# kubectl apply -f /resources/yaml/nfs-storage-class/deployment.yaml
[root@clientvm ~]# kubectl apply -f /resources/yaml/nfs-storage-class/class.yaml
[root@clientvm ~]# kubectl get sc
NAME                  PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage   fuseim.pri/ifs   Delete          Immediate           false                  6m55s

```

### 创建PVC

```shell
[root@clientvm ~]# cat sc-pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: managed-nfs-storage
  resources:
    requests:
      storage: 100Mi
      
[root@clientvm ~]# kubectl apply -f sc-pvc.yaml
persistentvolumeclaim/test-pvc created

[root@clientvm ~]# kubectl get pvc
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
test-pvc   Bound    pvc-17482f72-47c7-4409-9810-ce6ac1178a02   100Mi      RWX            managed-nfs-storage   9s
```

### 创建Pod

```shell
[root@clientvm ~]# cat sc-pvc-pod.yaml
kind: Pod
apiVersion: v1
metadata:
  name: test-pod-sc
spec:
  containers:
  - name: test-pod-sc
    image: nginx
    imagePullPolicy: IfNotPresent
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/usr/share/nginx/html/"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-pvc
        
[root@clientvm ~]# kubectl apply -f sc-pvc-pod.yaml
pod/test-pod-sc created
[root@clientvm ~]# kubectl get pod
NAME                                              READY   STATUS      RESTARTS   AGE
......
test-pod-sc                                       1/1     Running     0          6s

[root@clientvm ~]# kubectl exec  test-pod-sc -it -- bash
root@test-pod-sc:/# df -h
Filesystem                                                                       Size  Used Avail Use% Mounted on
......
192.168.241.128:/data/default-test-pvc-pvc-17482f72-47c7-4409-9810-ce6ac1178a02  8.0G  3.9G  4.2G  48% /usr/share/nginx/html
......
root@test-pod-sc:/# echo AAAAAAAAAAAA>/usr/share/nginx/html/index.html
root@test-pod-sc:/# exit
exit

[root@clientvm ~]# ll /data/default-test-pvc-pvc-17482f72-47c7-4409-9810-ce6ac1178a02/
total 4
-rw-r--r--. 1 root root 13 Dec 14 17:12 index.html
[root@clientvm ~]# cat /data/default-test-pvc-pvc-17482f72-47c7-4409-9810-ce6ac1178a02/index.html
AAAAAAAAAAAA

#删除PVC与Pod后，NFS上的数据也被删除
[root@clientvm ~]# kubectl delete -f sc-pvc-pod.yaml
pod "test-pod-sc" deleted
[root@clientvm ~]# kubectl delete -f sc-pvc.yaml
persistentvolumeclaim "test-pvc" deleted
[root@clientvm ~]# ll /data/
total 0
```

### 设置默认StorageClass

```shell
kubectl patch storageclass managed-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

[root@clientvm ~]# kubectl annotate storageclasses.storage.k8s.io managed-nfs-storage storageclass.kubernetes.io/is-default-class=true
storageclass.storage.k8s.io/managed-nfs-storage annotated
[root@clientvm ~]#
[root@clientvm ~]# kubectl get sc
NAME                            PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage (default)   fuseim.pri/ifs   Delete          Immediate           false                  2m41s
```

