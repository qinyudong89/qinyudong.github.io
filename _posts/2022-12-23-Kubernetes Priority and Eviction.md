---
title: "Kubernetes 优先级与驱逐机制"
date: 2022-12-23T12:20:30-04:00
categories:
  - blog
tags:
  - Kubernetes
  - Pod
  - Scheduler
---

### 资源模型

在 Kubernetes 里，Pod 是最小的原子调度单位。这也就意味着，所有跟调度和资源管理相关的属性都应该是属于 Pod 对象的字段。而这其中最重要的部分，就是 Pod 的 CPU 和内存配置。

Pod 需要多少资源，这个其实是开发者最为了解的。因此，Kubernetes 提供了两个参数 limits 和 requests 供开发者根据需要来设置。

其中，request 是给调度器用的，Kubernetes 选择哪个节点运行 Pod，只会根据 requests 的值来进行决策；而 limits 才是给 cgroups 用的，Kubernetes 在向 cgroups 的传递资源配额时，会按照limits的值来进行设置。

Kubernetes 这种对 CPU 和内存资源限额的设计，实际上参考了 Borg 论文中对“动态资源边界”的定义，既：**容器化作业在提交时所设置的资源边界，并不一定是调度系统所必须严格遵守的，这是因为在实际场景中，大多数作业使用到的资源其实远小于它所请求的资源限额**。

基于这种假设，Borg 在作业被提交后，会主动减小它的资源限额配置，以便容纳更多的作业、提升资源利用率。而当作业资源使用量增加到一定阈值时，Borg 会通过“快速恢复”过程，还原作业原始的资源限额，防止出现异常情况。

而 Kubernetes 的 requests + limits 的做法，其实就是上述思路的一个简化版：用户在提交 Pod 时，可以声明一个相对较小的 requests 值供调度器使用，而 Kubernetes 真正设置给容器 Cgroups 的，则是相对较大的 limits 值。不难看到，这跟 Borg 的思路相通的。



### QoS模型(Quality of Service) 即服务质量

QoS(Quality of Service) 即服务质量，QoS 是一种控制机制。在 Kubernetes 中，不同的 requests 和 limits 的设置方式，其实会将这个 Pod 划分到不同的 QoS 级别当中。它的主要应用场景，是当宿主机资源紧张的时候，kubelet 对 Pod 进行 Eviction（即资源回收）时需要用到的。



- `Guaranteed`：如果 Pod 中所有的容器都设置了limits和requests，且两者的值相等；

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: qos-demo
    namespace: qos-example
  spec:
    containers:
    - name: qos-demo-ctr
      image: nginx
      resources:
        limits:
          memory: "200Mi"
          cpu: "700m"
        requests:
          memory: "200Mi"
          cpu: "700m"
  ```

  

- `Burstable`：如果 Pod 中有部分容器的 requests 值小于limits值，或者只设置了requests而未设置limits；

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: qos-demo-2
    namespace: qos-example
  spec:
    containers:
    - name: qos-demo-2-ctr
      image: nginx
      resources:
        limits
          memory: "200Mi"
        requests:
          memory: "100Mi"
  ```

  

- `BestEffort`：如果是前面说的那种情况，limits和requests两个都没设置；

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: qos-demo-3
    namespace: qos-example
  spec:
    containers:
    - name: qos-demo-3-ctr
      image: nginx
  ```

  

三者的优先级如下所示，依次递增：

```go
Guaranteed > Burstable > BestEffort
```

一般来说，我们会建议把数据库应用等有状态的应用，或者是一些重要的、要保证不能中断的业务的服务质量等级定为 Guaranteed。这样，除非是 Pod 使用超过了它们的 limits 所描述的不可压缩资源，或者节点的内存压力大到 Kubernetes 已经杀光所有等级更低的 Pod 了，否则它们都不会被系统自动杀死。

而相对地，我们也应该把一些临时的、不那么重要的任务设置为 BestEffort，这样有利于它们调度时能在更大的节点范围中寻找宿主机，也利于它们在宿主机中利用更多的资源，快速地完成任务，然后退出，尽量缩减影响范围；当然，遇到系统资源紧张时，它们也更容易被系统杀掉。



### 优先级

除了服务质量等级以外，Kubernetes 还允许系统管理员自行决定 Pod 的优先级，这是通过类型为 PriorityClass 的资源来实现的。优先级决定了 Pod 之间的关系是否平等，同时它还会直接影响 Pod 调度与生存的关键。

优先级主要有两点影响：

1. 调度成功率

   > 高优先级的 Pod 会可以优先抢占节点资源，因此调度成功几率越大。

2. 抢占机制

   > 如果没有设置优先级的情况下，如果 Pod 调度失败，就会暂时处于 Pending 状态被搁置起来，直到集群中有新节点加入或者旧 Pod 退出。
   >
   > 如果有一个已被设置了明确优先级的 Pod 调度失败，无法创建的话，Kubernetes 就会在系统中寻找出一批牺牲者（Victims），把它们杀掉以便给更高优先级的 Pod 让出资源。而这个寻找的原则，就是在优先级低于待调度 Pod 的所有已调度的 Pod 里，按照优先级从低到高排序，从最低的杀起，直至腾出的资源可以满足待调度 Pod 的成功调度为止，或者已经找不到更低优先级的 Pod 为止。



### 驱逐机制

当有新的 Pod 因集群中所有节点的空闲资源都不满足调度时。Kubernetes 就迫不得已要杀掉一部分 Pod，以腾出资源来保证其余 Pod 能正常运行，而这个行为叫作**驱逐**。

**Pod 的驱逐机制是通过 kubelet 来执行的**，kubelet 是部署在每个节点的集群管理程序，因为它本身就运行在节点中，所以最容易感知到节点的资源实时耗用情况。kubelet 一旦发现某种不可压缩资源将要耗尽，就会主动终止节点上服务质量等级比较低的 Pod，以保证其他更重要的 Pod 的安全。而被驱逐的 Pod 中，所有的容器都会被终止，Pod 的状态会被更改为 Failed。

由于，驱逐有可能会导致服务产生中断，驱逐机制中就有了**软驱逐（Soft Eviction）**、**硬驱逐（Hard Eviction）**以及优雅退出期（Grace Period）的概念：

- **软驱逐**：通常会配置一个比较低的警戒线（比如可用内存仅剩 20%），当触及此线时，系统就会进入一段观察期。如果只是暂时的资源抖动，在观察期内能够恢复到正常水平的话，那就不会真正启动驱逐操作。否则，资源持续超过警戒线一段时间，就会触发 Pod 的优雅退出（Grace Shutdown），系统会通知 Pod 进行必要的清理工作（比如将缓存的数据落盘），然后自行结束。在优雅退出期结束后，系统会强制杀掉还没有自行了断的 Pod。
- **硬驱逐**：通常会配置一个比较高的终止线（比如可用内存仅剩 10%），一旦触及此线，系统就会立即强制杀掉 Pod，不理会优雅退出。

**软驱逐是为了减少资源抖动对服务的影响，硬驱逐是为了保障核心系统的稳定，它们并不矛盾，一般会同时使用。**



### 垃圾回收 VS 驱逐

1. **垃圾回收是安全的内存回收行为，而驱逐 Pod 是一种毁坏性的清理行为**，它有可能会导致服务产生中断，因而必须更加谨慎。比如说，要同时兼顾到硬件资源可能只是短时间内，间歇性地超过了阈值的场景，以及资源正在被快速消耗，很快就会危及高服务质量的 Pod、甚至是整个节点稳定的场景。
2. 垃圾回收可以“应收尽收”，而驱逐显然不行，系统不能无缘无故地把整个节点中所有可驱逐的 Pod 都清空掉。但是，系统通常也不能只清理到刚刚低于警戒线就停止，必须要考虑到驱逐之后的新 Pod 调度与旧 Pod 运行的新增消耗。为此，Kubernetes 提供了--eviction-minimum-reclaim参数，用于设置一旦驱逐发生之后，至少要清理出来多少资源才会终止。另外，为了避免被驱逐的 Pod ，过段时间又重新被调度到当前节点。Kubernetes 还提供了另一个参数--eviction-pressure-transition-period来约束调度器，在驱逐发生之后多长时间内，不能往该节点调度 Pod。