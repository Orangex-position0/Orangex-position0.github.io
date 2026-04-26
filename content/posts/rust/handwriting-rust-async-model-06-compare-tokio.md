+++
title = '手写 Rust Async Model（六）：与 Tokio 相比'
date = 2026-04-26T15:00:00+08:00
draft = false
description = '将手写的简易异步运行时与生产级 Tokio 逐项对比，看清差距与学习方向'
categories = ['rust']
tags = ['rust', 'async', 'tokio']
series = '手写 Rust Async Model'
seriesWeight = 6
+++

## 背景 & 问题：我们做到了哪一步？

前面的五章，我们从零开始手写了一个简化版的 Rust 异步运行时：

1. **Future Trait**：定义了异步任务的标准协议
2. **Reactor-Executor 模型**：实现了事件驱动的任务调度
3. **组合器**：实现了 JoinAll、Then、Select 三种并发控制模式
4. **Pin**：解决了自引用结构的内存安全问题
5. **async/await**：揭示了编译器语法糖的本质

但我们的实现只是一个**教学级别的最小可行版本**。与生产级运行时 Tokio 相比，还有很大的差距。这一章我们逐项对比，看清差距，也明确学习方向。

## 方案：逐项对比

我们从 Future Trait、Executor、Waker 三个维度，对比手写版与 Tokio 的差异。

### Future Trait 对比

```rust
// ===== 手写的版本 =====
pub trait MyFuture {
    type Output;
    fn poll(&mut self, waker: &Waker) -> PollState<Self::Output>;
}

// ===== 标准库版本 =====
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

| 差异 | 我们的 MyFuture | std::future::Future |
|---|---|---|
| self 类型 | `&mut self` | `Pin<&mut Self>` |
| 通知参数 | `waker: &Waker` | `cx: &mut Context<'_>` |
| 返回类型 | `PollState<T>` | `Poll<T>` |
| 状态枚举 | `Ready(T)` / `Pending` | `Ready(T)` / `Pending` |

两个关键差异：

- **`Pin<&mut Self>`**：保证自引用结构在 `poll` 过程中不会被 move（第四章讲过）
- **`Context<'_>`**：提供扩展性。多一层包装后，如果未来需要在 `poll` 中传递额外信息（如调度器优先级、任务元数据），只需在 `Context` 中添加新方法即可，不用改 `poll` 的签名

### Executor 对比

我们的手写版相当于 MVP 版 Executor——能跑，但功能最少。Tokio 在我们基础上增加了大量生产级能力：

| 能力 | 我们的 Executor | tokio |
|---|---|---|
| 线程模型 | 单线程或固定线程 | 多线程 + work-stealing |
| I/O 后端 | epoll（通过 mio） | epoll + io_uring（可选） |
| 定时器 | 不支持 | Timer Wheel 实现 |
| 任务取消 | 不支持 | `JoinHandle::abort()` |
| 优雅关闭 | 不支持 | 优雅等待所有任务完成 |
| 信号处理 | 不支持 | Unix 信号 / Windows 事件 |
| 文件 I/O | 不支持 | 基于 blocking 线程池的异步文件操作 |

几个值得关注的点：

- **work-stealing**：我们的 Executor 任务绑定在创建它的线程上，不能跨线程迁移。Tokio 使用工作窃取算法——空闲线程可以从忙碌线程"偷"任务，实现负载均衡
- **io_uring**：Linux 5.1+ 提供的全新异步 I/O 接口，相比 epoll 有更好的性能，Tokio 已支持
- **Timer Wheel**：高效定时器数据结构，支持大量定时任务的插入和过期检查，时间复杂度为 O(1)
- **任务取消**：我们的实现中，Future 一旦启动就只能等它完成。Tokio 支持通过 `JoinHandle::abort()` 主动取消任务

### Waker 对比

| 差异 | 我们的 Waker | Tokio 的 Waker |
|---|---|---|
| 唤醒方式 | `thread::unpark` | 原子计数 + unpark |
| 跨线程 | 支持（Thread handle） | 支持（原子操作） |
| 性能 | 每次唤醒都需要获取锁 | 无锁设计 |
| 实现复杂度 | 简单直接 | 实现复杂 |

我们的 Waker 用 `Arc<Mutex<VecDeque>>` 存储就绪队列，每次 `wake()` 都需要获取锁。Tokio 的 Waker 使用原子计数实现无锁唤醒，在高并发场景下性能优势明显——但实现复杂度也高得多。

## 示例 & 踩坑

### 我们实现了什么

虽然与 Tokio 差距很大，但我们的手写版本覆盖了异步运行时的**核心概念**：

```
Future（状态机）
  → PollState（状态枚举）
  → Waker（通知机制）
  → Reactor（事件监听）
  → Executor（任务调度）
  → 组合器（并发控制）
  → Pin（内存安全）
  → async/await（语法糖）
```

理解了这些概念，再看 Tokio 的源码或文档就不会觉得陌生。Tokio 的每一个模块都可以对应到我们手写的某个组件：

- `tokio::runtime::Runtime` → 我们的 `Executor`
- `tokio::net` → 我们的 `Reactor`（但用 mio/io_uring 替代了 `thread::sleep`）
- `std::future::Future` → 我们的 `MyFuture`
- `std::task::Waker` → 我们的 `Waker`
- `futures::join!` → 我们的 `JoinAll`
- `futures::select!` → 我们的 `Select`

### 下一步学习方向

如果你想继续深入 Rust 异步编程，以下资源值得一读：

- [Tokio 官方教程](https://tokio.rs/tokio/tutorial) — 从零开始学习 Tokio，涵盖异步文件 I/O、网络编程、通道、信号处理等实用主题
- [Rust Async Book](https://rust-lang.github.io/async-book/) — Rust 官方异步编程书籍，深入讲解 Future、async/await、Pin、Waker 等核心概念
- [std::future::Future 文档](https://doc.rust-lang.org/std/future/trait.Future.html) — 标准库 Future trait 的官方文档，包含完整的 API 说明和示例

## 总结

经过六章的循序渐进，我们从阻塞式编程的问题出发，一步步手写了 Rust 异步运行时的核心组件：

| 章节 | 内容 | 核心收获 |
|---|---|---|
| 一 | Future 结构 | Future 是被动状态机，`poll()` 驱动前进 |
| 二 | Reactor-Executor | 事件驱动取代盲目轮询，Waker 连接事件与调度 |
| 三 | 并发调度 | JoinAll/Then/Select 三种组合器模式 |
| 四 | Pin | 禁止 move 保护自引用结构 |
| 五 | async/await | 编译器生成的状态机语法糖 |
| 六 | 与 Tokio 对比 | 看清教学版与生产级的差距 |

手写的目的不是造轮子，而是**理解轮子的构造原理**。当你理解了每一层抽象为什么存在、解决了什么问题之后，再使用 Tokio 这样的生产级框架时，就能做到知其然也知其所以然。
