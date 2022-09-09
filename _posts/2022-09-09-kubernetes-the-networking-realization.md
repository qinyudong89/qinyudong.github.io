---
title: "Kubernetes的网络实现"
date: 2022-09-09T11:20:30-04:00
categories:
- blog
  tags:
- kubernetes
- pause
---

- Container 与 Container 之间如何通信？
- Pod 与 Pod 之间如何通信？
- Pod 与 Service 之间如何通信？
- Cluster 与 外部组件如何通信？

### **Container 与 Container 之间通信**

同一个 Pod 内的容器（Pod 内的容器不会跨宿主机）共享同一个网络命名空间，共享同一个 Linux 协议栈。因此它们之间只需通过 localhost 就可以通信。

这样做的好处就是简单、安全和高效，也能减少将已存在的程序从物理机或虚拟机中移植到容器下运行的难度。

![创建 Pod 时，首先容器运行时为容器创建一个网络命名空间。](https://learnk8s.io/a/f240df957191a80f14f3dff4e03a1f04.svg)

### **Pod 与 Pod 之间的通信**



##### 同一个 Node 上 Pod 之间的通信

![由于目标不是命名空间中的容器之一，Pod-A 将数据包发送到其默认接口 eth0。 该接口绑定到 veth 对的一端并用作隧道。 这样，数据包就会被转发到节点上的根命名空间。](https://learnk8s.io/a/91902010f7202b086f3e78260af25446.svg)

从上图我们可以看出，由于Pod A 与 Pod B 关联在同一个 docker0 网桥上，地址网段相同，且默认路由都是 docker0 的地址。所以他们之间的通信都会默认发送到 docker0 网桥上，由 docker0 网桥直接中转。

##### 不同一个 Node 上 Pod 之间的通信



### **Pod 与 Service 之间的通信**



### **Cluster 与 外部组件的通信**