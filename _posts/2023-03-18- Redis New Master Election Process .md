---
title: "Sentinel 模式下新的主节点选举流程"
date: 2023-03-18T08:20:30-04:00
categories:
  - blog
tags:
  - Redis
  - Sentinel
---
Sentinel Leader 会将已下线主节点的所有从节点中选举出最佳的从节点，当做新的主节点。

1. 从节点要正常在线
2. 从节点最近 5 秒内与 Sentinel Leader 有过成功通信
3. 从节点从节点 10 秒内与客观下线的主节点有过通信
4. 从节点优先级最高优先
5. 从节点复制偏移量最大优先
6. 从节点 runid 最小优先

### 参考：

Redis 设计与实现 第 16 章dubbo.apache.org/zh-cn/overview/mannual/java-sdk/reference-manual/config/principle/