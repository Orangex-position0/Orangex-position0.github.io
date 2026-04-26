+++
title = '手写 Rust Async Model（一）：简易版 Future 结构'
date = 2026-04-26T10:00:00+08:00
draft = true
description = '从阻塞式编程的痛点出发，手写一个最简单的 Future Trait 和状态机，理解 Rust 异步编程的最底层抽象'
categories = ['rust']
tags = ['rust', 'async']
series = '手写 Rust Async Model'
seriesWeight = 1
+++

## 背景 & 问题：阻塞式编程的代价

假设你有 3 个任务要执行，每个任务耗时 1 秒，最直觉的写法是这样的：

```rust
for i in 1..=3 {
    println!("任务 {} 开始", i);
    thread::sleep(Duration::from_secs(1));
    println!("任务 {} 完成", i);
}
```

这段代码没有任何问题——除了**慢**。三个任务串行执行，总耗时约 3 秒。在这 3 秒里，线程在 `thread::sleep` 期间什么都不做，只能傻等。这就是**阻塞式编程**。

> 类比现实：这就像你站在厨房，烧水的时候死盯着水壶看，什么都不做。等水开了再去切菜，等菜切完了再煮汤。效率极低。

我们需要一种**非阻塞**的方式：当一个任务在等待时，线程可以先去做别的事情。Rust 中实现这种思路的核心抽象就是 `Future`。

## 方案：Future 状态机

Rust 中的 Future 本质是一个**被动状态机**——它由"数据 + 状态 + 状态推进逻辑"组成。外部通过调用 `poll()` 方法来驱动它前进。

整个过程分为三步：

1. **定义协议**——`Future` Trait 和状态枚举 `PollState`
2. **实现状态机**——让具体的异步任务实现这个 Trait
3. **驱动执行**——写一个 Runtime 反复调用 `poll()` 直到任务完成

## 实现步骤

### 第一步：定义 Future Trait 和 PollState

要让所有异步任务都遵循同一套协议，我们需要一个 Trait 作为标准：

```rust
/// Future 状态机的状态
pub enum PollState<T> {
    /// 任务已完成，携带返回值 T
    Ready(T),
    /// 任务尚未完成
    Pending,
}

/// 简化版 Future Trait
pub trait MyFuture {
    /// 异步任务的返回值类型
    type Output;

    /// 推进状态机
    /// 调用后要么返回 Ready(结果)，要么返回 Pending（稍后再试）
    fn poll(&mut self) -> PollState<Self::Output>;
}
```

- `type Output` 是关联类型，表示这个 Future 完成后返回什么
- `poll()` 是核心方法：调用一次，状态机前进一步。要么 `Ready(结果)`，要么 `Pending`

> 注意：这里 `poll()` 的签名和标准库不同（没有 `Pin` 和 `Context`），我们先从最简单的版本开始，后续章节会逐步补全。

### 第二步：实现一个具体的 Future

接下来让一个具体的结构体实现 `MyFuture`。`CountFuture` 持有一个计数器，每次 `poll` 减一，减到零时返回完成：

```rust
/// 需要多次 poll 才能完成的 Future
struct CountFuture {
    count: u32,
}

impl CountFuture {
    fn new(count: u32) -> Self {
        Self { count }
    }
}

impl MyFuture for CountFuture {
    type Output = &'static str;

    fn poll(&mut self) -> PollState<Self::Output> {
        if self.count == 0 {
            PollState::Ready("finished")
        } else {
            println!("  count = {}", self.count);
            self.count -= 1;
            PollState::Pending
        }
    }
}
```

关键点：**Future 内部持有状态**，`poll()` 负责推进状态。每次调用 `poll()`，`count` 减一，状态机往前走一步。

### 第三步：写一个 Runtime 驱动 Future

Future 本身只是状态机，需要一个外部"驱动器"来反复调用 `poll()`。这个驱动器就是 **Runtime**：

```rust
struct Runtime;

impl Runtime {
    fn block_on<F: MyFuture>(&mut self, mut future: F) -> F::Output {
        loop {
            match future.poll() {
                PollState::Ready(val) => return val,
                PollState::Pending => {
                    // 模拟等待（真实情况是 IO 事件就绪）
                    thread::sleep(Duration::from_millis(200));
                }
            }
        }
    }
}
```

`block_on` 做的事很简单：

1. 接收一个实现了 `MyFuture` 的任务
2. 反复调用 `poll()`，推动状态机前进
3. 如果返回 `Ready(val)`，把结果返回给调用者
4. 如果返回 `Pending`，等一会儿再 `poll()`

把所有代码串起来运行：

```rust
fn main() {
    let mut rt = Runtime;
    let result = rt.block_on(CountFuture::new(3));
    println!("最终结果: {}", result);
}
```

输出：

```
  count = 3
  count = 2
  count = 1
最终结果: finished
```

## 示例 & 踩坑

### 当前模型的核心问题

现在我们有了一个完整的闭环：Future 定义"做什么"，Runtime 负责驱动。但这个 Runtime 有一个致命问题——**它在盲目轮询（busy polling）**。

这就像你点了外卖，每隔 10 秒打电话问外卖员："到了没？到了没？到了没？" 外卖员还没出发呢，你的电话费先亏完了。

问题出在 Runtime 的这段代码：

```rust
PollState::Pending => {
    // 模拟等待
    thread::sleep(Duration::from_millis(200));
}
```

Runtime 在"猜"什么时候该 `poll`——它不知道事件什么时候就绪，只能傻等。这浪费了 CPU 资源和等待时间。

**本质问题：Future 没有主动通知 Runtime 的能力。**

### 本章完整代码

```rust
use std::thread;
use std::time::Duration;

// ================== 核心抽象 ==================

/// Future 的状态
pub enum PollState<T> {
    /// 任务已完成，携带返回值
    Ready(T),
    /// 任务尚未完成
    Pending,
}

/// 所有异步任务都必须实现这个 trait
pub trait MyFuture {
    /// 异步任务的返回值类型
    type Output;

    /// 推进状态机
    fn poll(&mut self) -> PollState<Self::Output>;
}

// ================== 具体实现 ==================

/// 需要多次 poll 才能完成的 Future
struct CountFuture {
    count: u32,
}

impl CountFuture {
    fn new(count: u32) -> Self {
        Self { count }
    }
}

impl MyFuture for CountFuture {
    type Output = &'static str;

    fn poll(&mut self) -> PollState<Self::Output> {
        if self.count == 0 {
            PollState::Ready("finished")
        } else {
            println!("  count = {}", self.count);
            self.count -= 1;
            PollState::Pending
        }
    }
}

// ================== 运行时 ==================

/// 最简单的 Runtime：反复调用 poll 直到完成
struct Runtime;

impl Runtime {
    fn block_on<F: MyFuture>(&mut self, mut future: F) -> F::Output {
        loop {
            match future.poll() {
                PollState::Ready(val) => return val,
                PollState::Pending => {
                    // 模拟等待（真实情况是 IO 事件就绪）
                    thread::sleep(Duration::from_millis(200));
                }
            }
        }
    }
}

// ================== main ==================

fn main() {
    let mut rt = Runtime;
    let result = rt.block_on(CountFuture::new(3));
    println!("最终结果: {}", result);
}
```

## 总结

这一章我们从阻塞式编程的问题出发，实现了 Rust 异步编程的最底层抽象：

- **`PollState<T>`**：Future 的两种状态——`Ready` 和 `Pending`
- **`MyFuture` Trait**：所有异步任务的标准协议，核心方法是 `poll()`
- **`CountFuture`**：一个具体的状态机实现
- **`Runtime`**：驱动 Future 执行的运行时

但当前的 Runtime 存在一个严重问题：**盲目轮询**。下一章我们将引入**事件驱动**模型，让 Future 在事件就绪时主动通知 Runtime，彻底解决这个问题。
