---
title: "Promethus 的数据模型"
date: 2023-02-12T08:20:30-04:00
categories:
  - blog
tags:
  - Promethus
  - Time Series Data
---

Prometheus 存储的所有数据都是**时间序列数据**（**Time Serie Data**，简称时序数据）。时序数据是具有时间戳的数据流，该数据流属于某个度量指标（Metric）和该度量指标下的多个标签（Label）。

![img](https://img2022.cnblogs.com/blog/1601821/202207/1601821-20220708232146655-390217774.png)

每个 Metric name 代表了一类的指标，他们可以携带不同的 Labels，每个 Metric name + Label 组合成代表了一条时间序列的数据。

在 Prometheus 的世界里面，所有的数值都是64bit的。每条时间序列里面记录的其实就是 **64 bit timestamp (时间戳) + 64 bit value (采样值)**。

- **Metric name（指标名称）**：该名字应该具有语义，一般用于表示 metric 的功能，例如：http_requests_total, 表示 http 请求的总数。其中，metric 名字由 ASCII 字符，数字，下划线，以及冒号组成，且必须满足正则表达式 [a-zA-Z_:][a-zA-Z0-9_:]*。
- **Lables（标签）**：使同一个时间序列有了不同维度的识别。例如 http_requests_total{method=“Get”} 表示所有 http 请求中的 Get 请求。当 method=“post” 时，则为新的一个 metric。标签中的键由 ASCII 字符，数字，以及下划线组成，且必须满足正则表达式 [a-zA-Z_:][a-zA-Z0-9_:]*。
- **timestamp(时间戳)**：数据点的时间，表示数据记录的时间。
- **Sample Value（采样值）**：实际的时间序列，每个序列包括一个 float64 的值和一个毫秒级的时间戳。

例如图上的数据：

```bash
http_requests_total{status="200",method="GET"}
http_requests_total{status="404",method="GET"}
```

根据上面的分析，时间序列的存储似乎可以设计成 key-value 存储的方式（基于BigTable）。

![img](https://img2022.cnblogs.com/blog/1601821/202207/1601821-20220708232200733-757921525.png)

进一步拆分，可以像下面这样子：

![img](https://img2022.cnblogs.com/blog/1601821/202207/1601821-20220708232212502-2062186293.png)

上图的第二条样式就是现在 Prometheus 内部的表现形式了，__name__ 是特定的 label 标签，代表了metric name。

再回顾一下Prometheus的整体流程：

![img](https://img2022.cnblogs.com/blog/1601821/202207/1601821-20220708232224367-411304706.png)

上面提到了K-V存储，当然是使用了 LevelDB 的引擎，它的特点是顺序读写性能非常高，这是非常符合时间序列的存储的。



### 小结

Prometheus 之所以选择使用时序数据是因为它非常适合用于监控和告警。与其他数据类型相比，时序数据有以下优点：

1. 易于收集：通过采集服务器或容器上的指标，可以快速获取时序数据。
2. 直观：可以通过图表来可视化时序数据，这样可以更容易地看出系统的性能趋势和问题。
3. 可查询：通过使用 Prometheus 的语言，可以方便地查询时序数据，从而确定系统的表现情况。
4. 易于存储：Prometheus 可以高效地存储大量的时序数据，并对其进行快速的检索和分析。

因此，使用时序数据可以有效地监控系统并预测可能出现的问题。此外，Prometheus 还支持多种数据类型，例如，counter、gauge、histogram 等，这些数据类型都是基于时序数据，可以帮助用户更好地监控系统。

### 参考

[https://www.cnblogs.com/liugp/p/16459922.html#%E4%B8%80prometheus%E7%AE%80%E4%BB%8B]