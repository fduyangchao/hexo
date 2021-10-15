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

Pod权限管理可以分为两类：1. 容器内进程所属的用户和组 2. 容器文件系统所属的用户和组。当且仅当容器进程用户为root:root，或者与文件系统用户相同时，才拥有对文件的读写权限。

先将权限配置姿势总结如下：

## 如何设置容器进程的uid/gid

### Dockerfile定义

```
FROM alpine:latest
RUN addgroup -g 1001 -S appuser && adduser -u 1001 -S appuser -G appuser
USER appuser
```

### PodSecurityContext

Pod spec.securityContext定义runAsUser, runAsGroup，比Dockerfile优先级更高

### ContainerSecurityContext

container securityContext也可定义runAsUser, runAsGroup，比pod securityContext优先级更高

## 如何设置Pod文件系统的uid/gid

### Dockerfile定义

```
FROM alpine:latest
RUN addgroup -g 1001 -S appuser && adduser -u 1001 -S appuser -G appuser
RUN chown -R 1001:1001 /mnt
```

### PosSecurityContext

自定义pod spec.securityContext.fsGroup，当容器进程权限>=fsGroup权限才可以对挂载的存储卷进行读写操作

### PV Configuration

pv mountOptions或者 pv.beta.kubernetes.io/gid 注释，定义路径挂载时使用哪个用户组

### InitContainer

```
initContainers:
- name: init
  image: alpine:latest
  command: ["sh", "-c", "chown -R 1001:1001 /mnt"]
  volumeMounts:
  - name: mount-pvc
    mountPath: /mnt
```

# 参考

https://stackoverflow.com/questions/51390789/kubernetes-persistent-volume-claim-mounted-with-wrong-gid

https://kubernetes.io/zh/docs/tasks/configure-pod-container/security-context/

https://serverfault.com/questions/906083/how-to-mount-volume-with-specific-uid-in-kubernetes-pod
