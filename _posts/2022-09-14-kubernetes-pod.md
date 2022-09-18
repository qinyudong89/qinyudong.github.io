---
title: "Kubernetes Pod"
date: 2022-09-14T23:20:30-04:00
categories:
    - blog
tags:
    - Pod
    - Kubernetes
---

### **什么是 Pod ?**

Pod，是由一个或多个紧密耦合的容器的集合。Pod 是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元。

> 紧密耦合容器的典型特征包括但不限于：互相之间会发生直接的文件交换、使用 localhost 或者 Socket 文件进行本地通信、会发生非常频繁的远程调用、需要共享某些 Linux Namespace（比如，一个容器要加入另一个容器的 Network Namespace）等等。
>
> 这也就意味着，并不是所有有“关系”的容器都属于同一个 Pod。比如，Java 应用容器和 MySQL 虽然会发生访问关系，但并没有必要、也不应该部署在同一台机器上，它们更适合做成两个 Pod。

### **Pod 的特点**

- 共享存储、网络、以及怎样运行这些容器的声明。
- Pod 中的每一个容器的生命周期是一致的。
- Pod 中的容器被自动安排到集群中的同一物理机或虚拟机上，并可以一起进行调度。
- 复制：Kubernetes 可以使用复制控制器根据需要水平扩展应用程序。

### **为什么我们会需要 Pod？**

因为在现实工作中，有些应用它们之间有着密切的协作关系，使得它们必须部署在同一台机器上。而如果事先没有 Pod 来维系，像这样的运维关系就会非常难以处理。

Pod 被设计成支持形成内聚服务单元的多个协作过程（形式为容器）。 Pod 中的容器被自动安排到集群中的同一物理机或虚拟机上，并可以一起进行调度。 容器之间可以共享资源和依赖、彼此通信、协调何时以及何种方式终止自身。

Pod 天生地为其成员容器提供了两种共享资源：[网络](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/#pod-networking)和[存储](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/#pod-storage)。

### **Pod 如何创建及管理？**

由于 Pod 被设计成了相对临时性的、用后即抛的一次性实体。当我们需要创建一个个 Pod 时，经常使用工作负载资源来创建和管理一个或多个 Pod。 资源的控制器能够处理副本的管理、上线，并在 Pod 失效时提供自愈能力。 例如，如果一个节点失败，控制器注意到该节点上的 Pod 已经停止工作， 就可以创建替换性的 Pod。调度器会将替身 Pod 调度到一个健康的节点执行。

下面是一些管理一个或者多个 Pod 的工作负载资源的示例：

- [Deployment](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/)
- [StatefulSet](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/statefulset/)
- [DaemonSet](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/daemonset/)

### **Pod 的生命周期？**

Pod 的 `status` 字段是一个 [PodStatus](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#podstatus-v1-core) 对象，其中包含一个 `phase` 字段。Pod 的阶段（Phase）是 Pod 在其生命周期中所处位置的简单宏观概述。 

![Pod Lifecycle](https://raw.githubusercontent.com/qinyudong89/imgBed/master/Pod%20Lifecycle.drawio.png)



- Pending：Pod 已被 Kubernetes 系统接受，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间。
- Running：Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。
- Succeeded：Pod 中的所有容器都已成功终止，并且不会再重启。
- Failed：Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 0 状态退出或者被系统终止。
- Unknown：因为某些原因无法取得 Pod 的状态。这种情况通常是因为与 Pod 所在主机通信失败。

### **参考**

[Pod](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/)

 [为什么我们会需要 Pod](https://time.geekbang.org/column/article/40092)