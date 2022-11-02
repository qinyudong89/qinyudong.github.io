---
title: "Kubernetes Pod"
date: 2022-11-02T13:20:30-04:00
categories:
    - blog
tags:
    - Linux
    - Namespace
    - Cgroups
---
*这篇文章是我们关于容器技术系列的一部分：*

*什么是命名空间和 cgroup，它们是如何工作的？（这个帖子）[构建更小的容器镜像](https://www.nginx.com/blog/building-smaller-container-images/)*


最近，我一直在研究[NGINX Unit](https://unit.nginx.org/)，我们的开源多语言应用服务器。作为我调查的一部分，我注意到 Unit 支持命名空间和 cgroup，这可以实现[进程隔离](https://unit.nginx.org/configuration/#process-isolation)。在这篇博客中，我们将了解这两种主要的 Linux 技术，它们也是[容器](https://www.nginx.com/resources/glossary/container/)的基础。

容器和 Docker 和[Kubernetes](https://www.nginx.com/resources/glossary/kubernetes)等相关工具已经出现了一段时间。它们帮助改变了软件在现代应用程序环境中的开发和交付方式。容器可以在自己的隔离环境中快速部署和运行每个软件，而无需构建单独的虚拟机 (VM)。

大多数人可能很少考虑容器是如何在幕后工作的，但我认为了解底层技术很重要——它有助于为我们的决策过程提供信息。就个人而言，完全理解某事是如何运作的只会让我开心！

## 什么是命名空间？

命名空间从 2002 年左右开始成为 Linux 内核的一部分，随着时间的推移，更多的工具和命名空间类型被添加了进来。然而，真正的容器支持仅在 2013 年才添加到 Linux 内核中。这就是使命名空间真正有用并将它们带给大众的原因。

但究竟什么是命名空间？[这是来自维基百科](https://en.wikipedia.org/wiki/Linux_namespaces)的冗长定义：

“命名空间是 Linux 内核的一项功能，它对内核资源进行分区，以便一组进程看到一组资源，而另一组进程看到另一组资源。”

换句话说，命名空间的关键特性是它们将进程彼此隔离。在运行许多不同服务的服务器上，将每个服务及其相关进程与其他服务隔离意味着更改的爆炸半径更小，安全相关问题的占用空间也更小。大多数情况下，隔离服务符合[Martin Fowler](https://martinfowler.com/articles/microservices.html)所描述的微服务架构风格。

在开发过程中使用容器为开发人员提供了一个看起来和感觉就像一个完整的 VM 的隔离环境。不过，它不是虚拟机——它是在某处服务器上运行的进程。如果开发人员启动两个容器，则在某处的单个服务器上运行两个进程——但它们是相互隔离的。

### 命名空间的类型

在 Linux 内核中，有不同类型的命名空间。每个命名空间都有自己独特的属性：

- 用户[命名空间](https://man7.org/linux/man-pages/man7/user_namespaces.7.html)有自己的一组用户 ID 和组 ID，用于分配给进程。特别是，这意味着一个进程可以`root`在其用户命名空间中拥有特权，而无需在其他用户命名空间中拥有它。
- [进程 ID (PID) 命名空间](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html)将一组 PID 分配给独立于其他命名空间中的 PID 集的进程。在新命名空间中创建的第一个进程的 PID 为 1，子进程被分配后续的 PID。如果一个子进程是用它自己的 PID 命名空间创建的，那么它在该命名空间中的 PID 为 1，在父进程的命名空间中它的 PID。请参阅[下面](https://www.nginx.com/blog/what-are-namespaces-cgroups-how-do-they-work/#pid-namespaces)的示例。
- [网络命名空间](https://man7.org/linux/man-pages/man7/network_namespaces.7.html)有一个独立的网络堆栈：它自己的私有路由表、一组 IP 地址、套接字列表、连接跟踪表、防火墙和其他与网络相关的资源。
- [挂载命名空间](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html)有一个独立的挂载点列表，命名空间中的进程可以看到这些挂载点。这意味着您可以在挂载命名空间中挂载和卸载文件系统，而不会影响主机文件系统。
- [进程间通信 (IPC) 命名空间](https://man7.org/linux/man-pages/man7/ipc_namespaces.7.html)有自己的 IPC 资源，例如POSIX[消息队列](https://man7.org/linux/man-pages/man7/mq_overview.7.html)。
- [UNIX 分时 (UTS) 命名空间](https://man7.org/linux/man-pages/man7/uts_namespaces.7.html)允许单个系统对不同的进程具有不同的主机名和域名。

### 父子 PID 命名空间示例

在下图中，有三个 PID 命名空间——一个父命名空间和两个子命名空间。在父命名空间中，有四个进程，命名为**PID1**到**PID4**。这些是正常的进程，它们都可以互相看到并共享资源。

**在父命名空间中具有PID2**和**PID3**的子进程也属于它们自己的 PID 命名空间，它们的 PID 为 1。从子命名空间内，**PID1**进程无法看到外部的任何内容。例如，两个子命名空间中的**PID1**都看不到父命名空间中的**PID4**。

这提供了不同命名空间内（在这种情况下）进程之间的隔离。

[![表示在三个 PID 命名空间中运行的四个进程（一个父进程和两个子进程）的图表](https://www.nginx.com/wp-content/uploads/2022/07/Namespaces-cgroups_PID-namespaces.png)](https://www.nginx.com/wp-content/uploads/2022/07/Namespaces-cgroups_PID-namespaces.png)

### 创建命名空间

掌握了所有这些理论，让我们通过实际创建一个新的命名空间来巩固我们的理解。Linux[`unshare`](https://man7.org/linux/man-pages/man1/unshare.1.html)命令是一个很好的起点。手册页表明它完全符合我们的要求：

```
NAME
          unshare - run program in new name namespaces
```

我目前以普通用户身份登录`svk`，它有自己的用户 ID、组等，但没有`root`权限：

```
svk $ id
uid=1000(svk) gid=1000(svk) groups=1000(svk) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c.1023
```

现在我运行以下`unshare`命令来创建一个具有自己的用户和 PID 命名空间的新命名空间。我将`root`用户映射到新的命名空间（换句话说，我`root`在新的命名空间中拥有特权），挂载一个新的 proc 文件系统，并`bash`在新创建的命名空间中 fork 我的进程（在本例中为 ）。

```
svk $ unshare --user --pid --map-root-user --mount-proc --fork bash
```

（对于那些熟悉容器的人来说，这与在运行的容器中发出命令的作用相同。）`<*runtime>* exec -it <*image>* /bin/bash`

该`ps` `-ef`命令显示有两个进程正在运行——`bash`以及`ps`命令本身——并且该`id`命令确认我`root`在新的命名空间中（这也由更改的命令提示符指示）：

```
root # ps -ef
UID         PID     PPID  C STIME TTY        TIME CMD
root          1        0  0 14:46 pts/0  00:00:00 bash
root         15        1  0 14:46 pts/0  00:00:00 ps -ef
root # id
uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c.1023
```

需要注意的关键是我只能看到命名空间中的两个进程，而看不到系统上运行的任何其他进程。我完全隔离在自己的命名空间中。

### 从外部查看命名空间

虽然我无法从命名空间中看到其他进程，但使用[`lsns`](https://man7.org/linux/man-pages/man8/lsns.8.html)(list namespaces) 命令，我可以从父命名空间（新命名空间之外）的角度列出所有可用的命名空间并显示有关它们的信息。

输出显示了三个命名空间——类型`user`、、`mnt`和`pid` ——它们对应于`unshare`我上面运行的命令中的参数。从这个外部角度来看，每个命名空间都以 user 身份运行，而`svk`不是`root`，而在命名空间内部，进程以 身份运行`root`，可以访问所有预期的资源。（为了便于阅读，输出分为两行。）

```
root # lsns --output-all | head -1; lsns --output-all | grep svk
        NS TYPE   PATH                   NPROCS    PID   PPID ...
4026532690 user   /proc/97964/ns/user         2  97964  97944 ...                
4026532691 mnt    /proc/97964/ns/mnt          2  97964  97944 ...        
4026532692 pid    /proc/97965/ns/pid          1  97965  97964 ...

  ... COMMAND                                                       UID USER               
  ... unshare --user --map-root-user --fork –pid --mount-proc bash  1000 svk               
  ... unshare --user --map-root-user --fork –pid --mount-proc bash  1000 svk               
  ... bash                                                          1000 svk
```

### 命名空间和容器

命名空间是构建容器的技术之一，用于强制资源隔离。我们已经展示了如何手动创建命名空间，但是像[Docker](https://www.docker.com/)、[rkt](https://github.com/rkt/rkt)和[podman](https://podman.io/)这样的容器运行时通过代表您创建命名空间使事情变得更容易。同样，NGINX 单元中的[`isolation`应用程序对象创建命名空间和 cgroup。](https://unit.nginx.org/configuration/#process-isolation)

## 什么是 cgroup？

控制组 (cgroup) 是一种 Linux 内核功能，它限制、说明和隔离进程集合的资源使用情况（CPU、内存、磁盘 I/O、网络等）。

Cgroups 提供以下功能：

- **资源限制** ——您可以配置一个 cgroup 来限制一个进程可以使用多少特定资源（例如内存或 CPU）。
- **优先级** - 当存在资源争用时，您可以控制一个进程与另一个 cgroup 中的进程相比可以使用多少资源（CPU、磁盘或网络）。
- **会计** – 在 cgroup 级别监控和报告资源限制。
- **控制** - 您可以使用单个命令更改 cgroup 中所有进程的状态（冻结、停止或重新启动）。

所以基本上你使用 cgroups 来控制一个进程或一组进程可以访问或使用多少给定的关键资源（CPU、内存、网络和磁盘 I/O）。Cgroups 是容器的关键组件，因为通常在一个容器中运行多个进程需要一起控制。在 Kubernetes 环境中，cgroups 可用于实现Pod 级别的[资源请求和限制以及相应的 QoS 类。](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/node-allocatable.md#recommended-cgroups-setup)

下图说明了当您将特定百分比的可用系统资源分配给 cgroup（在本例中为**cgroup‑1**）时，剩余百分比如何可用于系统上的其他 cgroup（和单个进程）。

[![图表显示了将系统资源的百分比分配给 cgroup 如何使剩余百分比可用于系统上的其他 cgroup（和单个进程）](https://www.nginx.com/wp-content/uploads/2022/07/Namespaces-cgroups_resource-limits.png)](https://www.nginx.com/wp-content/uploads/2022/07/Namespaces-cgroups_resource-limits.png)

### Cgroup 版本

根据[维基百科](https://en.wikipedia.org/wiki/Cgroups)，cgroups 的第一个版本在 2007 年末或 2008 年初并入 Linux 内核主线，“cgroups-v2 的文档首次出现在 [the] Linux kernel ... [in] 2016”。在第 2 版的众多变化中，最大的变化是大大简化的树形架构、cgroup 层次结构中的新特性和接口，以及更好地适应“无根”容器（具有非零 UID）。

v2 中我最喜欢的新界面是用于[压力失速信息 (PSI)](https://www.kernel.org/doc/html/latest/accounting/psi.html)。它以比以前更精细的方式提供了对每个进程内存使用和分配的洞察（这超出了本博客的范围，但这是一个非常酷的主题）。

### 创建一个 cgroup

以下命令创建了一个 v1 cgroup（您可以通过路径名格式来判断）`foo`，并将其内存限制设置为 50,000,000字节 (50 MB)。

```
root # mkdir -p /sys/fs/cgroup/memory/foo
root # echo 50000000 > /sys/fs/cgroup/memory/foo/memory.limit_in_bytes
```

现在我可以为 cgroup 分配一个进程，从而对它施加 cgroup 的内存限制。我编写了一个名为 的 shell 脚本`test.sh`，它打印`cgroup` `testing` `tool`到屏幕上，然后什么也不做。就我的目的而言，它是一个持续运行的过程，直到我停止它为止。

我从`test.sh`后台开始，它的 PID 报告为 2428。脚本产生它的输出，然后我通过将进程的 PID 管道分配到 cgroup 文件**/sys/fs/cgroup/memory/foo/cgroup.procs**来将进程分配给 cgroup 。

```
root # ./test.sh &
[1] 2428
root # cgroup testing tool
root # echo 2428 > /sys/fs/cgroup/memory/foo/cgroup.procs
```

为了验证我的进程实际上是否受我为 cgroup 定义的内存限制的影响`foo`，我运行以下`ps`命令。该`-o` `cgroup`标志显示指定进程 (2428) 所属的 cgroup。输出确认其内存 cgroup 为`foo`。

```
root # ps -o cgroup 2428
CGROUP
12:pids:/user.slice/user-0.slice/\
session-13.scope,10:devices:/user.slice,6:memory:/foo,...
```

默认情况下，操作系统会在进程超过其 cgroup 定义的资源限制时终止该进程。

## 结论

命名空间和 cgroup 是容器和现代应用程序的构建块。当我们将应用程序重构为更现代的架构时，了解它们的工作原理很重要。

命名空间提供系统资源的隔离，而 cgroups 允许对这些资源进行细粒度控制和实施限制。

容器不是您可以使用命名空间和 cgroup 的唯一方式。命名空间和 cgroup 接口内置在 Linux 内核中，这意味着其他应用程序可以使用它们来提供分离和资源约束。

了解有关[NGINX 单元](https://unit.nginx.org/)的更多信息并[下载源代码](https://unit.nginx.org/installation/#obtaining-sources)以亲自尝试。

## 参考

文本转载于https://www.nginx.com/blog/what-are-namespaces-cgroups-how-do-they-work/#pid-namespaces