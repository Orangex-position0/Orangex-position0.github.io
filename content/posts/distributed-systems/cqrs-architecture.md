+++
title = '使用 CQRS 处理读写分离架构'
date = 2026-05-05T10:00:00+08:00
draft = false
description = '介绍 CQRS 架构模式的核心思想、五种设计原则、三种演进层次，以及电商订单系统的完整实践案例'
categories = ['distributed-systems']
tags = ['CQRS', '架构模式', 'DDD', '读写分离', '事件驱动']
+++

## 背景 & 问题

在传统架构中，单个数据模型通常兼顾读和写的职责。但随着系统规模扩展，读和写操作在性能和扩展性上呈现出明显的不对称性：

- **数据表示不匹配**：写入时需要的字段结构与读取时需要的展示形式往往不同。例如，创建订单需要商品 ID 列表，但查询订单详情时用户希望看到商品名称和价格
- **锁竞争严重**：读操作和写操作共享同一份数据，复杂查询（join / 聚合 / 过滤）会长时间占用连接，阻塞写操作
- **性能无法兼顾**：写入追求强一致性和事务安全，读取追求高并发和低延迟，两者的优化方向天然矛盾

这些问题的本质是：**单一数据模型无法同时高效满足读与写需求**。

本文将介绍 CQRS（Command Query Responsibility Segregation）模式，说明它如何通过读写职责分离来解决上述问题，并给出从简单到复杂的三种演进方案和完整的电商订单系统案例。

### 前置知识

阅读本文需要了解以下概念（不熟悉也没关系，文中会配合上下文说明）：

- **领域驱动设计（DDD）**：一种以业务领域为核心组织代码的方法论，强调领域模型和业务规则的封装
- **领域模型（Domain Model）**：面向业务的对象模型，包含实体、值对象和聚合根，封装业务规则
- **领域事件（Domain Event）**：表示领域中发生的重要事情（如"订单已创建"），用于解耦不同模块之间的通信
- **聚合根（Aggregate Root）**：一组相关对象的入口点，保证业务规则的一致性边界

## 方案或原理

### CQRS 是什么

CQRS（Command Query Responsibility Segregation）是一种架构模式，核心思想是：**将修改数据的操作（Command）和读取数据的操作（Query）彻底解耦**，使用不同的模型、流程和存储来分别服务。

简单来说，就是让"写"和"读"各走各的路，各自优化。

### 优缺点

**优势：**
- 写操作聚焦强一致性和业务规则，读操作聚焦高并发和数据展示
- 读写可独立扩容，读模型可以使用缓存、搜索引擎等专门优化
- 降低单个模型的复杂度，职责更清晰

**代价：**
- 需要维护两套模型，开发成本和学习成本增加
- 读模型通常异步更新，存在数据延迟（最终一致性）
- 事件同步、消息队列等基础设施需要额外运维

### 适用场景

| 场景 | 说明 |
|------|------|
| 读多写少 | 用户频繁查询但写入较少，如电商订单查询 |
| 查询逻辑复杂 | 需要多表 join、聚合计算，如报表系统 |
| 读写性能要求差异大 | 写要求强一致，读要求低延迟高并发 |
| 业务逻辑复杂 | 领域规则复杂，不适合在查询中混入业务逻辑 |

> 如果你的系统读写都很简单，或者读远大于写但查询也不复杂，引入 CQRS 可能是过度设计。先评估实际需求，再决定是否采用。

### 五个设计原则

#### 1. 读写职责分离

- 写操作只负责**修改状态 + 执行业务规则**
- 读操作只负责**数据获取 + 展示优化**
- 禁止在 Query 中修改状态

目的：避免读路径被业务逻辑污染，也保证查询可独立优化。

#### 2. 一致性策略明确

- 写路径：强一致（通过事务和聚合保证）
- 读路径：最终一致（允许短暂延迟）

必须根据业务定义"可接受延迟范围"。例如：订单创建后 1 秒内可在查询中看到，这通常是可接受的。

目的：用"可控不一致"换取系统性能与扩展性。

#### 3. 模型分离

- **写模型（Domain Model）**：面向业务规则与一致性
- **读模型（Read Model）**：面向查询与展示，可以反范式化（去范式化）以提高查询效率

读模型不需要与数据库表结构一致，它只为查询服务。

目的：避免一个模型同时承担"复杂业务 + 高性能查询"两种职责。

#### 4. 存储分离

- 写存储：事务型数据库（如 MySQL），保证强一致性
- 读存储：缓存（如 Redis）/ 搜索引擎（如 ElasticSearch）/ 副本库

是否拆分存储取决于读压力与查询复杂度，并非所有场景都需要物理分离。

目的：让存储层也服务于不同的优化目标。

#### 5. 异步解耦

- 通过领域事件（Domain Event）连接读写模型
- 流程：写操作 → 发布事件 → 异步更新读模型（Projection）

需要接受最终一致性（存在延迟）。

目的：降低系统耦合，支持独立扩展读模型。

### CQRS 架构概览

```
┌─────────────┐     Command      ┌──────────────────┐     Event       ┌──────────────────┐
│   Client     │ ──────────────→ │  Command Handler  │ ─────────────→ │  Event Handler    │
│              │                  │  (写流程编排)      │               │  (Projection)     │
└─────────────┘                  └────────┬─────────┘               └────────┬─────────┘
                                          │                                  │
                                          ▼                                  ▼
                                  ┌──────────────┐                  ┌──────────────┐
                                  │  Write Store  │                  │  Read Store   │
                                  │  (MySQL)      │                  │  (Redis/ES)   │
                                  └──────────────┘                  └──────────────┘
                                                                            ▲
┌─────────────┐     Query        ┌──────────────────┐                     │
│   Client     │ ──────────────→ │  Query Handler    │ ───────────────────┘
│              │                  │  (读流程编排)      │
└─────────────┘                  └──────────────────┘
```

## 实现步骤

CQRS 并不是一步到位的，可以根据系统复杂度分三个层次渐进式引入。不必一上来就用最复杂的方案。

### Level 1：逻辑分离（单库）

**思路**：仍然使用单个数据库，只在代码层面分离读和写。

**做法**：
- 代码中分为 Command 对象和 Query 对象
- 写操作使用 Domain Model，读操作使用独立的 Read Model（DTO / VO）
- 共享同一个数据库，读模型可以通过 SQL 视图或简单映射得到

**适用场景**：系统初期，读写压力都不大，但希望代码职责清晰。

**优点**：改动最小，风险最低
**局限**：读和写仍然竞争同一份数据库资源

### Level 2：物理分离（双库）

**思路**：将数据库分为写库和读库，读库独立优化。

**做法**：
- 写库（MySQL 主库）：处理所有写操作，保证强一致性
- 读库（MySQL 从库 / Redis / ElasticSearch）：处理所有读操作
- 写库数据通过主从复制或同步机制同步到读库

**适用场景**：读多写少，查询逻辑较复杂，读操作需要独立扩容。

**优点**：读写不竞争数据库资源，读库可针对性优化
**局限**：需要维护数据同步机制，存在同步延迟

### Level 3：事件驱动（Event Sourcing + CQRS）

**思路**：写路径产生的状态变化抽象为领域事件（Domain Event），由 Event Handler 异步构建和更新 Read Model。

**做法**：
- 写操作修改 Domain Model 后，发布领域事件
- Event Handler 监听事件，将数据转换为读模型需要的格式，写入 Read Store
- 读操作直接查询 Read Store，完全不访问 Write Store

**适用场景**：业务逻辑复杂，需要完整的审计日志，或需要多个读模型服务不同查询场景。

**优点**：读写完全解耦，可灵活构建多个专用读模型
**局限**：架构最复杂，需要事件基础设施（消息队列），调试难度最高

### 三种层次对比

| 维度 | L1 逻辑分离 | L2 物理分离 | L3 事件驱动 |
|------|------------|------------|------------|
| 存储数量 | 1 个 | 2 个 | 2+ 个 |
| 代码改动 | 小 | 中 | 大 |
| 读写解耦程度 | 低 | 中 | 高 |
| 一致性 | 强一致 | 最终一致（秒级延迟） | 最终一致（毫秒~秒级延迟） |
| 基础设施要求 | 无 | 主从复制 / 同步工具 | 消息队列 + Event Store |
| 适用阶段 | 项目初期 | 系统成长期 | 大型复杂系统 |

## 示例 / 踩坑

下面通过一个电商订单系统，演示 CQRS Level 3（事件驱动）的完整实现。

### 业务场景

电商订单系统，核心需求：
- **写**：创建订单、支付、取消（要求强一致性和复杂业务规则校验）
- **读**：订单列表、订单详情（用户频繁查询，需要低延迟）
- **特点**：读多写少，查询涉及多表 join 和聚合计算

### CQRS 改造思路

- **写（Command）**：CreateOrder → OrderAggregate → MySQL
- **读（Query）**：GetOrderDetail → Redis / Read DB（预聚合数据）
- **连接**：OrderCreatedEvent → Event Handler 异步更新 Read Model

### 代码实现

#### Command：写路径的输入模型

```java
public class CreateOrderCommand {
    private Long userId;
    private List<Long> productIds;

    // 构造方法 / getter
}
```

#### Domain Model：核心领域对象

```java
public class Order {
    private Long id;
    private Long userId;
    private List<Long> productIds;
    private String status;

    public static Order create(Long userId, List<Long> productIds) {
        if (productIds == null || productIds.isEmpty()) {
            throw new IllegalArgumentException("商品不能为空");
        }

        Order order = new Order();
        order.userId = userId;
        order.productIds = productIds;
        order.status = "CREATED";
        return order;
    }
}
```

#### Command Handler：写流程编排

```java
public class CreateOrderHandler {

    private OrderRepository repository;
    private EventPublisher eventPublisher;

    public Long handle(CreateOrderCommand cmd) {
        // 1. 构建领域对象（业务规则校验在 create 方法中）
        Order order = Order.create(cmd.getUserId(), cmd.getProductIds());

        // 2. 持久化（事务保证强一致性）
        repository.save(order);

        // 3. 发布事件（异步更新读模型）
        eventPublisher.publish(new OrderCreatedEvent(order.getId()));

        return order.getId();
    }
}
```

#### Read Model：读模型（专为查询设计）

```java
public class OrderView {
    private Long orderId;
    private String status;
    private List<String> productNames;
    private BigDecimal totalPrice;

    // getter
}
```

注意读模型和写模型的区别：写模型存储 `productIds`（商品 ID），读模型存储 `productNames`（商品名称）和 `totalPrice`（总价）——这是面向查询优化的结果。

#### Query Handler：读流程编排

```java
public class GetOrderDetailHandler {

    private OrderReadRepository readRepository;

    public OrderView handle(Long orderId) {
        // 直接查缓存或 Read DB，无业务逻辑
        return readRepository.findById(orderId);
    }
}
```

#### Event Handler：投影器（Projection）

Projection 监听领域事件，将 Command Side 的数据转换为 Read Model 并持久化：

```java
public class OrderProjectionHandler {

    private OrderReadRepository readRepository;
    private ProductRepository productRepository;

    public void on(OrderCreatedEvent event) {

        List<String> productNames =
                productRepository.findNames(event.getProductIds());

        OrderView view = new OrderView();
        view.setOrderId(event.getOrderId());
        view.setStatus(event.getStatus());
        view.setProductNames(productNames);

        readRepository.save(view);
    }
}
```

### 常见问题与踩坑

**问题 1：读模型更新延迟导致用户看不到最新数据**

这是 CQRS 最常见的问题。用户刚创建订单，立刻查询却找不到。

解决方案：
- 前端在写操作成功后，短暂轮询或使用 WebSocket 等待读模型更新
- 在读模型未就绪时，回退到写库查询（Fallback 策略）
- 在 UI 上提示用户"数据正在同步中"

**问题 2：事件处理失败导致读写数据不一致**

Event Handler 处理事件时可能失败（网络抖动、数据库异常等），导致读模型缺失数据。

解决方案：
- 事件消费必须实现幂等性，重复消费不会产生副作用
- 使用消息队列的重试机制（如 RabbitMQ 的 Dead Letter Queue）
- 定期做数据对账，比对写库和读库的数据差异并修复

**问题 3：读模型结构变更困难**

业务变化导致读模型需要新增字段，但历史事件中不包含这些字段。

解决方案：
- 新增字段设置默认值，避免历史数据缺失
- 设计事件时采用"事件增量"模式，通过版本号管理字段变更
- 需要时可以发起全量重建（Rebuild）读模型

**问题 4：过度设计**

并非所有系统都需要 CQRS。如果读写逻辑简单、并发量不大，引入 CQRS 反而增加不必要的复杂度。

判断标准：当你的系统确实面临"读写性能无法兼顾"的瓶颈时，再考虑引入。

## 总结

本文介绍了 CQRS 模式的核心思想和实践方法，回顾要点：

1. **核心思路**：将 Command（写）和 Query（读）解耦，各自优化
2. **五个设计原则**：读写职责分离、一致性策略明确、模型分离、存储分离、异步解耦
3. **三种演进层次**：逻辑分离 → 物理分离 → 事件驱动，根据系统复杂度渐进式引入
4. **关键取舍**：用最终一致性换取读写性能和扩展性

CQRS 不是银弹，它解决的是"读写性能无法兼顾"这一类特定问题。如果你的系统面临这个瓶颈，可以从 Level 1（逻辑分离）开始，渐进式演进到更复杂的方案。