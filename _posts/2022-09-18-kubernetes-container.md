---
title: "Kubernetes Container"
date: 2022-09-18T10:20:30-04:00
categories:
    - blog
tags:
    - Container
    - Kubernetes
 
---

本篇我们主要介绍在 Kubernetes 中容器的状态、容器重启策略及容器探针（健康检测）相关内容。

# 一、容器状态

一旦调度器将 Pod 分派给某个节点，kubelet 就通过 容器运行时 开始为 Pod 创建容器。 容器的状态有三种：Waiting（等待）、Running（运行中）和 Terminated（已终止）。

要检查 Pod 中容器的状态，你可以使用 `kubectl describe pod <pod name>`。 其输出中包含 Pod 中每个容器的状态。

##### Waiting

如果容器并不处在 Running 或 Terminated 状态之一，它就处在 Waiting 状态。 处于 Waiting 状态的容器仍在运行它完成启动所需要的操作：例如，从某个容器镜像 仓库拉取容器镜像，或者向容器应用 Secret 数据等等。 

##### Running

Running 状态表明容器正在执行状态并且没有问题发生。 

##### Terminated

处于 Terminated 状态的容器已经开始执行并且或者正常结束或者因为某些原因失败。

# 容器重启策略

Pod 的 spec 中包含一个 restartPolicy 字段，其应用于当某个容器异常退出时或者健康检查失败时，kubelet 将根据何用策略重启容器 ，默认值是 Always 。

Always：当容器失效时，由 kubelet 自动重启该容器。

OnFailure ：当容器终止运行且退出码不为 0 时，由 kubelet 自动重启该容器。

Never：不论容器运行状态如何，kubelet都不会重启该容器。 

# 容器探针

[kubelet](https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kubelet/) 对容器执行的定期诊断。 要执行诊断，kubelet 既可以在容器内执行代码，也可以发出一个网络请求。

#### 检查机制

###### exec

在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。

###### grpc

使用 gRPC 执行一个远程过程调用。 目标应该实现 gRPC健康检查。 如果响应的状态是 "SERVING"，则认为诊断成功。 gRPC 探针是一个 alpha 特性，只有在你启用了 "GRPCContainerProbe" 特性门控时才能使用。

###### httpGet

对容器的 IP 地址上指定端口和路径执行 HTTP GET 请求。如果响应的状态码大于等于 200 且小于 400，则诊断被认为是成功的。

###### tcpSocket

对容器的 IP 地址上的指定端口执行 TCP 检查。如果端口打开，则诊断被认为是成功的。 如果远程系统（容器）在打开连接后立即将其关闭，这算作是健康的。

#### 探测类型

###### livenessProbe（存活探针）

指示容器是否正在运行。如果存活态探测失败，则 kubelet 会杀死容器， 并且容器将根据其重启策略决定未来。如果容器不提供存活探针， 则默认状态为 Success。

###### readinessProbe（绪态探针）

指示容器是否准备好为请求提供服务。如果就绪态探测失败， 端点控制器将从与 Pod 匹配的所有服务的端点列表中删除该 Pod 的 IP 地址。 初始延迟之前的就绪态的状态值默认为 Failure。 如果容器不提供就绪态探针，则默认状态为 Success。

###### startupProbe（启动探针）

指示容器中的应用是否已经启动。如果提供了启动探针，则所有其他探针都会被禁用，直到此探针成功为止。如果启动探测失败，kubelet 将杀死容器，而容器依其 重启策略进行重启。 如果容器没有提供启动探测，则默认状态为 Success。

#### 各个探针何时使用

livenessProbe：

- 如果你希望容器在探测失败时被杀死并重新启动，那么请指定一个存活态探针， 并指定 restartPolicy 为 "Always" 或 "OnFailure"。

readinessProbe：

- 如果要仅在探测成功时才开始向 Pod 发送请求流量，请指定就绪态探针。
- 如果你希望容器能够自行进入维护状态，也可以指定一个就绪态探针。

startupProbe：

- 对于所包含的容器需要较长时间才能启动就绪的 Pod 而言，启动探针是有用的。 你不再需要配置一个较长的存活态探测时间间隔，只需要设置另一个独立的配置选定， 对启动期间的容器执行探测，从而允许使用远远超出存活态时间间隔所允许的时长。

# 参考

[Pod Lifecycle](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/)

[configure-liveness-readiness-startup-probes](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
