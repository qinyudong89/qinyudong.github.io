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

我们从上面的架构图可以看出 Prometheus 的主要模块包含:Server, Exporters, Pushgateway, PromQL, Alertmanager, WebUI 等。我们逐一认识一下各个模块的功能作用。


#### 模块介绍

- **Retrieval**是负责定时去暴露的目标页面上去抓取采样指标数据。
- **Storage** 是负责将采样数据写入指定的时序数据库存储。
- **PromQL** 是Prometheus提供的查询语言模块。可以和一些webui比如grfana集成。
- **Jobs / Exporters**:Prometheus 可以从 Jobs 或 Exporters 中拉取监控数据。Exporter 以 Web API 的形式对外暴露数据采集接口。
- **Prometheus Server**:Prometheus 还可以从其他的 Prometheus Server 中拉取数据。
- **Pushgateway**:对于一些以临时性 Job 运行的组件，Prometheus 可能还没有来得及从中 pull 监控数据的情况下，这些 Job 已经结束了，Job 运行时可以在运行时将监控数据推送到 Pushgateway 中，Prometheus 从 Pushgateway 中拉取数据，防止监控数据丢失。
- **Service discovery**:是指 Prometheus 可以动态的发现一些服务，拉取数据进行监控，如从DNS，Kubernetes，Consul 中发现, file_sd 是静态配置的文件。
- **AlertManager**:是一个独立于 Prometheus 的外部组件，用于监控系统的告警，通过配置文件可以配置一些告警规则，Prometheus 会把告警推送到 AlertManager。

#### 工作流程

- **Prometheus server** 定期从配置好的 jobs 或者 exporters 中拉 metrics，或者接收来自 Pushgateway 发过来的 metrics，或者从其他的 Prometheus server 中拉 metrics。
- **Prometheus server** 在本地存储收集到的 metrics，并运行已定义好的 alert.rules，记录新的时间序列或者向 Alertmanager 推送警报。
- **Alertmanager** 根据配置文件，对接收到的警报进行处理，发出告警。
在图形界面中，可视化采集数据。

### 参考

https://www.cnblogs.com/liugp/p/16459922.html#%E4%B8%80prometheus%E7%AE%80%E4%BB%8B
