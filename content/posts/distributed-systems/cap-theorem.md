+++
title = 'CAP 定理：分布式系统的权衡'
date = 2026-03-16T13:00:00+08:00
draft = false
description = '理解分布式系统中的 CAP 定理及其实践应用'

categories = ['distributed-systems']
tags = ['distributed-systems', 'consistency', 'availability', 'partition-tolerance', 'theory']
series = ''
seriesWeight = 0
+++

## CAP 定理

CAP 定理指出，分布式系统最多只能同时满足以下三个特性中的两个：

### 一致性 (Consistency)

所有节点在同一时间看到相同的数据。

### 可用性 (Availability)

每个请求都能得到响应（成功或失败），保证系统持续服务。

### 分区容错性 (Partition Tolerance)

系统在网络分区的情况下仍能继续运行。

## 三种权衡组合

| 组合 | 说明 | 典型系统 |
|------|------|----------|
| CA | 放弃分区容错 | 单机数据库（RDBMS） |
| CP | 放弃可用性 | HBase, MongoDB, Redis |
| AP | 放弃强一致性 | Cassandra, DynamoDB |

## BASE 理论

CAP 的替代方案：
- **Basically Available**：基本可用
- **Soft state**：软状态
- **Eventually consistent**：最终一致性

## 总结

在设计分布式系统时，需要根据业务场景选择合适的权衡：
- 金融交易：优先保证一致性（CP）
- 社交媒体：优先保证可用性（AP）
