+++
title = 'Java 为什么总是避免不了 NPE？从 Nullability 到 NullAway'
date = 2026-09-21T00:00:00+08:00
draft = true
description = '介绍如何使用 JSpecify 描述 Java API 的 Nullability Contract，并通过 NullAway 在编译期发现潜在的空指针问题。'
categories = ['java']
image = 'cover.png'
tags = ['java', 'null-safety', 'jspecify', 'nullaway', 'spring-boot']
+++

{{< details summary="文章更新记录" >}}
- v1.0：介绍 Java Nullability、JSpecify 与 NullAway 的基本用法。
{{< /details >}}

Java 开发中有一个很常见的问题：方法签名看起来完全正常，但运行时却突然抛出 `NullPointerException`。

例如：

```java
interface OrderRepository {
    Order findById(String id);
}
```

调用方无法仅通过这个方法签名判断两件事：

1. `id` 是否允许传入 `null`；
2. 查询不到订单时，返回值是否可能为 `null`。

这种信息缺失，正是 Java 中许多 NPE 的根源。

Spring Framework 7 和 Spring Boot 4 开始采用基于 JSpecify 的 Nullability 标注。本文不讨论如何彻底消灭 NPE，而是介绍一套更现实的方案：

> 使用 JSpecify 描述 Nullness Contract，再使用 NullAway 在编译期检查这些 Contract。

## 背景：NPE 的问题不只是 `null`

Java 允许引用类型拥有 `null` 值，这是历史兼容性带来的结果。

问题在于，传统 Java 类型系统无法直接表达下面两种不同的含义：

```java
String name;
```

它可能表示：

- `name` 一定不为 `null`；
- `name` 允许为 `null`；
- 编写这个 API 的人根本没有说明。

这三种情况在 Java 的方法签名中看起来完全一样。

相比之下，其他语言可以在类型层面表达这类信息：

```kotlin
String     // 不允许为 null
String?    // 允许为 null
```

```typescript
string;
string | null;
```

Rust 则使用：

```rust
Option<String>
```

来表示一个值可能不存在。

Java 很难直接改变现有引用类型的语义。因为 Java 生态中存在大量旧代码、第三方库和二进制依赖，贸然修改类型规则会破坏兼容性。

本质是 **nullability 没有被显式表达在 API 中**。因为仅通过方法签名，往往无法判断一个值是否允许为 `null`。

## Nullability 的三个状态

一般来说，Java 中一个值的 Nullability 状态有两种：“非空”和“可为空”，但是这还忽略了一种最常见的中间态：“**未指定是否能为空”。

Java 中的 Nullability 可以简单理解为三个状态：

| 状态          | 含义                      |
| ------------- | ------------------------- |
| `Non-null`    | 明确不允许为 `null`       |
| `Nullable`    | 明确允许为 `null`         |
| `Unspecified` | 没有说明是否允许为 `null` |

其中最容易混淆的是 `Unspecified` 和 `Nullable`。

```text
Nullable    = 明确知道这里允许 null
Unspecified = 不知道这里是否允许 null
```

例如，一个没有任何注解的旧方法：

```java
Order findById(String id);
```

并不一定表示它返回的订单不可能为 `null`。它也可能只是没有声明自己的 Nullability Contract。

这也是 Java 生态中进行渐进式迁移时必须保留 `Unspecified` 状态的原因。如果把所有未标注代码都直接当成 `Nullable`，检查结果会非常宽松；如果把所有未标注代码都直接当成 `Non-null`，大量遗留代码又会立即产生错误。

## 为什么不直接使用 Optional？

看到这里，有人可能会想到：

> 既然返回值可能为空，那直接使用 `Optional<Order>` 不就可以了吗？

`Optional` 确实是一种很好的业务建模方式，但它不能完全替代 Nullability 标注。

首先，`Optional` 会改变 API 的签名：

```java
Optional<Order> findById(String id);
```

这适合新设计的业务接口，但不一定适合已有的大量 API。

其次，`Optional` 主要解决的是“返回值可能不存在”的问题，而 Nullability 标注还可以描述：

- 方法参数是否允许为 `null`；
- 字段是否允许为 `null`；
- 泛型中的元素是否允许为 `null`；
- 数组元素是否允许为 `null`；
- 第三方 API 或遗留 API 的边界。

因此，两者的关系不是二选一：

- Nullability 注解用于描述 API 的空值契约；
- `Optional` 用于表达业务中的“值不存在”。

## JSpecify：描述 Nullness Contract

JSpecify 是 Java 生态中的 Nullness Specification，提供了一组用于描述 Nullability 的标准注解。

> 而且随便提一嘴，Spring Boot 4.0 也将正式引入 JSpecify 作为 Spring 生态统一的 Null Safety 规范。

### 使用 `@NullMarked` 设置默认规则

通常可以在包级别添加 `@NullMarked`：

```java
@NullMarked
package com.example.order;

import org.jspecify.annotations.NullMarked;
```

它表示：

> 在这个作用域内，未显式标注的引用类型默认视为 non-null。

因此：

```java
@NullMarked
public interface OrderRepository {
    Order findById(String id);
}
```

可以理解为：

```java
public interface OrderRepository {
    @NonNull Order findById(@NonNull String id);
}
```

这里的 `@NonNull` 是为了帮助理解默认语义，实际代码通常不需要到处显式添加它。

### 使用 `@Nullable` 标记例外

如果一个返回值确实可能为 `null`，就应该显式标注：

```java
@NullMarked
public interface OrderRepository {

    @Nullable
    Order findById(String id);
}
```

这时，方法契约就清晰了：

- `id` 不允许为 `null`；
- 查询结果允许为 `null`。

对于调用方而言，这个 API 不再是“猜测返回值是否为空”，而是有明确的契约可以遵循。

## NullAway：在编译期检查契约

JSpecify 负责描述规则，但它本身不会自动检查代码。

NullAway 是一个基于 Error Prone 的静态分析工具，可以在编译阶段检查 Nullability 问题。由 Uber 开源，github 上也可以查看。

它能够发现类似下面的代码：

```java
@NullMarked
public class OrderService {

    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }

    public String getOrderStatus(String orderId) {
        Order order = repository.findById(orderId);

        return order.getStatus();
    }
}
```

因为 `findById` 的返回值被标记为 `@Nullable`，所以 `order` 可能为 `null`。直接调用：

```java
order.getStatus()
```

就存在 NPE 风险。

NullAway 会在编译阶段报告这个问题，而不是等到代码部署后才由线上请求触发。

{{< figure src="1.png" alt="JSpecify 描述 Nullness Contract，NullAway 在编译期检查并发现 NPE 风险" caption="JSpecify 负责描述空值契约，NullAway 负责在编译期检查调用方是否正确处理可能为 null 的值" >}}

### Maven 中如何接入 NullAway

项目首先需要引入 JSpecify：

```xml
<dependency>
    <groupId>org.jspecify</groupId>
    <artifactId>jspecify</artifactId>
    <version>1.0.0</version>
</dependency>
```

NullAway 通常通过 Error Prone 接入 Java 编译过程。核心配置思路如下：

```text
-Xplugin:ErrorProne
-Xep:NullAway:ERROR
-XepOpt:NullAway:OnlyNullMarked=true
```

其中：

- `NullAway:ERROR`：发现 Nullability 问题时让编译失败；
- `OnlyNullMarked=true`：只检查显式标记为 `@NullMarked` 的代码；
- 通过 `@NullMarked` 控制检查范围，方便旧项目渐进式迁移。

不同 JDK、Maven Compiler Plugin 和 Error Prone 版本的配置方式可能不同，实际项目应以 NullAway 官方 Maven 示例为准。

建议不要一开始就让 NullAway 检查整个遗留项目，而是先从新代码或核心包开始：

```java
@NullMarked
package com.example.order;
```

## 一个完整的订单查询示例

### Repository 层表达真实契约

```java
@NullMarked
public interface OrderRepository {

    @Nullable
    Order findById(String id);
}
```

实体对象本身的字段默认为 non-null：

```java
@NullMarked
public record Order(
        String id,
        String status
) {
}
```

### Service 层处理空结果

Service 层可以将允许为空的底层结果转换成 `Optional`：

```java
@NullMarked
public class OrderService {

    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }

    public Optional<Order> findOrder(String orderId) {
        return Optional.ofNullable(repository.findById(orderId));
    }

    public String getOrderStatus(String orderId) {
        return findOrder(orderId)
                .map(Order::status)
                .orElse("NOT_FOUND");
    }
}
```

这样，Service 层对外暴露的是一个更明确的业务结果：

```java
Optional<Order>
```

调用方不需要猜测返回值是否为空，也不需要依赖运行时异常来发现问题。

### Controller 层处理业务结果

```java
@NullMarked
@RestController
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping("/orders/{id}/status")
    public ResponseEntity<String> getStatus(@PathVariable String id) {
        return ResponseEntity.ok(orderService.getOrderStatus(id));
    }
}
```

在这个例子中，每一层的职责比较清晰：

- Repository 用 `@Nullable` 描述底层查询可能没有结果；
- Service 将底层的 `null` 转换为 `Optional`；
- Controller 只处理 Service 提供的业务结果。

{{< figure src="2.png" alt="订单查询中 Controller、Service、Repository 与数据库之间的 Nullability 边界" caption="Repository 暴露 Nullable，Service 负责转换，Controller 消费明确的业务结果" >}}

## 遗留项目如何渐进式迁移？

在已有项目中，最现实的做法通常不是一次性修改所有代码，而是分阶段进行。

### 第一阶段：从新代码开始

给新建的包添加 `@NullMarked`：

```java
@NullMarked
package com.example.order;
```

这样可以保证新代码默认采用 non-null 语义。

### 第二阶段：优先处理公共 API

公共接口、Service、Repository 和对外 SDK 的 Nullability 信息最有价值。

因为这些位置决定了空值信息如何在模块之间传播。

### 第三阶段：处理旧代码边界

对于尚未迁移的旧代码，可以暂时保留 `Unspecified` 语义，或者使用 `@NullUnmarked` 明确退出外层的 `@NullMarked` 作用域。

这并不意味着旧代码一定安全，而是承认：

> 这部分代码目前还没有足够的 Nullability 信息。

### 第四阶段：关注外部边界

数据库、JSON 反序列化、反射、第三方库和代码生成工具都可能引入 `null`。

静态检查器只能依据代码中的契约进行推理。如果契约本身写错了，或者外部系统没有遵循契约，运行时仍然可能出现 NPE。

## 这套方案解决什么，不能解决什么？

JSpecify + NullAway 主要解决的是：

- API 没有表达参数和返回值是否允许为 `null`；
- 调用方忘记检查 `@Nullable` 返回值；
- 非空值被错误地传递给可能为空的变量；
- Nullability 信息无法在模块之间传播。

但它不能保证所有 NPE 消失。

以下问题仍然需要额外处理：

- 反射生成的对象；
- 数据库字段与实体定义不一致；
- 反序列化结果不符合预期；
- 第三方库缺少 Nullability 标注；
- 错误的 `@Nullable` 或 `@NullMarked` 声明；
- 通过强制转换、原始类型或其他不安全方式绕过检查。

因此，JSpecify + NullAway 更准确的定位是：

> 把一部分原本只能在运行时发现的空值错误，提前转化为编译期错误。

所以我的建议是：在开启新 Java 项目时，推荐使用 JSpecify + NullAway，在 Java 中享受一下现代编程语言的特性！

## 总结

Java 的 NPE 问题，不只是因为语言允许 `null`，更重要的是大量 API 没有明确表达 Nullability。

JSpecify 和 NullAway 分工不同：

- **JSpecify**：描述参数、返回值、字段和泛型的 Nullness Contract；
- **NullAway**：在编译期检查代码是否遵守这些 Contract；
- **Optional**：用于表达业务层面的“值不存在”。

在新项目中，可以通过 `@NullMarked` 建立默认 non-null 语义，再用 `@Nullable` 标注少数允许为空的场景。

在旧项目中，则应该从新包、公共 API 和核心业务链路开始，逐步扩大检查范围。

这套方案不是给 Java 增加了一个新的 `?` 语法，也不是运行时防护机制。它更像是为 Java 现有类型系统补充了一份可检查的契约，让开发者不必再依赖文档、经验和线上 NPE 来猜测一个值到底能不能为 `null`。

## 参考资料

- [Null Safety in Java with JSpecify and NullAway - Spring I/O 2025](https://www.youtube.com/watch?v=5Lbxq6LP7FY)
- [JSpecify 官方文档](https://jspecify.dev/)
- [Spring Framework Null Safety](https://docs.spring.io/spring-framework/reference/core/null-safety.html)
- [NullAway 官方文档](https://github.com/uber/NullAway)
