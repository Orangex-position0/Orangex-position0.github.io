+++
title = '纯函数带来的架构思考：Functional Core 与 Imperative Shell'
date = 2026-04-08T10:00:00+08:00
draft = false
description = '使用纯函数思想分离业务逻辑与副作用，通过 Functional Core / Imperative Shell 模式提升代码可测试性和可维护性'

categories = ['functional-programming']
tags = ['函数式编程', '纯函数', '架构设计', '重构', '可测试性']
+++

## 一、🎯 背景 & 问题：当业务逻辑遇上副作用

作者作为一个 Java 后端开发，经常需要写这样的函数，不知你也是否写过这样的函数：

> 一个 `createOrder` 方法，既要计算订单金额、应用折扣规则，还要保存到数据库、打印日志。

这看起来是个很正常的业务函数，但当你想给"折扣计算"写单元测试时，却发现**必须连接数据库**才能运行测试。

这种"业务逻辑 + 副作用"混在一起的模式，带来了三个核心问题：

| 问题 | 表现 |
|------|------|
| ❌难以测试 | 必须依赖数据库/网络等外部系统 |
| ❌难以预测 | 函数行为依赖外部状态，相同输入不一定得到相同输出 |
| ❌难以复用 | 逻辑绑死在 I/O 上，换个场景就得重写 |

⚠️ **本质原因**：**纯计算**和**副作用**混在一起，导致核心业务逻辑被 I/O 操作"绑架"。

💡 **读完本文，你将学到**：
- 纯函数的核心思想及其对架构设计的启发
- **Functional Core / Imperative Shell** 模式：如何将纯计算与副作用分离
- 通过 Java 和 Rust 两个案例，掌握具体的重构方法

---

## 二、🧩 方案或原理：纯函数与架构分离

### 什么是纯函数（Pure Function）

纯函数是函数式编程的核心概念，指**相同输入下始终返回相同输出，且不产生任何副作用**的函数。

两个关键特征：
- **结果可预测（Determinism）**：输入唯一决定输出，不依赖外部状态
- **无副作用（No Side Effects）**：不修改外部变量、不操作 I/O、不影响外部世界

```rust
// 纯函数：相同输入 → 相同输出，无副作用
fn add(a: i32, b: i32) -> i32 {
    a + b
}

// 纯函数：只依赖输入，返回新值，不修改原数据
fn to_uppercase(s: &str) -> String {
    s.to_uppercase()
}
```

纯函数天然适合以下场景：基础计算、数据转换、集合处理、参数校验。它们的共同点是——**只关心输入到输出的映射，不关心数据从哪来、到哪去**。

### 什么是副作用（Side Effects）

副作用指函数在执行过程中，除返回值外带来的其他影响：

| 副作用类型 | 带来的影响 |
|------------|-----------|
| 修改外部变量 / 全局状态 | 破坏可预测性，增加隐式依赖 |
| I/O 操作（文件、网络、数据库） | 不可控（延迟/失败），结果不可复现 |
| 控制台输出（print） | 引入环境依赖，测试需要 mock |
| 依赖外部状态（环境变量、配置） | 行为不再由输入决定 |
| 使用时间 / 随机数 | 相同输入得到不同输出 |

副作用本身不是"坏东西"——没有副作用的程序无法与真实世界交互。**问题在于，副作用和业务逻辑混在一起时，代码变得难以理解和测试。**

### 核心思想：纯计算与副作用分离

这就引出了本文的核心架构模式：**Functional Core / Imperative Shell**。

```
┌───────────────────────────────────────────┐
│         Imperative Shell                  │
│         （命令式外壳 - 负责副作用）         │
│                                           │
│  · 读取数据库 / 文件                       │
│  · 调用 Functional Core 完成业务计算       │
│  · 保存结果到数据库                        │
│  · 记录日志                               │
│                                           │
│         ┌─────────────────────────┐       │
│         │   Functional Core       │       │
│         │   （纯逻辑内核 - 只做计算）│       │
│         │                         │       │
│         │  · 业务规则计算          │       │
│         │  · 权限判断              │       │
│         │  · 数据转换              │       │
│         │  · 校验逻辑              │       │
│         │                         │       │
│         │  ✅ 无副作用             │       │
│         │  ✅ 可预测               │       │
│         │  ✅ 容易测试（无需 mock） │       │
│         └─────────────────────────┘       │
└───────────────────────────────────────────┘
```

**Functional Core（纯逻辑内核）**：
- 由纯函数组成，只负责业务计算
- 不访问数据库、不打印日志、不修改外部状态
- 相同输入 → 相同输出，可独立测试

**Imperative Shell（命令式外壳）**：
- 负责与外界交互（I/O、事务、日志）
- 调用 Functional Core 完成业务计算
- 包含副作用（不可避免），但不承担复杂业务逻辑

> 💡 **核心思想**：不是消除副作用，而是**隔离副作用**，保护核心逻辑的可测试性。

---

## 三、🛠️ 使用步骤：重构"脏函数"

### 重构模板

将一个"业务逻辑 + 副作用"混合的函数，拆分为 FC/IS 结构，只需三步：

1. **提取纯业务逻辑**：将计算、校验等逻辑提取为独立函数（只依赖输入参数）
2. **提取副作用逻辑**：将数据库、I/O、日志等操作留在 Shell 中
3. **Shell 调用 Core**：在副作用函数中调用纯业务函数

### 案例1：Java - 创建订单

#### 场景

- 输入：商品价格、数量、用户等级
- 输出：订单金额 + 折扣结果
- 同时需要：保存数据库 + 打日志

#### ❌ 传统写法：逻辑 + 副作用混在一起

```java
class OrderService {

    public double createOrder(double price, int quantity, int userLevel) {
        double total = price * quantity;

        // 折扣逻辑（业务计算）
        if (userLevel == 2) total *= 0.9;
        if (userLevel == 3) total *= 0.8;

        // 数据库写入（副作用）
        Database.save(total);

        // 日志（副作用）
        System.out.println("Order created: " + total);

        return total;
    }
}
```

**问题**：想测试"折扣计算"逻辑，必须先连接数据库。

#### ✅ 重构后：FC/IS 分离

**Step 1：提取 Functional Core（纯逻辑内核）**

```java
/**
 * 纯逻辑内核 - 只负责业务规则计算
 *
 * 特点：
 * - 只依赖输入参数
 * - 不访问数据库 / 不打印日志 / 不修改外部状态
 * - 相同输入 → 相同输出
 */
class OrderCore {

    /**
     * 计算订单最终价格（纯函数）
     */
    public static double calculateTotal(double price, int quantity, int userLevel) {
        double total = price * quantity;

        if (userLevel == 2) return total * 0.9;
        if (userLevel == 3) return total * 0.8;

        return total;
    }
}
```

**Step 2：构建 Imperative Shell（命令式外壳）**

```java
/**
 * 命令式外壳 - 负责副作用和流程编排
 *
 * 职责：
 * - 调用纯函数完成业务计算
 * - 处理所有副作用（数据库 / 日志 / 外部交互）
 *
 * 不承担复杂业务逻辑，只做"编排"
 */
class OrderService {

    public double createOrder(double price, int quantity, int userLevel) {
        // 调用纯逻辑（核心计算）
        double total = OrderCore.calculateTotal(price, quantity, userLevel);

        // 副作用：持久化数据
        Database.save(total);

        // 副作用：记录日志
        System.out.println("Order created: " + total);

        return total;
    }
}
```

**Step 3：测试纯逻辑，无需 mock**

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class OrderCoreTest {

    @Test
    void 普通用户无折扣() {
        assertEquals(100.0, OrderCore.calculateTotal(50.0, 2, 1));
    }

    @Test
    void 二级会员九折() {
        assertEquals(90.0, OrderCore.calculateTotal(50.0, 2, 2));
    }

    @Test
    void 三级会员八折() {
        assertEquals(80.0, OrderCore.calculateTotal(50.0, 2, 3));
    }
}
```

测试无需数据库连接、无需 mock 框架，直接验证业务规则。

---

### 案例2：Rust - 文件单词统计

#### 场景

实现一个 CLI 工具，读取文件内容并统计单词数量。

#### ❌ 传统写法：逻辑 + I/O 混在一起

```rust
use std::fs;

fn count_words_in_file(path: &str) -> usize {
    // 副作用：读取文件
    let content = fs::read_to_string(path).unwrap();

    // 业务逻辑：统计单词
    let count = content.split_whitespace().count();

    // 副作用：打印结果
    println!("Word count: {}", count);

    count
}
```

**问题**：
- 想测试"单词统计"逻辑，必须依赖真实文件
- 逻辑绑死在文件 I/O 上，换数据来源就得重写

#### ✅ 重构后：FC/IS 分离

**Step 1：提取 Functional Core（纯逻辑内核）**

```rust
/// 统计字符串中的单词数量（纯函数）
///
/// - 只依赖输入参数 content
/// - 不进行 I/O 操作
/// - 相同输入 → 相同输出
fn count_words(content: &str) -> usize {
    content.split_whitespace().count()
}
```

**Step 2：构建 Imperative Shell（命令式外壳）**

```rust
use std::fs;

fn run(path: &str) {
    // 副作用：读取文件
    let content = fs::read_to_string(path)
        .expect("Failed to read file");

    // 调用纯函数（核心逻辑）
    let count = count_words(&content);

    // 副作用：输出结果
    println!("Word count: {}", count);
}
```

**Step 3：测试纯逻辑**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn 统计普通文本() {
        assert_eq!(count_words("hello world rust"), 3);
    }

    #[test]
    fn 空字符串返回零() {
        assert_eq!(count_words(""), 0);
    }

    #[test]
    fn 处理多余空白() {
        assert_eq!(count_words("  hello   world  "), 2);
    }
}
```

测试无需文件系统，直接验证计算逻辑。

---

## 四、💡 示例 / 踩坑

### 识别非纯函数

在实践中，学会识别哪些操作是不纯的，是正确应用 FC/IS 模式的前提。以下都是非纯函数的常见形式：

```rust
// 修改外部状态 - 依赖并修改全局变量
let mut total = 0;
fn add_to_total(x: i32) { total += x; }

// 依赖外部状态 - 行为由环境变量决定
fn get_env_port() -> String {
    std::env::var("PORT").unwrap_or("3000".to_string())
}

// 依赖时间 - 每次调用结果不同
fn get_timestamp() -> u64 {
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs()
}

// 控制台输出 - 产生外部可观察行为
fn log_and_add(a: i32, b: i32) -> i32 {
    println!("calculating...");
    a + b
}
```

它们的共同特征：**函数的行为不完全由输入决定**。

### 常见陷阱

| ⚠️ 陷阱 | 表现 | ✅ 正确做法 |
|---------|------|------------|
| **过度追求纯函数** | 试图让所有函数都变纯，导致代码复杂度暴增 | 只隔离核心业务逻辑，I/O 操作交给 Shell |
| **边界划分不清** | 不知道哪些逻辑属于 Core，哪些属于 Shell | 判断标准：是否依赖外部状态？是否可预测？ |
| **Shell 承担业务逻辑** | Shell 中又混入了计算逻辑 | Shell 只做编排和 I/O，计算全交给 Core |
| **忽略副作用传染性** | 在 Core 中调用了不纯函数，导致 Core 也不纯 | Core 中的所有调用链都必须是纯的 |

### 什么时候适合使用 FC/IS 模式？

```
适合使用的场景：
├── 业务逻辑较复杂，需要频繁测试
├── 函数中混合了计算和 I/O 操作
└── 多个场景复用同一套计算逻辑

不一定需要的场景：
├── 简单的 CRUD 操作（逻辑简单，收益不大）
├── 纯 I/O 转发（没有业务计算）
└── 一次性脚本（不需要长期维护）
```

---

## 五、📋 总结

### 核心要点回顾

1. **纯函数的本质**：相同输入 → 相同输出，无副作用。它是可预测、可测试、可组合的基石
2. **FC/IS 模式的核心思想**：不是消除副作用，而是**隔离副作用**，保护核心逻辑
3. **重构方法**：提取纯逻辑为 Functional Core，将副作用留在 Imperative Shell，Shell 调用 Core

### 与方法拆分的关系

本文的 FC/IS 模式和"方法拆分"重构（[重构实战：打破依赖关系的两种核心方法]({{< relref "/posts/refactoring/breaking-dependencies.md" >}})）是一脉相承的：
- **方法拆分**：在同一类内分离纯逻辑与副作用（微观层面）
- **FC/IS 模式**：将纯逻辑和副作用分到不同的模块/类（架构层面）

### 延伸方向

- **领域驱动设计（DDD）**：Functional Core 可以对应领域层（Domain Layer），Imperative Shell 对应基础设施层（Infrastructure Layer）
- **六边形架构（Hexagonal Architecture）**：核心业务逻辑不依赖外部，通过端口与适配器与外界交互
- **Effect Systems**：在类型系统层面追踪副作用（如 Haskell 的 IO Monad）

💡 **实践建议**：下次写一个包含计算和 I/O 的函数时，先问自己——"能不能把计算逻辑提取出来，让它不依赖任何外部系统？"
