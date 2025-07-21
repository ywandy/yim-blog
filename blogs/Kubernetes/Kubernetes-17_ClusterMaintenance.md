---
title: K8S-17_集群维护（ClusterMaintenance）
tags: 
  - Kubernetes
date: 2023-11-27 09:00:00
categories:	
  - Kubernetes
---

## 事件查看

```sh
[root@clientvm ~]# kubectl describe pod nginx2 
[root@clientvm ~]# kubectl logs nginx2
```

## 节点维护

### 禁止调度

```sh
#cordon 停止调度
#影响最小，只会将node调为SchedulingDisabled，之后再发创建pod，不会被调度到该节点，旧有的pod不会受到影响，仍正常对外提供服务
[root@clientvm ~]# kubectl cordon worker1.example.com
node/worker1.example.com cordoned
[root@clientvm ~]# kubectl get node
NAME                  STATUS                     ROLES    AGE   VERSION
master.example.com    Ready                      master   19d   v1.19.4
worker1.example.com   Ready,SchedulingDisabled   <none>   18d   v1.19.4
worker2.example.com   Ready                      <none>   18d   v1.19.4
```

### 驱逐Pod

```sh
#首先，驱逐node上的pod，其他节点重新创建，接着，将节点调为SchedulingDisabled
[root@clientvm ~]# kubectl drain worker1.example.com --ignore-daemonsets
node/worker1.example.com already cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/kube-proxy-c9ghq
node/worker1.example.com drained
```

### 重新开始调度

```sh
[root@clientvm ~]# kubectl uncordon worker1.example.com
node/worker1.example.com uncordoned
[root@clientvm ~]#
[root@clientvm ~]# kubectl get node
NAME                  STATUS   ROLES    AGE   VERSION
master.example.com    Ready    master   19d   v1.19.4
worker1.example.com   Ready    <none>   18d   v1.19.4
worker2.example.com   Ready    <none>   18d   v1.19.4
```

### 安装Metrics Server

Metrics Server是Kubernetes内置自动缩放管道的可扩展，高效的容器资源指标来源。

Metrics Server从Kubelet收集资源指标，并通过Metrics API在Kubernetes apiserver中公开它们，以供Horizontal Pod Autoscaler使用。 还可以通过kubectl top访问Metrics API，从而更容易调试自动缩放。

Metrics Server提供：

- 适用于大多数群集的Deployment
- 可扩展支持多达5,000个节点集群
- 资源效率：Metrics Server使用1m核心CPU和每个节点3 MB内存

#### 安装参考官方文档

https://github.com/kubernetes-sigs/metrics-server

```sh
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

#### 下载components.yaml

注意添加--kubelet-insecure-tls 以忽略证书问题。

```sh
[root@clientvm ~]# kubectl apply -f /resources/yaml/metrics-server-components.yaml
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
```

#### 查看

```sh
[root@clientvm ~]# kubectl top node
NAME                  CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
master.example.com    119m         2%     1417Mi          51%
worker1.example.com   24m          0%     953Mi           20%
worker2.example.com   1079m        26%    1079Mi          23%

[root@clientvm ~]# kubectl top pod
NAME                     CPU(cores)   MEMORY(bytes)
nginx-taint              0m           1Mi
nginx2                   0m           1Mi
readiness-exec           2m           0Mi
test-pod-secret-volume   0m           1Mi
with-node-affinity       0m           1Mi
with-pod-affinity        0m           1Mi
```

## HPA(Horizontal Pod Autoscaler)

Pod 水平自动扩缩（Horizontal Pod Autoscaler） 可以基于 CPU 利用率自动扩缩 Deployment、ReplicaSet 和 StatefulSet 中的 Pod 数量。  Pod 自动扩缩不适用于无法扩缩的对象，比如 DaemonSet。

Pod 水平自动扩缩器的实现是一个控制回路，由控制器管理器(/etc/kubernetes/manifests/kube-controller-manager.yaml)的 `--horizontal-pod-autoscaler-sync-period` 参数指定周期（默认值为 15 秒），其他可用值：

--horizontal-pod-autoscaler-downscale-stabilization：即自从上次缩容执行结束后，多久可以再次执行缩容，默认时间是 5 分钟

`--horizontal-pod-autoscaler-initial-readiness-delay` 参数（默认为 30s）用于设置 Pod 准备时间， 在此时间内的 Pod 统统被认为未就绪。

`--horizontal-pod-autoscaler-cpu-initialization-period` 参数（默认为5分钟） 用于设置 Pod 的初始化时间， 在此时间内的 Pod，CPU 资源度量值将不会被采纳。

对于按 Pod 统计的资源指标（如 CPU），控制器从资源指标 API 中获取每一个 HorizontalPodAutoscaler 指定的 Pod 的度量值，如果设置了目标使用率， 控制器获取每个 Pod 中的容器资源使用情况，并计算资源使用率。

注：自 Kubernetes 1.11 起，从 Heapster 获取指标特性已废弃。

#### HPA默认行为：

```sh
kubectl explain HorizontalPodAutoscaler.spec.behavior
```

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Percent
      value: 100
      periodSeconds: 15
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 100
      periodSeconds: 15
    - type: Pods
      value: 4
      periodSeconds: 15
    selectPolicy: Max
```

#### HPA实验

##### 创建Deployment & Service

```sh
[root@clientvm ~]# cat hpa-example.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: pilchard/hpa-example
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m

---

apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
    
[root@clientvm ~]# kubectl apply -f hpa-example.yaml
deployment.apps/php-apache created
service/php-apache created

[root@master manifests]# curl 10.105.137.102
OK!
```

##### 创建HPA

```sh
[root@clientvm ~]# kubectl autoscale deployment -h
Usage:
  kubectl autoscale (-f FILENAME | TYPE NAME | TYPE/NAME) [--min=MINPODS] --max=MAXPODS [--cpu-percent=CPU] [options]

Examples:
  # Auto scale a deployment "foo", with the number of pods between 2 and 10, no target CPU utilization specified so a
default autoscaling policy will be used:
  kubectl autoscale deployment foo --min=2 --max=10

  # Auto scale a replication controller "foo", with the number of pods between 1 and 5, target CPU utilization at 80%:
  kubectl autoscale rc foo --max=5 --cpu-percent=80

[root@clientvm ~]# kubectl autoscale deployment --max=10 php-apache --cpu-percent=30
horizontalpodautoscaler.autoscaling/php-apache autoscaled

[root@clientvm ~]# kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/30%    1         10        1          20s
```

##### 测试

在master上使用ab工具发送大量并发

```sh
[root@master manifests]# ab -n 1000 -c 200  http://10.105.137.102/
```

在clientvm节点上观察

```sh
[root@clientvm ~]# kubectl get pod -w
NAME                          READY   STATUS    RESTARTS   AGE
nginx2                        1/1     Running   0          4d23h
php-apache-69f8f79bfc-sbzhm   1/1     Running   0          8m48s
php-apache-69f8f79bfc-wdwwq   0/1     Pending   0          0s
php-apache-69f8f79bfc-wdwwq   0/1     Pending   0          0s
php-apache-69f8f79bfc-fkb7k   0/1     Pending   0          0s
php-apache-69f8f79bfc-92pv4   0/1     Pending   0          0s
php-apache-69f8f79bfc-fkb7k   0/1     Pending   0          0s
php-apache-69f8f79bfc-92pv4   0/1     Pending   0          0s
php-apache-69f8f79bfc-wdwwq   0/1     ContainerCreating   0          0s
php-apache-69f8f79bfc-fkb7k   0/1     ContainerCreating   0          0s
php-apache-69f8f79bfc-92pv4   0/1     ContainerCreating   0          0s
php-apache-69f8f79bfc-92pv4   1/1     Running             0          2s
php-apache-69f8f79bfc-fkb7k   1/1     Running             0          2s
php-apache-69f8f79bfc-wdwwq   1/1     Running             0          2s
php-apache-69f8f79bfc-fwb5x   0/1     Pending             0          0s
php-apache-69f8f79bfc-fwb5x   0/1     Pending             0          0s
php-apache-69f8f79bfc-s5624   0/1     Pending             0          0s
php-apache-69f8f79bfc-qkf7f   0/1     Pending             0          0s
php-apache-69f8f79bfc-s5624   0/1     Pending             0          0s
php-apache-69f8f79bfc-jbgtp   0/1     Pending             0          0s
php-apache-69f8f79bfc-qkf7f   0/1     Pending             0          0s
php-apache-69f8f79bfc-fwb5x   0/1     ContainerCreating   0          0s
php-apache-69f8f79bfc-jbgtp   0/1     Pending             0          0s
php-apache-69f8f79bfc-s5624   0/1     ContainerCreating   0          0s
php-apache-69f8f79bfc-qkf7f   0/1     ContainerCreating   0          0s
php-apache-69f8f79bfc-jbgtp   0/1     ContainerCreating   0          0s
php-apache-69f8f79bfc-qkf7f   1/1     Running             0          3s
php-apache-69f8f79bfc-jbgtp   1/1     Running             0          3s
php-apache-69f8f79bfc-fwb5x   1/1     Running             0          3s

[root@clientvm k8s]# kubectl get hpa
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   206%/30%   1         10        10         7m3s
```

#### HPA的算法

```text
期望副本数 = ceil[当前副本数 * (当前指标 / 期望指标)]
```

## etcd

### Backup Etcd

```sh
#在master节点上查找etcd 容器
[root@master manifests]# docker ps | grep etcd
44623e84772a        d4ca8726196c                                        "etcd --advertise-cl…"   6 days ago          Up 6 days                               k8s_etcd_etcd-master.example.com_kube-system_1511fba334ccb18c8972b0adfa135f94_0

#copy etcdctl命令到本机
[root@master manifests]# docker cp 44623e84772a:/usr/local/bin/etcdctl /usr/bin/
#或者使用kubectl exec登录容器copy指令
[root@master manifests]# kubectl exec etcd-master.example.com -n kube-system -it -- sh   
sh-5.1# cp /usr/local/bin/etcdctl /etc/kubernetes/pki/etcd/etcdctl
sh-5.1# exit
exit
[root@master manifests]# ls /etc/kubernetes/pki/etcd/
ca.crt  ca.key  etcdctl  healthcheck-client.crt  healthcheck-client.key  peer.crt  peer.key  server.crt  server.key

#根据 静态Pod etcd.yaml文件的内容指定ca相关证书备份etcd
[root@master manifests]# ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/peer.crt \
--key=/etc/kubernetes/pki/etcd/peer.key snapshot save /tmp/etcd.db

#
[root@master manifests]# ETCDCTL_API=3 etcdctl --write-out=table snapshot status /tmp/etcd.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 88cde7c7 |  1498047 |       1854 |     3.3 MB |
+----------+----------+------------+------------+
```

### Restore Etcd

1. Stop API Server：移动kube-apiserver.yaml到其他目录
2. Stop ETCD Server：移动etcd.yaml到其他目录
3. backup ETCD data: /var/lib/etcd/
4. Restore
5. start ETCD SERVER and API server： 移回kube-apiserver.yaml和etcd.yaml.old到/etc/kubernetes/manifests

```sh
[root@master manifests]# ETCDCTL_API=3 etcdctl snapshot restore \
/tmp/etcd.db --data-dir="/var/lib/etcd" 

[root@master manifests]# ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key endpoint health
https://127.0.0.1:2379 is healthy: successfully committed proposal: took = 11.884856ms

[root@clientvm ~]# kubectl get pod -n kube-system
NAME                                         READY   STATUS    RESTARTS   AGE
calico-kube-controllers-6dfcd885bf-dk4jj     1/1     Running   0          6d23h
calico-node-7nlkr                            1/1     Running   0          6d23h
calico-node-8xdqh                            1/1     Running   0          6d23h
calico-node-dskkk                            1/1     Running   0          6d23h
coredns-6d56c8448f-lsl8p                     1/1     Running   0          7d
coredns-6d56c8448f-t8t55                     1/1     Running   0          23h
etcd-master.example.com                      1/1     Running   1          7d
kube-apiserver-master.example.com            1/1     Running   0          7d
kube-controller-manager-master.example.com   1/1     Running   1          6d21h
kube-proxy-2ddnd                             1/1     Running   0          7d
kube-proxy-cjl2b                             1/1     Running   0          6d23h
kube-proxy-n5djk                             1/1     Running   0          6d23h
kube-scheduler-master.example.com            1/1     Running   1          6d21h
metrics-server-85b5d6b8fb-vmprh              1/1     Running   0          3h51m
```

## 集群升级

### 升级Master节点

#### 选择要升级的版本

```sh
[root@master ~]# yum list --showduplicates kubeadm
Installed Packages
kubeadm.x86_64                                                                   1.26.0-0                                                                    @local
Available Packages
kubeadm.x86_64                                                                   1.24.0-0                                                                    local
kubeadm.x86_64                                                                   1.24.3-0                                                                    local
kubeadm.x86_64                                                                   1.26.0-0                                                                    local
kubeadm.x86_64                                                                   1.26.1-0                                                                    local
kubeadm.x86_64                                                                   1.26.3-0                                                                    local
```

#### 升级kubeadm

```sh
[root@master ~]# yum install kubeadm-1.26.3-0 -y
```

#### 验证版本

```sh
[root@master ~]# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"26", GitVersion:"v1.26.3", GitCommit:"9e644106593f3f4aa98f8a84b23db5fa378900bd", GitTreeState:"clean", BuildDate:"2023-03-15T13:38:47Z", GoVersion:"go1.19.7", Compiler:"gc", Platform:"linux/amd64"}
```

#### 在master上执行升级计划

```sh
kubeadm upgrade plan
```

#### 在master执行升级命令

修改集群配置对应版本，然后执行升级

```sh
[root@master ~]# kubeadm upgrade apply v1.26.3
......
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.26.3". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

#### 设置不可调度，并驱逐控制节点上的Pod

```sh
[root@clientvm ~]# kubectl drain master.example.com --ignore-daemonsets
```

#### 升级kubelet和kubectl

```sh
yum install -y  kubelet-1.26.3-0 kubectl-1.26.3-0
```

#### 重启kubelet 服务

```sh
systemctl daemon-reload
systemctl restart kubelet
```

#### 恢复节点调度

```sh
[root@clientvm ~]# kubectl uncordon master.example.com
[[root@master ~]# kubectl get node
NAME                  STATUS   ROLES           AGE   VERSION
master.example.com    Ready    control-plane   66m   v1.26.3
worker1.example.com   Ready    <none>          64m   v1.26.0
worker2.example.com   Ready    <none>          64m   v1.26.0
```

### 逐个升级worker节点

#### 在worker节点上升级kubeadm

```sh
[root@worker1 ~]# yum install kubeadm-1.26.3-0
[root@worker1 ~]# kubeadm upgrade node
```

#### 将节点标记为不可调度并逐出工作负载，为维护做好准备

```sh
kubectl drain worker1.example.com --ignore-daemonsets
```

#### 升级kubelet和kubectl

```sh
yum install -y kubelet-1.26.3-0 kubectl-1.26.3-0
```

#### 重启 kubelet 服务

```sh
systemctl daemon-reload
systemctl restart kubelet
```

#### 恢复节点调度

```sh
kubectl uncordon worker1.example.com
[root@master ~]#  kubectl get node
NAME                  STATUS   ROLES           AGE   VERSION
master.example.com    Ready    control-plane   70m   v1.26.3
worker1.example.com   Ready    <none>          68m   v1.26.3
worker2.example.com   Ready    <none>          68m   v1.26.0
```

#### 验证集群状态，并逐一升级其他worker节点

```sh
[root@master ~]#  kubectl get node
NAME                  STATUS   ROLES           AGE   VERSION
master.example.com    Ready    control-plane   70m   v1.26.3
worker1.example.com   Ready    <none>          68m   v1.26.3
worker2.example.com   Ready    <none>          68m   v1.26.3
```

#### 升级完成后验证集群组件

```sh
[root@clientvm ~]# kubectl get pod -n kube-system
NAME                                         READY   STATUS    RESTARTS   AGE
calico-kube-controllers-6dfcd885bf-dk4jj     1/1     Running   0          7d1h
calico-node-7nlkr                            1/1     Running   0          7d1h
calico-node-8xdqh                            1/1     Running   0          7d1h
calico-node-dskkk                            1/1     Running   0          7d1h
coredns-6d56c8448f-h6wvr                     1/1     Running   0          6m41s
coredns-6d56c8448f-lsl8p                     1/1     Running   0          7d1h
etcd-master.example.com                      1/1     Running   0          6m57s
kube-apiserver-master.example.com            1/1     Running   0          6m42s
kube-controller-manager-master.example.com   1/1     Running   0          6m40s
kube-proxy-8422l                             1/1     Running   0          6m4s
kube-proxy-df9t9                             1/1     Running   0          5m55s
kube-proxy-vmm7j                             1/1     Running   0          5m25s
kube-scheduler-master.example.com            1/1     Running   0          6m38s
metrics-server-85b5d6b8fb-vmprh              1/1     Running   0          5h23m

[root@master ~]# kubectl version --output=yaml
clientVersion:
  buildDate: "2023-03-15T13:40:17Z"
  compiler: gc
  gitCommit: 9e644106593f3f4aa98f8a84b23db5fa378900bd
  gitTreeState: clean
  gitVersion: v1.26.3
  goVersion: go1.19.7
  major: "1"
  minor: "26"
  platform: linux/amd64
kustomizeVersion: v4.5.7
serverVersion:
  buildDate: "2023-03-15T13:33:12Z"
  compiler: gc
  gitCommit: 9e644106593f3f4aa98f8a84b23db5fa378900bd
  gitTreeState: clean
  gitVersion: v1.26.3
  goVersion: go1.19.7
  major: "1"
  minor: "26"
  platform: linux/amd64
```

