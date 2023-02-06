---
title: "Kubernetes 的调度机制"
date: 2022-12-23T00:20:30-04:00
categories:
  - blog
tags:
  - Kubernetes
  - Pod
  - Scheduler
---


### 调度流程

在一个集群中，满足一个 Pod 调度请求的所有 Node 称之为 **可调度 Node**。 调度器先在集群中通过一组 **Predicates 算法** 找到一个 Pod 的可调度 Node，然后根据一组 **Priorities 算法** 对这些可调度 Node 打分， 之后选出其中得分最高的 Node 来运行 Pod。 最后，调度器将这个调度决定告知 kube-apiserver，这个过程叫做 **绑定（Binding）**。



### 过滤(Predicates )

Predicate 本质上是一组节点过滤器（Filter），它会根据预设的过滤策略来筛选节点。Kubernetes 中默认有三种过滤策略，分别是：

1. **通用过滤策略**：最基础的调度过滤策略，用来检查节点是否能满足 Pod 声明中需要的资源。比如处理器、内存资源是否满足，主机端口与声明的 NodePort 是否存在冲突，Pod 的选择器或者nodeAffinity指定的节点是否与目标相匹配，等等。
2. **卷过滤策略**：与存储相关的过滤策略，用来检查节点挂载的 Volume 是否存在冲突（比如将一个块设备挂载到两个节点上），或者 Volume 的可用区域是否与目标节点冲突，等等。在“Kubernetes 存储设计”中提到的 Local PersistentVolume 的调度检查，就是在这里处理的。
3. **节点过滤策略**：与宿主机相关的过滤策略，最典型的是 Kubernetes 的污点与容忍度机制（Taints and Tolerations），比如默认情况下，Kubernetes 会设置 Master 节点不允许被调度，这就是通过在 Master 中施加污点来避免的。前面我提到的控制节点处于驱逐状态，或者在驱逐后一段时间不允许调度，也是在这个策略里实现的。



##### 调度器性能调优

请试想一下，假设现在有一个由数千节点组成的集群，每次 Pod 的创建，都必须依据各节点的实时资源状态来确定调度的目标节点，然而我们知道，各节点的资源是随着程序运行无时无刻都在变动的，资源状况只有它本身才清楚。

这样，如果每次调度都要发生数千次的远程访问来获取这些信息的话，那压力与耗时都很难降下来。所以结果不仅会让调度器成为集群管理的性能瓶颈，还会出现因耗时过长，某些节点上资源状况已发生变化，调度器的资源信息过时，而导致调度结果不准确等问题。

因此，针对前面所说的问题，Google 在论文《[Omega: Flexible, Scalable Schedulers for Large Compute Clusters](https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/41684.pdf)》里总结了自身的经验，并参考了当时[Apache Mesos](https://en.wikipedia.org/wiki/Apache_Mesos)和[Hadoop on Demand（HOD）](https://hadoop.apache.org/docs/r1.0.4/cn/hod.html)的实现，提出了一种共享状态（Shared State）的双循环调度机制。



这种调度机制后来不仅应用在 Google 的 Omega 系统（Borg 的下一代集群管理系统）中，也同样被 Kubernetes 继承了下来，它整体的工作流程如下图所示：

![img](https://static001.geekbang.org/resource/image/4a/88/4ab5138f0f80db796bc07d5cb1b10d88.jpg?wh=2000*944)



“**状态共享的双循环**”中，第一个控制循环可被称为“**Informer Loop**”，它是一系列Informer的集合，这些 Informer 会持续监视 etcd 中与调度相关资源（主要是 Pod 和 Node）的变化情况，一旦 Pod、Node 等资源出现变动，就会触发对应 Informer 的 Handler。

Informer Loop 的职责是根据 etcd 中的资源变化，去更新**调度队列（Priority Queue）**和**调度缓存（Scheduler Cache）**中的信息。

比如当有新 Pod 生成，就将其入队（Enqueue）到调度队列中，如有必要，还会根据优先级触发插队和抢占操作。再比如，当有新的节点加入集群，或者已有的节点资源信息发生变动，Informer 也会把这些信息更新同步到调度缓存之中。

另一个控制循环可被称为“Scheduler Loop”，它的核心逻辑是不停地把调度队列中的 Pod 出队（Pop），然后使用 Predicate 算法进行节点选择。



**另外 Kubernetes 还通过并发来提高调度器的执行效率。**

当开始调度一个 Pod 时，Kubernetes 调度器会同时启动 16 个 Goroutine，来并发地为集群里的所有 Node 计算 Predicates，最后返回可以运行这个 Pod 的宿主机列表。

在为每个 Node 执行 Predicates 时，调度器会按照固定的顺序来进行检查。这个顺序，是按照 Predicates 本身的含义来确定的。比如，宿主机相关的 Predicates 会被放在相对靠前的位置进行检查。要不然的话，在一台资源已经严重不足的宿主机上，上来就开始计算 PodAffinityPredicate，是没有实际意义的。



### 打分(Priorities )

经过 Predicate 算法筛选出来符合要求的节点集，会交给 Priorities 算法来打分（0~10 分）排序，以便挑选出“最恰当”的一个。

那么，具体该如何为节点打分呢。

Kubernetes 提供了不同的打分规则来满足不同的主观需求，比如最常用的 LeastRequestedPriority 规则，它的计算公式是：

```
score = (cpu((capacity-sum(requested))×10/capacity) + memory((capacity-sum(requested))×10/capacity))/2
```

从公式上，我们能很容易地看出，这就是在选择处理器和内存空闲资源最多的节点，因为这些资源剩余越多，得分就越高。经常与它一起工作的是 BalancedResourceAllocation 规则，它的公式是：

```
score = 10 - variance(cpuFraction,memoryFraction,volumeFraction)×10
```

在这个公式中，三种 Fraction 的含义是 Pod 请求的资源除以节点上的可用资源，variance 函数的作用是计算各种资源之间的差距，差距越大，函数值越大。由此可知，BalancedResourceAllocation 规则的意图是希望调度完成后，所有节点里各种资源分配尽量均衡，避免节点上出现诸如处理器资源被大量分配、而内存大量剩余的尴尬状况。



### 最后

这样，经过 Predicate 的筛选、Priorities 的评分之后，调度器已经选出了调度的最终目标节点，最后一步就是通知目标节点的 kubelet 可以去创建 Pod 了。我们要知道，调度器并不会直接与 kubelet 通讯来创建 Pod，它只需要把待调度的 Pod 的nodeName字段更新为目标节点的名字即可，kubelet 本身会监视该值的变化来接手后续工作。

不过，从调度器在 etcd 中更新nodeName，到 kubelet 从 etcd 中检测到变化，再执行 **Admit 操作二次确认**调度可行性，最后到 Pod 开始实际创建，这个过程可能会持续一段不短的时间，如果一直等待这些工作都完成了，才宣告调度最终完成，那势必也会显著影响调度器的效率。

所以实际上，Kubernetes 调度器采用了**乐观绑定（Optimistic Binding）**的策略来解决这个问题，它会同步地更新调度缓存中 Pod 的nodeName字段，并异步地更新 etcd 中 Pod 的nodeName字段，这个操作被称为绑定（Binding）。如果最终调度成功了，那 etcd 与调度缓存中的信息最终必定会保持一致，否则如果调度失败了，那就会由 Informer 来根据 Pod 的变动，将调度成功却没有创建成功的 Pod 清空nodeName字段，重新同步回调度缓存中，以便促使另外一次调度的开始。



### 参考

https://jvns.ca/blog/2017/07/27/how-does-the-kubernetes-scheduler-work/

https://time.geekbang.org/column/article/360125

https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/kube-scheduler/