---
title: "Promethus 介绍"
date: 2023-02-12T08:20:30-04:00
categories:
  - blog
tags:
  - Promethus
---
### Prometheus 简介

[Prometheus](https://github.com/prometheus)是一个开源系统监控和警报工具包，最初由 [SoundCloud](https://soundcloud.com/)构建。Prometheus 将其指标收集并存储为时间序列数据，即指标信息与记录时的时间戳以及称为标签的可选键值对一起存储。



### Prometheus 的特点有哪些？

- 通过指标名称和标签(key/value对）区分的多维度、时间序列数据模型

- 灵活的查询语法 PromQL

- 不需要依赖额外的存储，一个服务节点就可以工作

- 利用http协议，通过pull模式来收集时间序列数据

- 需要push模式的应用可以通过中间件gateway来实现

- 监控目标支持服务发现和静态配置

- 支持各种各样的图表和监控面板组件



### Prometheus 的核心组件有哪些？

整个Prometheus生态包含多个组件，除了Prometheus server组件其余都是可选的

- **Prometheus Server**：主要的核心组件，用来收集和存储时间序列数据。
- **Client Library:**：客户端库，为需要监控的服务生成相应的 metrics 并暴露给 Prometheus server。当 Prometheus server 来 pull 时，直接返回实时状态的 metrics。
- **push gateway**：主要用于短期的 jobs。由于这类 jobs 存在时间较短，可能在 Prometheus 来 pull 之前就消失了。为此，这次 jobs 可以直接向 Prometheus server 端推送它们的 metrics。这种方式主要用于服务层面的 metrics，对于机器层面的 metrices，需要使用 node exporter。
- **Exporters**: 用于暴露已有的第三方服务的 metrics 给 Prometheus。
- **Alertmanager**: 从 Prometheus server 端接收到 alerts 后，会进行去除重复数据，分组，并路由到对收的接受方式，发出报警。常见的接收方式有：电子邮件，pagerduty，OpsGenie, webhook 等。
- 各种支持工具。



### Prometheus 的架构设计？

Prometheus 直接或通过一个用于短期作业的中间推送网关从检测作业中抓取指标。它在本地存储所有抓取的样本，并对这些数据运行规则，以聚合和记录现有数据的新时间序列或生成警报。[Grafana](https://grafana.com/)或其他 API 消费者可用于可视化收集的数据。

![普罗米修斯架构](https://prometheus.io/assets/architecture.png)


### 参考

https://www.cnblogs.com/liugp/p/16459922.html#%E4%B8%80prometheus%E7%AE%80%E4%BB%8B
