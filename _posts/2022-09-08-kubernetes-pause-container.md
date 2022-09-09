---
title: "Kubernetes pause 容器"
date: 2022-09-08T11:20:30-04:00
categories:
  - blog 
tags:
  - kubernetes
  - pause
---

### **pause 容器是什么？**

> Pause Container 保存 pod 的网络命名空间。 Kubernetes 创建暂停容器以获取相应 pod 的 IP 地址，并为加入该 pod 的所有其他容器设置网络命名空间。
>
>  暂停容器确保有一个网络堆栈要映射，并且它不会改变。暂停容器是一个 pod 实现细节，其中一个暂停容器用于每个 pod，并且在列出属于 pod 成员的容器时不应显示
>
> 在创建单个容器之前，暂停容器会保存 pod 的 cgroup、预留和命名空间。 pause 容器的镜像始终存在，因此 pod 的资源分配是即时的。
>
>  默认情况下，暂停容器是隐藏的，但您可以通过运行 docker ps -a 来查看它们。

### **为什么需要 pause 容器？**

> Docker支持以containers的方式部署软件，container也非常适合用来部署单个软件。但是，当我们想一起运行一个软件的多个模块的时候，这种方式又会变得非常的笨重。我们会常常遇到这种情况，当开发人员创建了多个docker镜像后，还需要使用监控模块去启动和管理多个进程。在生产环境下，会发现如果把这些应用部署为一组容器，并将这些容器组彼此分隔，每个容器组共享一个环境，这种方式会更有效。
>
> Kubernetes为应对这种case，提出了pod的抽象概念。Pod的概念，隐藏了docker中复杂的标志位以及管理docker容器、共享卷及其他docker资源的复杂性。同时也隐藏了不同容器运行环境的差异。

### **pause 容器两个主要功能**

第一，它提供整个pod的Linux命名空间的基础。第二，启用PID命名空间，它在每个pod中都作为PID为1进程，并回收僵尸进程。

- 提供整个pod的Linux命名空间的基础
- 启用PID命名空间，它在每个pod中都作为PID为1进程，并回收僵尸进程

### **如果 Pause 容器被删除会怎样？**

当 Pause 容器被删除时，Kubernetes 认为整个 Pod 不可用，即使 Pod 中还有其他业务容器正在运行。因此，Kubernetes 将使用全新的 IP 地址重新创建 Pod。但是如果业务容器停止，Kubernetes 不会重新创建 pod，只会重新创建业务应用容器，没有任何 IP 变化。

### **参考**

[Demystifying Kubernetes Networking — Episode 2](https://sanjimoh.medium.com/kubernetes-secret-recipe-8d03892b27ae)

[kubernetes pod为什么需要pause容器](https://zhuanlan.zhihu.com/p/81666226)

