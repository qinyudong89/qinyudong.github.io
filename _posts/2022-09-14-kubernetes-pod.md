---
title: "Kubernetes的网络实现"
date: 2022-09-14T23:20:30-04:00
categories:
    - blog
tags:
    - Pod
    - Kubernetes
---

### **什么是 Pod ?**

Pod，是由一个或多个紧密耦合的容器的集合。Pod 是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元。

### **Pod 的特点**

- 共享存储、网络、以及怎样运行这些容器的声明。
- Pod 中的容器被自动安排到集群中的同一物理机或虚拟机上，并可以一起进行调度。
- 复制：Kubernetes 可以使用复制控制器根据需要水平扩展应用程序。

### **为什么我们会需要 Pod？**

因为在现实工作中，有些应用它们之间有着密切的协作关系，使得它们必须部署在同一台机器上。而如果事先没有 Pod 来维系，像这样的运维关系就会非常难以处理。

Pod 被设计成支持形成内聚服务单元的多个协作过程（形式为容器）。 Pod 中的容器被自动安排到集群中的同一物理机或虚拟机上，并可以一起进行调度。 容器之间可以共享资源和依赖、彼此通信、协调何时以及何种方式终止自身。

例如，你可能有一个容器，为共享卷中的文件提供 Web 服务器支持，以及一个单独的 "边车 (sidercar)" 容器负责从远端更新这些文件，如下图所示：

<img src="https://d33wubrfki0l68.cloudfront.net/aecab1f649bc640ebef1f05581bfcc91a48038c4/728d6/images/docs/pod.svg" alt="Pod 创建示意图" style="zoom: 5%;" />

Pod 天生地为其成员容器提供了两种共享资源：[网络](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/#pod-networking)和[存储](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/#pod-storage)。

### **Pod 如何创建及管理？**



### **Pod 的生命周期？**



### **参考**

[Pod](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/)

 [为什么我们会需要 Pod](https://time.geekbang.org/column/article/40092)