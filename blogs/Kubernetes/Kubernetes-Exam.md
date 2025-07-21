---
title: K8S-Exam
tags: 
  - Kubernetes
date: 2023-11-21 09:00:00
categories:	
  - Kubernetes
---

## 1.RBAC（4/25%）

创建一个名为 deployment-clusterrole 且仅允许创建以下资源类型的新 ClusterRole、Deployment、StatefulSet、DaemonSet，在现有的 namespace app-team1 中创建一个名为 cicd-token 的新 ServiceAccount。

限于 namespace app-team1，将新的 ClusterRole deployment-clusterrole 绑定到新的 ServiceAccount cicd-token

```shell
# 创建clusterrole
kubectl clusterrole --name=deployment-clusterrole --verb=create --resource=deployments.app,statefulsets.app,daemonsets.app

# 创建serviceaccount
kubectl create serviceaccount cicd-token -n app-team1

# 创建rolebinding，不能创建 clusterrolebinding 因为限定在 app-team1 namespace 中
kubectl create rolebinding my-binding --clusterrole=deployment-clusterrole --serviceaccount
```

参考：https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/

---

## 2.pod（4/25%）

将一个名为 ek8s-node-1 的节点设置为不可用并将其上的 pod 重新调度

```shell
#A.设置为维护状态 cordon
kubectl cordon ek8s-node-1

#B.驱逐已存在的 pod，ignore daemonsets & delete empty-dir
kubectl drain node ek8s-node-1 --force --ignore-daemonsets --delete-empty-dir
```

---

## 3.版本升级（7/25%）

在master升级kubeadm,kubelet,kubectl到指定版本，但不要升级etcd，container manager，CNI插件，DNS service以及其他插件

```shell
#A.设置维护状态
#预设置环境
kubectl config use-context mk8s
kubectl get nodes
kubectl cordon k8s-master
#驱逐节点
kubectl drain k8s-master --delete-emptydir-data --ignore-daemonsets --force

#B.
#1 取消 hold 版本
#2 设置升级的版本 
#3 hold 住版本
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm='1.19.0-00' && \
apt-mark hold kubeadm

sudo kubeadm upgrade apply v1.19.0 --etcd-upgrade=false

#C.update kubectl & kubelet
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet='1.24.1-00' kubectl='1.24.1-00' && \
apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

#D.cancel cordon
#解除节点保护 只在 master 节点执行
kubectl uncordon k8s-master
```

参考：https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

---

## 4.备份Etcd（7/25%）

首先，为运行在 https://127.0.0.1:2379 上的现有 etcd 实例创建快照并将快照保存至 /data/bucket/etcd-snapshot.db。然后还原位于/srv/data/etcd-snaphot-previous.db 的现有先前快照 

 提示: 为给定实例创建快照预计在几秒内完成。如果该操作似乎挂起，则命令可能有问题。 用 ctrl+c 来取消操作，然后重试。 

 提供了以下 TLS 证书和密钥，以通过 etcdctl 连接到服务器

- CA证书:/opt/KUIN00601/ca.crt

- 客户端证书: /opt/KUIN00601/etcd-client.crt

- 客户端密钥:/opt/KUIN00601/etcd-client.key 

```shell
# 备份命令行格式
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file> \
  snapshot save <backup-file-location>

ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/opt/KUIN00601/ca.crt
 --cert=/opt/KUIN00601/etcd-client.crt --key=/opt/KUIN00601/etcd-client.key \ snapshot save /data/bucket/etcd-snapshot.db

# 还原命令行格式
ETCDCTL_API=3 etcdctl --endpoints 127.0.0.1:2379 snapshot restore --data-dir <data-dir-location> snapshotdb

rm-rf /var/lib/etcd 
ETCDCTL_APT=3 etcdctl --endpoints 127.0.0.1:2379 snapshot restore --data-dir /var/lib/etcd/ /srv/data/etcd-snaphot-previous.db --skip-hash-check

ETCDCTL_API=3 etcdctl --endpoint=127.0.0.1 snapshot restore --data-dir=/var/lib/etcd xxx --skip-hash-check
```

参考：https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/configure-upgrade-etcd/

---

## 5.Networkpolicy（7/20%）

创建一个名为 allow-port-from-namespace 的新 NetworkPolicy，以允许现有 namespace internal 中的 Pods 连接到同一 namespace 中其他 Pods 的端口 8080。

确保新的 NetworkPolicy

- 不允许对没有在监听端口8080的pods的访问 

- 不允许不来自 namespace internal 的 pods 的访问 

- (注意题意为限制 pod 入站规则，入站只允许访问同 ns 中的 8080端口，未限制出站)

```shell
# 检查 ns 标签
kubectl get ns internal --show-labels
# 如果没有，先打上标签
kubectl label ns internal project=internal

# 创建 networkpolicy （到官网 networkpolicy 复制 yaml）
vim 05-networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-from-namespace
  namespace: internal 
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              project: internal
      ports:
        - protocol: TCP
          port: 8080

kubectl apply -f 05-networkpolicy.yaml

# 验证：
kubectl get networkpolicy -n internal
```

参考：https://kubernetes.io/zh-cn/docs/concepts/services-networking/network-policies/

---

## 6.Service（7/20%）

重新配置一个已经存在的控制器（front-end），给已经存在的容器（nginx）暴露一个别名为http的80/tcp端口；创建一个名为front-end-svc的service暴露容器中的http的端口；把上述创建好的service放到已经被调度的node中

```shell
# 切换集群
kubectl config use-context k8s

# 编辑
kubectl edit deployment front-end
```

```yaml
# 添加以下行，注意缩进
ports:
     - name: http
       containerPort: 80
       protocol: TCP
```

```shell
# 执行一下
kubectl expose deployment front-end --target-port=http --name=front-end-svc --type=NodePort
# 验证
kubectl get svc
```

参考：https://kubernetes.io/zh-cn/docs/tutorials/services/connect-applications-service/

---

## 7.Ingress（7/20%）

创建一个名为nginx的ingress，条件如下

- name：pong
- namespace：ing-internal
- 暴露一个hi的service的5678端口，能访问到/hi的这个url

验证：curl -kL <ip>/hi，返回的是hi

```shell
# 创建ingress的yaml
vim 7.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pong
  namespace: ing-internal
spec:
  rules:
  - http:
      paths:
      - path: /hi
        pathType: Prefix
        backend:
          service:
            name: hi
            port:
              number: 5678

# 执行以下
kubectl apply -f 7.yaml

# 验证
kubetcl get pods -A -owide | grep ingress
curl -kl <ingress ip>/hi
```

---

## 8.动态扩容pod（4/15%）

扩容给定的pod

```shell
# 切换集群名称
kubectl config use-context k8s

# 水平扩容至给定副本
kubectl scale deployment.app presentation --replicas=3
```

---

## 9.Pod指定节点部署（4/15%）

部署指定节点

- name：nginx-kusc00401
- image: nginx
- NodeSelector: disk=spinning

```shell
# 查看节点 label
kubectl get node --show-labels

# 给节点打 label
kubectl label nodes k8s-node2 disk=spining

# 创建 pod 
vim 9.yaml

apiVersion: v1
kind: Pod
metadata:
  name: nginx-kusc00401
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disk: spinning

# 执行一下
kubectl apply -f 9.yaml
# 验证
kubectl get pods -owide | grep nginx
```

参考：https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/assign-pods-nodes/

---

## 10.检查node的健康状态（4/15%）

检查ready状态的node，不包含污点为NoSchedule的，把数量写到指定文件中

```shell
# 切换集群
kubectl config use-context k8s

# 记录ready的总数
kubectl get node | grep -i ready 

# 记录NoSchedule的总数
kubectl describe node | grep Taint | grep NoSchedule

# 输出到文件中
echo 2 >> /opt/KUSC00402/kusc00402.txt
```

---

## 11.一个pod封装多个容器（4/15%）

创建一个名称为kucc1的pod，包含一到四个镜像：nginx,redis,memcachedl,consul

```shell
kubectl config use-context k8s

vim 11.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kucc8
spec:
  containers:
  - name: nginx
    image: nginx
  - name: redis
    image: redis
  - name: memcached
    image: memcached
  - name: consul
    image: consul
```

```shell
kublectl apply -f 11.yaml
```

参考：https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/assign-pods-nodes/

---

## 12：持久化存储卷PersistentVolume（4/10%）

创建一个名为app-config的pv，容量是2G，访问模式是ReadWriteMany，用到的存储是宿主机的指定目录

```shell
kubectl config use-context hk8s

vim 12.yaml
```

12.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-config
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/srv/app-config"
```

```shell
kubectel apply -f 12.yaml
```

参考：https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-persistent-volume-storage/

---

## 13：PersistentVolumeClaims（7/10%）

创建一个pvc

- name: pv-volume
- Class: csi-hostpath-sc
- Capacity: 10Mi

创建一个新pod，挂载上述pvc

- name: web-server
- image: nginx
- Mount Path: /usr/share/nginx/html

访问模式是ReadWriteOnce

最后扩容pvc至70Mi

```shell
kubectl config use-context ok8s

vim 13.yaml
```

13.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-volume
spec:
  storageClassName: csi-hostpath-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
```

```shell
kubectl apply -f 13.yaml
```

13-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  volumes:
    - name: pv-volume
      persistentVolumeClaim:
        claimName: pv-volume
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-volume
```

```shell
kubectl apply -f 13-pod.yaml

# 扩容
kubectl edit pvc pv-volume
```

![扩容pvc](https://pic-new-1304161434.cos.ap-guangzhou.myqcloud.com/img/202403211712842.png)

---

## 14：监控pod日志（5/30%）

监控给定名称pod的日志，把指定参数的监控的日志写到指定目录下

```shell
kubectl config use-context k8s

kubectl logs foobar | grep unable-to-access-website > /opt/KUTR00101/foobar
```

---

## 15：SideCar代理（7/30%）

在已存在的容器中添加一个指定名称的sidecar容器，新的sidecar容器要执行以下命令

```sh
/bin/sh -c tail -n+1 -F /var/log/big-corp-app.log
```

并且挂载一个指定名称的卷，能访问到指定目录下的log文件

注意：

不要改变已存在的容器及已存在的目录，但是他们要能访问到指定的目录下的log文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox:1.28
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done      
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-agent
    image: registry.k8s.io/fluentd-gcp:1.30
    env:
    - name: FLUENTD_ARGS
      value: -c /etc/fluentd-config/fluentd.conf
    volumeMounts:
    - name: varlog
      mountPath: /var/log
    - name: config-volume
      mountPath: /etc/fluentd-config
  volumes:
  - name: varlog
    emptyDir: {}
  - name: config-volume
    configMap:
      name: fluentd-config
```



参考： https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/logging/

---

## 16：监控pod度量指标（5/30%）

监控指定名称的label，找出高cpu占用的pod，输出到指定文件中

```shell
kubectl config use-context k8s
```

```shell
kubectl top pod -A -l name=cpu-user
echo <podname> > /opt/KUTR00401.txt
```

---

## 17：集群故障排查（13/30%）

给定的node，状态是NotReady，请把他正常调度，并开机自启动

```shell
kubectl config use-context wk8s

# 查看哪个 node 不正常
kubectl get nodes
# 查看具体问题
k describe nodes k8s-node1
# ssh 连接
ssh k8s-node1
sudo i
systemctl status kubelet
systemctl start kubelet
systemctl enable kubelet
exit
exit
# master 节点查看是否恢复
kubectl get nodes
```

