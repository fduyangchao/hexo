---
title: Pod中的容器权限管理和配置
date: 2021-10-14 10:52:55
tags:
  - Kubernetes
  - Docker
  - container
categories:
  - 专业
  - 开发
  - Kubernetes
---

开源项目部署后，如果升级镜像版本，有时会出现容器内部权限不足，导致业务进程无法读写文件。本文从容器和文件系统两个角度出发，总结容器镜像和Kubernetes如何控制并配置业务权限。

<!-- more -->

Pod权限管理可以分为两类：1. 容器内进程所属的用户和组 2. 容器文件系统所属的用户和组。当且仅当容器进程用户为`root:root`，或者与文件系统用户相同时，才拥有对文件的读写权限。

如果不配置任何用户组，容器进程和文件系统默认为root用户组。下面做一个简单的demo，起一个alpine容器，将宿主机的`/tmp/mnt`目录挂载到Pod的`/mnt`目录，不做任何权限配置。

```
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  containers:
  - args:
    - -c
    - sleep 36000000
    command:
    - /bin/sh
    image: alpine:latest
    imagePullPolicy: IfNotPresent
    name: container
    volumeMounts:
    - mountPath: /mnt
      name: mnt
  volumes:
  - name: mnt
    hostPath:
      path: /tmp/mnt
      type: DirectoryOrCreate
```

进入容器内部查看，容器进程、文件系统的用户组均为`root:root`

```
[root@laptop]# kubectl exec -it demo sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
/ # 
/ # whoami
root
/ # 
/ # ls -atl /mnt
total 0
drwxr-xr-x    2 root     root             6 Oct 15 08:05 .
drwxr-xr-x    1 root     root            29 Oct 15 08:05 ..
/ # 
/ # ps ef
PID   USER     TIME  COMMAND
    1 root      0:00 sleep 36000000
   14 root      0:00 sh
   21 root      0:00 ps ef

```

## 如何设置容器进程的uid/gid

### Dockerfile定义

```
FROM alpine:latest
RUN addgroup -g 1001 -S appuser && adduser -u 1001 -S appuser -G appuser
USER appuser
```

### PodSecurityContext

Pod `spec.securityContext`定义`runAsUser`, `runAsGroup`，比Dockerfile优先级更高

```
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  securityContext:
    runAsUser: 1001
    runAsGroup: 1001
  containers:
  - args:
    - -c
    - sleep 36000000
    command:
    - /bin/sh
    image: alpine:latest
    imagePullPolicy: IfNotPresent
    name: container
    volumeMounts:
    - mountPath: /mnt
      name: mnt
  volumes:
  - name: mnt
    hostPath:
      path: /tmp/mnt
      type: DirectoryOrCreate
```

### SecurityContext

Container `securityContext`也可定义`runAsUser`, `runAsGroup`，比Pod `securityContext`优先级更高

```
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  securityContext:
    runAsUser: 1001
    runAsGroup: 1001
  containers:
  - args:
    - -c
    - sleep 36000000
    command:
    - /bin/sh
    image: alpine:latest
    imagePullPolicy: IfNotPresent
    name: container
    volumeMounts:
    - mountPath: /mnt
      name: mnt
    securityContext:
      runAsUser: 2000    <------------ 以这个配置为准
      runAsGroup: 2000
  volumes:
  - name: mnt
    hostPath:
      path: /tmp/mnt
      type: DirectoryOrCreate
```



## 如何设置Pod文件系统的uid/gid

### Dockerfile定义

```
FROM alpine:latest
RUN addgroup -g 1001 -S appuser && adduser -u 1001 -S appuser -G appuser
RUN chown -R 1001:1001 /mnt
```

### PosSecurityContext

自定义pod `spec.securityContext.fsGroup`，当容器进程权限 >= fsGroup权限才可以对挂载的存储卷进行读写操作。

按照Kubernetes官方文档的说法：由于设置了`fsGroup`，容器中所有进程也会是补充组ID 2000的一部分；存储卷挂载路径`/mnt`和在该路径中创建的任何文件都属于组ID 2000，因此容器进程有读写挂载存储卷的权利。但是实际验证下来，发现hostPath目录属于`root:root`，而不是设置的`fsGroup`，导致容器进程无法对`hostPath`挂载的路径`/mnt`进行读写操作。

查看官方文档和自测用例的区别，发现官方文档使用`emptyDir`，自测用例使用的是`hostPath`，可能是两种存储插件的权限管理有区别？下面定义Pod挂载`hostPath`和`emptyDir`两种类型的存储卷，并设置`fsGroup=2000`，验证二者是否存在差别。

```
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  securityContext:
    runAsUser: 1001
    runAsGroup: 1001
    fsGroup: 2000
  containers:
  - args:
    - -c
    - sleep 36000000
    command:
    - /bin/sh
    image: alpine:latest
    imagePullPolicy: IfNotPresent
    name: container
    volumeMounts:
    - mountPath: /mnt
      name: mnt
    - mountPath: /emptydir
      name: emptydir
  nodeSelector:
    kubernetes.io/hostname: tos055
  volumes:
  - name: mnt
    hostPath:
      path: /tmp/mnt
      type: DirectoryOrCreate
  - name: emptydir
    emptyDir: {}
```

进入容器内部查看容器进程所属`uid/gid/group`，以及挂载路径的`gid`：

```
[root@laptop]# kubectl exec -it demo sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
/ $ 
/ $ id
uid=1001 gid=1001 groups=2000
/ $ 
/ $ ls -l 
total 8
drwxr-xr-x    2 root     root          4096 Aug 27 11:05 bin
drwxr-xr-x    5 root     root           360 Oct 15 08:36 dev
drwxrwsrwx    2 root     2000            15 Oct 15 08:40 emptydir
drwxr-xr-x    1 root     root            66 Oct 15 08:36 etc
drwxr-xr-x    2 root     root             6 Aug 27 11:05 home
drwxr-xr-x    7 root     root           247 Aug 27 11:05 lib
drwxr-xr-x    5 root     root            44 Aug 27 11:05 media
drwxr-xr-x    2 root     root             6 Oct 15 08:05 mnt
drwxr-xr-x    2 root     root             6 Aug 27 11:05 opt
dr-xr-xr-x  841 root     root             0 Oct 15 08:36 proc
drwx------    2 root     root             6 Aug 27 11:05 root
drwxr-xr-x    1 root     root            21 Oct 15 08:36 run
drwxr-xr-x    2 root     root          4096 Aug 27 11:05 sbin
drwxr-xr-x    2 root     root             6 Aug 27 11:05 srv
dr-xr-xr-x   13 root     root             0 Oct 15 08:36 sys
drwxrwxrwt    2 root     root             6 Aug 27 11:05 tmp
drwxr-xr-x    7 root     root            66 Aug 27 11:05 usr
drwxr-xr-x   12 root     root           137 Aug 27 11:05 var
/ $ 
/ $ cd /emptydir/
/emptydir $ touch file
/emptydir $ 
/emptydir $ cd ..
/ $ 
/ $ cd /mnt
/mnt $ touch file
touch: file: Permission denied

```

确实如猜测的那样，`fsGroup`对`hostPath`挂载的存储卷并不生效：业务进程有权限在`/emptydir`目录下创建文件（容器进程属于补充组ID 2000，`/emptydir`属于组ID 2000），却没有权限在`/mnt`目录下创建文件（容器进程`uid=1001, gid=1001, group=2000`，而`/mnt`属于`root:root`）

容器`hostPath`挂载路径的用户和组权限是不是跟着宿主机走呢？下面来验证。

将宿主机目录`/tmp/mnt`设置为`300:300`，再将该路径通过`hostPath`挂载到Pod中：

```
[root@laptop]# groupadd -g 3000 host
[root@laptop]# useradd -g host -u 3000 host
[root@laptop]# chown -R host:host /tmp/mnt
[root@laptop]# ls -atl /tmp/mnt
total 8
drwxrwxrwt. 47 root root 4096 10月 15 16:48 ..
drwxr-xr-x   2 host host    6 10月 15 16:05 .
[root@laptop]# kubectl create -f pod.yaml 
pod/demo created
```

shell进入容器进程：

```
[root@laptop]# kubectl exec -it demo sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
/ $ 
/ $ 
/ $ id
uid=1001 gid=1001 groups=2000
/ $ 
/ $ ls -l
total 8
drwxr-xr-x    2 root     root          4096 Aug 27 11:05 bin
drwxr-xr-x    5 root     root           360 Oct 15 09:23 dev
drwxrwsrwx    2 root     2000             6 Oct 15 09:23 emptydir
drwxr-xr-x    1 root     root            66 Oct 15 09:23 etc
drwxr-xr-x    2 root     root             6 Aug 27 11:05 home
drwxr-xr-x    7 root     root           247 Aug 27 11:05 lib
drwxr-xr-x    5 root     root            44 Aug 27 11:05 media
drwxr-xr-x    2 host     host             6 Oct 15 08:05 mnt
drwxr-xr-x    2 root     root             6 Aug 27 11:05 opt
dr-xr-xr-x  841 root     root             0 Oct 15 09:23 proc
drwx------    2 root     root             6 Aug 27 11:05 root
drwxr-xr-x    1 root     root            21 Oct 15 09:23 run
drwxr-xr-x    2 root     root          4096 Aug 27 11:05 sbin
drwxr-xr-x    2 root     root             6 Aug 27 11:05 srv
dr-xr-xr-x   13 root     root             0 Oct 15 09:23 sys
drwxrwxrwt    2 root     root             6 Aug 27 11:05 tmp
drwxr-xr-x    7 root     root            66 Aug 27 11:05 usr
drwxr-xr-x   12 root     root           137 Aug 27 11:05 var
/ $ cd /emptydir/
/emptydir $ touch file
/emptydir $ cd ..
/ $ cd /mnt
/mnt $ 
/mnt $ touch file
touch: file: Permission denied
```

**如设想的一样，容器内部挂载的路径`/mnt`也是`300:300`权限，继承宿主机目录。**

在云平台中，使用最多的通过PVC接入的第三方存储卷，`fsGroup`对这类存储卷是否生效？

类似地，创建一个带有PVC模板的`StatefulSet`

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: demo
spec:
  serviceName: demo
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      securityContext:
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 2000
      containers:
        - name: container
          image: alpine:latest
          command:
          - /bin/sh
          args:
          - -c
          - sleep 36000000
          volumeMounts:
            - mountPath: /mnt
              name: mnt
            - mountPath: /emptydir
              name: emptydir
            - mountPath: /pvc
              name: pvc
      volumes:
      - name: mnt
        hostPath:
          path: /tmp/mnt
          type: DirectoryOrCreate
      - name: emptydir
        emptyDir: {}
  volumeClaimTemplates:
    - metadata:
        name: pvc
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Mi
        storageClassName: longhorn-default
```

等待Pod Runing之后，shell进入容器进程，**发现fsGroup对PVC挂载路径的权限生效**。

```
[root@laptop]# kubectl exec -it demo-0 sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
/ $ 
/ $ id
uid=1001 gid=1001 groups=2000
/ $ 
/ $ ls -l
total 8
drwxr-xr-x    2 root     root          4096 Aug 27 11:05 bin
drwxr-xr-x    5 root     root           360 Oct 15 09:39 dev
drwxrwsrwx    2 root     2000             6 Oct 15 09:39 emptydir
drwxr-xr-x    1 root     root            66 Oct 15 09:39 etc
drwxr-xr-x    2 root     root             6 Aug 27 11:05 home
drwxr-xr-x    7 root     root           247 Aug 27 11:05 lib
drwxr-xr-x    5 root     root            44 Aug 27 11:05 media
drwxr-xr-x    2 root     root             6 Oct 15 08:05 mnt
drwxr-xr-x    2 root     root             6 Aug 27 11:05 opt
dr-xr-xr-x  841 root     root             0 Oct 15 09:39 proc
drwxrwsr-x    2 root     2000             6 Oct 15 09:39 pvc
drwx------    2 root     root             6 Aug 27 11:05 root
drwxr-xr-x    1 root     root            21 Oct 15 09:39 run
drwxr-xr-x    2 root     root          4096 Aug 27 11:05 sbin
drwxr-xr-x    2 root     root             6 Aug 27 11:05 srv
dr-xr-xr-x   13 root     root             0 Oct 15 09:39 sys
drwxrwxrwt    2 root     root             6 Aug 27 11:05 tmp
drwxr-xr-x    7 root     root            66 Aug 27 11:05 usr
drwxr-xr-x   12 root     root           137 Aug 27 11:05 var
/ $ 
/ $ cd /emptydir/
/emptydir $ touch file
/emptydir $ cd ..
/ $ cd /pvc
/pvc $ touch file
/pvc $ cd ..
/ $ cd /mnt
/mnt $ touch file
touch: file: Permission denied
```

**综上，对于Pod挂载的存储卷，如果通过hostPath挂载，则fsGroup不生效，容器内部的挂载路径继承成宿主机的权限；如果通过emptyDir或者PVC挂载，fsGroup生效。**

### PersitentVolume

PV `mountOptions`或者 `pv.beta.kubernetes.io/gid` 注释，定义路径挂载时使用哪个用户组

### InitContainer

类似于Dockerfile，在initContainer中使用`chown`修改文件系统的用户组

```
initContainers:
- name: init
  image: alpine:latest
  command: ["sh", "-c", "chown -R 1001:1001 /mnt"]
  volumeMounts:
  - name: mount-pvc
    mountPath: /mnt
```

## 参考

https://stackoverflow.com/questions/51390789/kubernetes-persistent-volume-claim-mounted-with-wrong-gid

https://kubernetes.io/zh/docs/tasks/configure-pod-container/security-context/

https://serverfault.com/questions/906083/how-to-mount-volume-with-specific-uid-in-kubernetes-pod
