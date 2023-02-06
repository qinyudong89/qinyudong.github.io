---
title: "Kubernetes 控制器设计模式"
date: 2022-12-20T00:20:30-04:00
categories:
  - blog
tags:
  - Kubernetes
  - Pod
  - Control Loop
  - Deployment
  - ReplicaSet
---


### 编排系统如何快速调整出错的服务？

了为方便我们理解 Kubernetes 控制器模式的工作原理，在文章的开始我们先抛出下面这样一个问题：

> 假设有个由数十个 Node、数百个 Pod、近千个 Container 所组成的分布式系统，作为管理员，你想要避免该系统因为外部流量压力、代码缺陷、软件更新、硬件升级、资源分配等各种原因而出现中断的状况，那么你希望编排系统能为你提供何种支持？

首先，我们当然最希望容器编排系统能自动地让每一个服务都永远健康，永不出错。但永不出错的服务是不切实际的，所以我们就只能退而求其次。让容器编排系统在这些服务出现问题、运行状态不正确的时候，能自动将它们调整成正确的状态。

为了实现这一目标，Kubernetes 借鉴在机器人技术和自动化领域中已经成功应用的方案，在 Kubernetes 中它被称为：[控制回路（Control Loop）](https://kubernetes.io/zh-cn/docs/concepts/architecture/controller/)。

### 控制回路

关于 *控制回路（Control Loop）*在 Kubernetes 的官方文档当中是这样描述的：

> 控制回路好比房间里的温度自动调节器，当你设置了温度，告诉了温度自动调节器你的**期望状态（Desired State）**。 房间的实际温度是**当前状态（Current State）**。 通过对设备的开关控制，温度自动调节器让其当前状态接近期望状态。

而在 Kubernetes 中，是通过不断的监控服务的当前状态与期望状态，当两个状态之前有差异时，Kubernetes 通过资源对应的控制器来调节资源的当前状态，使资源状态逐渐向期望状态靠拢。下面是控制器实现的伪代码：

```go
for {
  实际状态 := 获取集群中对象X的实际状态（Actual State）
  期望状态 := 获取集群中对象X的期望状态（Desired State）
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    执行编排动作，将实际状态调整为期望状态
  }
}
```



### 声明式 API

如果你想让资源始终维持在正确的状态，要通过配置文件描述清楚这些资源的期望状态，由 Kubernetes 中对应监视这些资源的控制器，来驱动资源的实际状态逐渐向期望状态靠拢，才能够达成自己的目的。而这种交互风格就被叫做 **Kubernetes 的声明式 API**，如果你之前有过实际操作 Kubernetes 的经验，那你日常在元数据文件中的 spec 字段所描述的就是资源的期望状态。



### Kubernetes 控制器模式的工作原理

好了，现在我们来了解一下，Kubernetes 控制器模式 具体是怎么做的呢？在回答之前，我想先解释下，毕竟我们不是在写 Kubernetes 的操作手册，没办法展开和详解每个控制器，所以下面我就以两三种资源和控制器为代表，来举例说明一下。

比如说，我们就以部署控制器（Deployment Controller）、副本集控制器（ReplicaSet Controller）和自动扩缩控制器（HPA Controller）为例，来看看 Kubernetes 控制器模式的工作原理。

> 通过服务编排，我们让任何分布式系统自动实现以下三种通用的能力：
>
> 1. Pod 出现故障时，能够自动恢复，不中断服务；
> 2. Pod 更新程序时，能够滚动更新，不中断服务；
> 3. Pod 遇到压力时，能够水平扩展，不中断服务。

先看第一个问题：**Pod 出现故障时，能够自动恢复，不中断服务；**

我们知道，虽然 Pod 本身也是资源，完全可以直接创建，但 Pod 一但出现故障无法自动恢复，所以在生产中是不提倡的。

那么，**正确的做法是通过副本集（ReplicaSet）来创建 Pod**。

Kubernetes 中的 ReplicaSet 主要的作用是维持一组 Pod 副本的运行，你可以在 ReplicaSet 资源的元数据中，描述你期望 Pod 副本的数量（即 spec.replicas 的值）。

当 ReplicaSet 成功创建之后，副本集控制器就会持续跟踪该资源，一旦有 Pod 发生崩溃退出，或者状态异常（默认是靠进程返回值，你还可以在 Pod 中设置探针，以自定义的方式告诉 Kubernetes 出现何种情况 Pod 才算状态异常），ReplicaSet 都会自动创建新的 Pod 来替代异常的 Pod；如果因异常情况出现了额外数量的 Pod，也会被 ReplicaSet 自动回收掉。

通过 ReplicaSet 我们可以确保在任何时候，集群中这个 Pod 副本的数量都会向期望状态靠拢。



再看第二个问题：**Pod 更新程序时，能够滚动更新，不中断服务**；

上面我们通过 ReplicaSet 可以保证 Pod 出现故障时自动恢复。 但是在升级程序版本时，ReplicaSet 就不得不主动中断旧 Pod 的运行，重新创建新版的 Pod 了，而这会造成服务中断。

对于某些不允许中断的业务，以前的 Kubernetes 曾经提供过 kubectl rolling-update 命令，来辅助实现滚动更新。然而，由于 rolling-update 是通过命令式交互，完全不符合 Kubernetes 声明式 API 的设计理念，继而被淘汰。

所以，现在新的部署资源（Deployment）与部署控制器就被设计出来了。具体的实现步骤是这样的：**我们可以由 Deployment 来创建 ReplicaSet，再由 ReplicaSet 来创建 Pod**，**当我们更新了 Deployment 中的信息以后（比如更新了镜像的版本），部署控制器就会跟踪到新的期望状态，自动地创建新 ReplicaSet，并逐渐缩减旧的 ReplicaSet 的副本数，直到升级完成后，彻底删除掉旧 ReplicaSet**。这个工作过程如下图所示：

![img](https://static001.geekbang.org/resource/image/36/64/36221d35f72de52836563857f5d8f364.jpg?wh=2000*864)



最后来看最后一个问题：**Pod 遇到压力时，能够水平扩展，不中断服务**

通过 Deployment 我们知道，在遇到流量压力时，我们完全可以手动修改 Deployment 中的副本数量，或者通过 kubectl scale 命令指定副本数量，促使 Kubernetes 部署更多的 Pod 副本来应对压力。然而这种扩容方式不仅需要人工参与，而且只靠人类经验来判断需要扩容的副本数量，也不容易做到精确与及时。

为此，Kubernetes 又提供了 Autoscaling 资源和自动扩缩控制器，它们能够自动地根据度量指标，如处理器、内存占用率、用户自定义的度量值等，来设置 Deployment（或者 ReplicaSet）的期望状态，实现当度量指标出现变化时，系统自动按照“Autoscaling→Deployment→ReplicaSet→Pod”这样的顺序层层变更，最终实现根据度量指标自动扩容缩容。



### 参考

https://time.geekbang.org/column/article/352652

https://kubernetes.io/zh-cn/docs/concepts/architecture/controller/

https://draveness.me/kubernetes-replicaset/