---
title: K8S-13_ConfigMap&Secret
tags: 
  - Kubernetes
date: 2023-11-24 09:00:00
categories:	
  - Kubernetes
---

## ConfigMap

### 基础理论

ConfigMap 是一种 API 对象，用来将非机密性的数据保存到健值对中。使用时， [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 可以将其用作环境变量、命令行参数或者存储卷中的配置文件。

ConfigMap 将您的环境配置信息和 [容器镜像](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-image) 解耦，便于应用配置的修改。

**注意：**ConfigMap 并不提供保密或者加密功能。 如果你想存储的数据是机密的，请使用 [Secret](https://kubernetes.io/zh/docs/concepts/configuration/secret/)， 或者使用其他第三方工具来保证你的数据的私密性，而不是用 ConfigMap。

使用 ConfigMap 来将你的配置数据和应用程序代码分开。

ConfigMap 在设计上不是用来保存大量数据的。在 ConfigMap 中保存的数据不可超过 1 MiB。如果你需要保存超出此尺寸限制的数据，你可能希望考虑挂载存储卷 或者使用独立的数据库或者文件服务。

### 创建ConfigMap

```shell
[root@clientvm ~]# kubectl create configmap -h
Usage:
  kubectl create configmap NAME [--from-file=[key=]source] [--from-literal=key1=value1] [--dry-run=server|client|none]
[options]
Examples:
  # Create a new configmap named my-config based on folder bar
  kubectl create configmap my-config --from-file=path/to/bar

  # Create a new configmap named my-config with specified keys instead of file basenames on disk
  kubectl create configmap my-config --from-file=key1=/path/to/bar/file1.txt --from-file=key2=/path/to/bar/file2.txt

  # Create a new configmap named my-config with key1=config1 and key2=config2
  kubectl create configmap my-config --from-literal=key1=config1 --from-literal=key2=config2

  # Create a new configmap named my-config from the key=value pairs in the file
  kubectl create configmap my-config --from-file=path/to/bar

  # Create a new configmap named my-config from an env file
  kubectl create configmap my-config --from-env-file=path/to/bar.env
  
[root@clientvm ~]# kubectl create configmap my-config --from-literal=db_name=wordpress --from-literal=db_user=redhat
```

### 使用configMap

#### 通过容器的环境变量使用ConfigMap

https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#define-a-container-environment-variable-with-data-from-a-single-configmap

```shell
[root@clientvm ~]# cat configmap-env-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-env
spec:
  containers:
    - name: test-container
      image: busybox
      imagePullPolicy: IfNotPresent
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: DB_USER
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: db_user
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: db_name
  restartPolicy: Never
  
[root@clientvm ~]# kubectl apply -f configmap-env-pod.yaml
pod/test-pod-env created
[root@clientvm ~]# kubectl get pod
NAME                                              READY   STATUS      RESTARTS   AGE
test-pod-env                                      0/1     Completed   0          23s

[root@clientvm ~]# kubectl logs test-pod-env | grep DB
DB_NAME=wordpress
DB_USER=redhat
```

#### 将 ConfigMap 以卷的形式进行挂载的 Pod 示例：

https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#add-configmap-data-to-a-specific-path-in-the-volume

```shell
[root@clientvm ~]# cat configmap-file-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod-configmap-file
spec:
  containers:
  - name: mypod
    image: nginx
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: foo
      mountPath: "/mnt/conf"
      readOnly: true
  volumes:
  - name: foo
    configMap:
      name: haproxy
      
[root@clientvm ~]# kubectl get pod
NAME                                              READY   STATUS      RESTARTS   AGE
mypod-configmap-file                              1/1     Running     0          4s

[root@clientvm ~]# kubectl exec mypod-configmap-file -it -- bash
root@mypod-configmap-file:/# ls /mnt/conf/
haproxy.cfg
```

## Secret

### 基础理论

`Secret` 对象类型用来保存敏感信息，例如密码、OAuth 令牌和 SSH 密钥。 将这些信息放在 `secret` 中比放在 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 的定义或者 [容器镜像](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-image) 中来说更加安全和灵活。

要使用 Secret，Pod 需要引用 Secret。 Pod 可以用三种方式之一来使用 Secret：

- 作为挂载到一个或多个容器上的 [卷](https://kubernetes.io/zh/docs/concepts/storage/volumes/) 中的[文件](https://kubernetes.io/zh/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod)。
- 作为[容器的环境变量](https://kubernetes.io/zh/docs/concepts/configuration/secret/#using-secrets-as-environment-variables)
- 由 [kubelet 在为 Pod 拉取镜像时使用](https://kubernetes.io/zh/docs/concepts/configuration/secret/#using-imagepullsecrets)

### 创建generic Secret

```shell
[root@clientvm ~]# kubectl create secret -h
Create a secret using specified subcommand.

Available Commands:
  docker-registry Create a secret for use with a Docker registry
  generic         Create a secret from a local file, directory or literal value
  tls             Create a TLS secret

Usage:
  kubectl create secret [flags] [options]
  
[root@clientvm ~]# echo root>username.txt
[root@clientvm ~]# echo redhat>password.txt
[root@clientvm ~]# kubectl create secret generic my-secret1 --from-file=username.txt --from-file=password.txt
secret/my-secret1 created
[root@clientvm ~]# kubectl describe secrets my-secret1
Name:         my-secret1
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password.txt:  7 bytes
username.txt:  5 bytes

[root@clientvm ~]# kubectl create secret generic my-secret2 --from-file=user=username.txt --from-file=passwd=password.txt
secret/my-secret2 created
[root@clientvm ~]# kubectl describe secrets my-secret2
Name:         my-secret2
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
passwd:  7 bytes
user:    5 bytes

[root@clientvm ~]# kubectl create secret generic my-secret3 --from-literal=user=root --from-literal=passwd='redhat'
secret/my-secret3 created

[root@clientvm ~]# kubectl edit secrets my-secret3
......
apiVersion: v1
data:
  passwd: cmVkaGF0
  user: cm9vdA==
kind: Secret
......

[root@clientvm ~]# echo cmVkaGF0 |base64 --decode
redhat
```

### 创建docker-registry Secret

```shell
kubectl create secret docker-registry my-secret --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER
--docker-password=DOCKER_PASSWORD 
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: <your-private-image>
  imagePullSecrets:
  - name: my-secret
```

### 创建TLS Secret

```shell
kubectl create secret tls my-tls-secret --cert=path/to/cert/file --key=path/to/key/file
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mynginx
spec:
  containers:
  - name: mypod
    image: nginx
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: foo
      mountPath: "/etc/nginx/ssl"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: my-tls-secret
```

## 使用Secret

### 以环境变量方式使用secret

```shell
[root@clientvm ~]# cat secret-env-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod-env
spec:
  containers:
    - name: test-container
      image: busybox
      imagePullPolicy: IfNotPresent
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
      - secretRef:
          name: my-secret3
  restartPolicy: Never
[root@clientvm ~]# kubectl apply -f secret-env-pod.yaml
pod/secret-test-pod-env created
[root@clientvm ~]# kubectl get pod
NAME                                              READY   STATUS      RESTARTS   AGE
secret-test-pod-env                               0/1     Completed   0          8s

[root@clientvm ~]# kubectl logs secret-test-pod-env
user=root
passwd=redhat

```

### 以卷形式挂载Secret

```shell
[root@clientvm ~]# cat secret-env-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-secret-volume
spec:
  containers:
  - name: mypod
    image: nginx
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: foo
      mountPath: "/mnt/aaa"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: my-secret1
      
[root@clientvm ~]# kubectl apply -f secret-env-volume.yaml
pod/test-pod-secret-volume created

[root@clientvm ~]# kubectl exec test-pod-secret-volume -it -- bash
root@test-pod-secret-volume:/# cat /mnt/aaa/username.txt
root
root@test-pod-secret-volume:/# cat /mnt/aaa/password.txt
redhat
```

