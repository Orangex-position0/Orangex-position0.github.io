+++
title = '重构实战：打破依赖关系的两种核心方法'
date = 2026-03-24T10:00:00+08:00
draft = false
description = '通过方法拆分和依赖倒置两种重构技术，解决代码难以测试的问题，提升代码可维护性'

categories = ['java', 'refactoring']
tags = ['重构', '单元测试', '解耦']
+++

## 一、🎯 背景 & 问题：你的代码为什么难以测试？

在日常开发中，我们经常会遇到这样的困境：

> 你想给一个 `checkout` 方法写单元测试，但发现它内部调用了数据库；你想测试 `registerUser` 方法，但发现它会真的发送邮件...

这些问题的共同特征是：**测试依赖外部系统（数据库、网络、第三方服务）**，导致：

| 问题 | 表现 |
|------|------|
| 测试运行慢 | 每次都要连接真实数据库/网络 |
| 测试结果不稳定 | 外部系统的状态不可控 |
| 无法独立测试核心逻辑 | 被副作用绑架 |

⚠️ **本质原因**：代码"设计耦合严重"——业务逻辑与副作用混在一起，形成了难以分割的依赖关系。

**本文目标**：通过两种实用的重构方法，让你的代码从"依赖具体实现"转变为"依赖抽象 + 可替换实现"，从而获得**可独立测试的纯业务逻辑**。

💡 **读完本文，你将学到**：两种常用的变更代码思路（如何解耦）

---

## 二、🧩 方案概述：解耦的核心思想

打破依赖关系的本质是 **解耦（Decoupling）**。

简单来说，就是让代码从：
```
调用方 --> 直接依赖具体实现
```

变成：
```
调用方 --> 依赖抽象接口 --> 可替换的具体实现（真实/测试）
```

本文介绍两种核心方法：

| 方法 | 核心思想 | 适用场景 |
|------|----------|----------|
| 🔧 **方法拆分** | 将纯业务逻辑与副作用操作分离 | 单方法内部混杂多种职责 |
| 🔄 **依赖倒置** | 通过接口抽象依赖，引入中介层 | 类与类之间的强耦合 |

---

## 三、🛠️ 实现步骤

### 🔧 方法1：方法拆分（逻辑隔离）

**核心概念**：
- 📌 **纯业务逻辑**：可直接测试的计算、校验等（无副作用）
- 🌊 **副作用操作**：依赖外部系统的操作（数据库、网络、I/O）

**重构目标**：将大方法中的"纯业务逻辑"和"副作用操作"拆分，让核心逻辑可单独测试。

#### 原始代码：测试被数据库绑架

```java
public class OrderService {

    public int checkout(Order order) {
        // ① 业务逻辑：计算价格
        int total = 0;
        for (Item item : order.getItems()) {
            total += item.getPrice();
        }

        // ② 副作用：访问数据库（不可控）
        Database db = new Database();
        db.save(order);

        return total;
    }
}
```

**问题**：想测试"价格计算"，必须先连接数据库。

#### 重构后：职责分离

```java
public class OrderService {

    public int checkout(Order order) {
        // 1️⃣ 只负责流程编排（orchestration）
        int total = calculateTotal(order);
        save(order);
        return total;
    }

    /**
     * 纯函数：无副作用，可单独测试
     * 通过提炼方法将纯逻辑分离出来
     */
    int calculateTotal(Order order) {
        int total = 0;
        for (Item item : order.getItems()) {
            total += item.getPrice();
        }
        return total;
    }

    /**
     * 副作用方法：隔离数据库依赖
     * 将副作用集中到单独方法中
     */
    private void save(Order order) {
        new Database().save(order);
    }
}
```

#### 单元测试：无需数据库

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class OrderServiceTest {

    @Test
    void testCalculateTotal() {
        // 准备测试数据
        Order order = new Order(List.of(
            new Item(10),
            new Item(20)
        ));

        // 执行测试 - 不依赖数据库
        int result = new OrderService().calculateTotal(order);

        // 验证结果
        assertEquals(30, result);
    }
}
```

**关键收益**：`calculateTotal` 现在是一个**纯函数（Pure Function）**——给定相同输入永远返回相同输出，且没有副作用，非常容易测试。

⭐ **测试优势**：无需任何 Mock 框架，直接调用即可验证

---

### 🔄 方法2：引入中介（依赖倒置）

当依赖关系跨越类边界时，方法拆分就不够用了。这时需要引入一个**中介（接口）**来替代具体实现，让依赖"可替换"。

这就是 **依赖倒置原则（Dependency Inversion Principle）** 的实践：
- 📌 高层模块不应依赖低层模块，两者都应依赖抽象
- 📌 抽象不应依赖细节，细节应依赖抽象

#### 原始代码：硬编码依赖

```java
public class UserService {
    // ❌ 硬编码依赖具体实现（强耦合）
    private RealEmailSender emailSender = new RealEmailSender();

    public void registerUser(String email) {
        // ... 业务逻辑：验证邮箱格式、检查重复等 ...

        // ❌ 副作用：真实发送邮件
        emailSender.sendWelcomeEmail(email);
    }
}
```

**测试困境**：测试 `registerUser` 时会真的发送邮件，既慢又有副作用。

#### 重构步骤

**Step 1：定义抽象（接口）**

```java
/**
 * 抽象：定义"发送邮件"的能力，而不是具体实现
 */
interface EmailProvider {
    void send(String to, String subject, String content);
}
```

**Step 2：提供真实实现（生产环境使用）**

```java
/**
 * 真实实现：生产环境发送真实邮件
 */
class RealEmailSender implements EmailProvider {

    @Override
    public void send(String to, String subject, String content) {
        // 实际发送邮件（网络请求）
        // 调用邮件服务商 API 等
    }
}
```

**Step 3：业务类依赖抽象**

```java
public class UserService {

    // ✅ 依赖抽象（接口）而不是具体类
    private final EmailProvider emailProvider;

    /**
     * 通过构造函数注入依赖（Dependency Injection）
     * 调用方决定使用哪个实现
     */
    public UserService(EmailProvider emailProvider) {
        this.emailProvider = emailProvider;
    }

    public void registerUser(String email) {
        // ... 业务逻辑 ...

        // ✅ 调用抽象，不关心具体如何发送
        emailProvider.send(email, "欢迎注册", "欢迎加入我们！");
    }
}
```

**Step 4：创建测试用的 Fake / Stub 实现**

```java
/**
 * 测试用实现：只记录调用，不执行真实操作
 */
class FakeEmailSender implements EmailProvider {

    // 记录发送的邮件，用于验证
    private final List<EmailRecord> sentEmails = new ArrayList<>();
    private boolean isCalled = false;

    @Override
    public void send(String to, String subject, String content) {
        // ✅ 不做真实操作，只记录
        sentEmails.add(new EmailRecord(to, subject, content));
        isCalled = true;
    }

    public boolean wasEmailSentTo(String email) {
        return sentEmails.stream()
            .anyMatch(e -> e.to().equals(email));
    }

    // 记录类
    record EmailRecord(String to, String subject, String content) {}
}
```

#### 单元测试：完全隔离外部依赖

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class UserServiceTest {

    @Test
    void registerUser_shouldSendWelcomeEmail() {
        // 1️⃣ 创建测试替身（Test Double）
        FakeEmailSender fakeEmailSender = new FakeEmailSender();

        // 2️⃣ 注入测试替身
        UserService userService = new UserService(fakeEmailSender);

        // 3️⃣ 执行被测方法
        userService.registerUser("user@example.com");

        // 4️⃣ 验证行为（而不是结果）
        assertTrue(fakeEmailSender.wasEmailSentTo("user@example.com"));
    }
}
```

**关键收益**：
- 🚀 测试速度快：没有网络请求
- ✅ 测试可重复：结果不受外部服务状态影响
- 🔍 验证行为：可以精确验证 `EmailProvider.send()` 是否被调用

---

## 四、💡 实战示例 & 踩坑指南

### 综合案例：订单处理服务

让我们看一个更复杂的例子，结合两种方法：

```java
/**
 * 重构后的订单处理服务：完全可测试
 */
public class OrderProcessor {

    private final PaymentGateway paymentGateway;
    private final InventoryService inventoryService;

    // 通过构造函数注入所有外部依赖
    public OrderProcessor(PaymentGateway paymentGateway,
                          InventoryService inventoryService) {
        this.paymentGateway = paymentGateway;
        this.inventoryService = inventoryService;
    }

    public OrderResult process(Order order) {
        // 1️⃣ 纯业务逻辑：验证订单
        validateOrder(order);

        // 2️⃣ 纯业务逻辑：计算总价
        BigDecimal total = calculateTotal(order);

        // 3️⃣ 副作用：检查库存（通过抽象）
        if (!inventoryService.isAvailable(order)) {
            return OrderResult.outOfStock();
        }

        // 4️⃣ 副作用：处理支付（通过抽象）
        PaymentResult payment = paymentGateway.charge(order.getCustomerId(), total);

        // 5️⃣ 纯业务逻辑：根据支付结果处理
        return buildResult(order, payment);
    }

    // 可独立测试的纯方法
    void validateOrder(Order order) {
        if (order == null || order.getItems().isEmpty()) {
            throw new IllegalArgumentException("订单无效");
        }
    }

    // 可独立测试的纯方法
    BigDecimal calculateTotal(Order order) {
        return order.getItems().stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    // 可独立测试的纯方法
    private OrderResult buildResult(Order order, PaymentResult payment) {
        return payment.isSuccess()
            ? OrderResult.success(order.getId())
            : OrderResult.paymentFailed(payment.getErrorMessage());
    }
}
```

### 常见陷阱 & 解决方案

| ⚠️ 陷阱 | 表现 | ✅ 解决方案 |
|------|------|----------|
| **接口过大** | 一个接口包含十几个方法，实现困难 | 遵循接口隔离原则，拆分为小接口 |
| **过度设计** | 简单场景也强行引入接口 | 先有测试需求再抽象，避免过早优化 |
| **Mock 滥用** | 测试全是 Mock，失去实际意义 | 优先用 Fake，核心业务逻辑用真对象 |
| **依赖隐藏** | 在方法内部 `new` 依赖对象 | 所有依赖通过构造函数注入 |

### 测试策略选择

```
┌─────────────────────────────────────────────────────────┐
│  纯业务逻辑（calculateTotal, validateOrder）              │
│  └── 直接测试，无需任何 Mock/Fake                        │
├─────────────────────────────────────────────────────────┤
│  外部依赖（EmailProvider, PaymentGateway）               │
│  ├── 集成测试：使用真实实现（连接测试环境）               │
│  └── 单元测试：使用 Fake/Stub 实现                      │
├─────────────────────────────────────────────────────────┤
│  协调/编排逻辑（checkout, process）                      │
│  └── 使用 Fake 验证交互行为                              │
└─────────────────────────────────────────────────────────┘
```

---

## 五、📋 总结

### 核心要点回顾

**为什么要打破依赖**：让代码可测试、可维护，将业务逻辑与副作用分离

**两种核心方法**：
   - 🔧 **方法拆分**：在同一类内分离纯逻辑与副作用
   - 🔄 **引入中介**：通过接口实现类间的依赖倒置

**关键原则**：
   - 依赖抽象，而非具体实现
   - 构造函数注入依赖
   - 优先使用可独立测试的纯函数

### 重构决策流程

```
开始
  │
  ▼
方法内部混杂了业务逻辑和副作用？ ──是──▶ 方法拆分（提炼纯函数）
  │否
  ▼
类依赖了具体实现导致难以测试？ ────是──▶ 引入中介（依赖倒置）
  │否
  ▼
结束（代码已足够清晰）
```

💡 **实践建议**：下次写代码时，先问自己——"这个方法能脱离数据库运行测试吗？"如果不能，也许就是重构的时候了。

## 六、📚 参考
- [程序员的README (豆瓣)](https://book.douban.com/subject/36457109/) - 第二章的部分内容