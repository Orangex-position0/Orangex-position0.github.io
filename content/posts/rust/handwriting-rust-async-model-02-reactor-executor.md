+++
title = '手写 Rust Async Model（二）：Reactor-Executor 模型'
date = 2026-04-26T11:00:00+08:00
draft = true
description = '引入事件驱动架构，手写 Waker、Reactor、Executor 三大组件，实现真正的非阻塞异步运行时'
categories = ['rust']
tags = ['rust', 'async', 'reactor-executor']
series = '手写 Rust Async Model'
seriesWeight = 2
+++

## 背景 & 问题：盲目轮询的代价

上一章我们写了一个最小 Runtime，它能跑，但有个致命问题——**盲目轮询（busy polling）**。即使 I/O 没准备好，Runtime 也在反复 `poll()`，白白消耗 CPU 资源。

这就像你点了外卖，每隔 10 秒打电话问外卖员"到了没？"——你在"猜"什么时候该 `poll`。

我们需要的是：**完全不主动 `poll`，等事件来了再 `poll`**。这就是**事件驱动**的核心思想。

## 方案：Reactor-Executor 模型

事件驱动的异步运行时由四个核心组件组成：

| 组件 | 本质角色 | 核心职责 |
|---|---|---|
| Future | 状态机 | 表示异步任务，在 `poll()` 时推进状态，必要时注册唤醒机制 |
| Executor | 调度器 | 维护就绪队列，调度任务并调用 `poll()` 持续推进 Future |
| Waker | 通知器 | 连接事件与调度的桥梁：在外部事件就绪时，通知 Executor 重新调度 |
| Reactor | 事件源 | 与操作系统交互（epoll/kqueue 等），监听 I/O/timer 事件，就绪时触发 Waker |

整体协作流程：

```
Future 返回 Pending 时 → 向 Reactor 注册事件
                                    ↓
Reactor 监听到事件就绪 → 调用 Waker.wake()
                                    ↓
Waker 将任务放回就绪队列 → unpark Executor 线程
                                    ↓
Executor 从就绪队列取任务 → 再次 poll Future
```

## 实现步骤

### 第一步：定义 Waker（通知器）

Waker 的职责：**把任务放回就绪队列，然后唤醒 Executor 线程**。

```rust
/// 通知器：将任务放回就绪队列，并唤醒 Executor 线程
#[derive(Clone)]
pub struct Waker {
    /// 任务 ID
    id: usize,
    /// 共享的就绪队列
    queue: Arc<Mutex<VecDeque<usize>>>,
    /// Executor 所在线程（用于 unpark）
    thread: Thread,
}

impl Waker {
    /// 唤醒：将任务放回就绪队列，并 unpark Executor
    pub fn wake(&self) {
        self.queue.lock().unwrap().push_back(self.id);
        self.thread.unpark();
    }
}
```

三个字段各司其职：

- `id`：标识需要被重新调度的任务
- `queue`：共享的就绪队列，存放"可以重新 `poll` 的任务 ID"
- `thread`：Executor 所在线程的句柄，用于 `unpark()` 唤醒

> 关键点：Waker 本身不执行任务，不调用 `poll()`，不推进状态机。它只负责"通知"。

### 第二步：定义 Reactor（事件监听者）

Reactor 的职责：等待外部事件，事件就绪后调用 Waker。

```rust
/// 事件监听者：后台线程等待 IO/timer 完成，完成后调用 wake
#[derive(Clone)]
pub struct Reactor;

impl Reactor {
    pub fn new() -> Self {
        Self
    }

    /// 注册一个定时器任务
    /// 在 dur 时间后调用 waker.wake()
    pub fn register_timer(&self, dur: Duration, waker: Waker) {
        thread::spawn(move || {
            thread::sleep(dur);
            waker.wake();
        });
    }
}
```

这里用 `thread::sleep` 模拟事件等待。在真实的运行时（如 tokio）中，Reactor 会使用操作系统提供的高效事件通知机制（Linux 的 epoll、macOS 的 kqueue 等）。

### 第三步：优化 Future Trait

由于引入了 Waker，Future 的 `poll()` 方法需要增加参数：

```rust
/// 简化版 Future Trait
pub trait MyFuture {
    type Output;
    fn poll(&mut self, waker: &Waker) -> PollState<Self::Output>;
}
```

核心改动：Future 不再被忙轮询，而是**主动注册唤醒通知**。`poll()` 检查到任务未就绪时，会向 Reactor 注册事件，让它在事件就绪时调用 Waker。

### 第四步：实现 Delay Future

`Delay` 是一个等待指定时间后完成的 Future，利用事件驱动机制：

```rust
/// 延迟指定时间后完成的 Future
pub struct Delay {
    /// 时间到期的绝对时间点
    when: Instant,
    /// 是否已注册到 Reactor（防止重复注册）
    registered: bool,
    /// 任务 ID（用于日志）
    id: usize,
}

impl Delay {
    pub fn new(id: usize, ms: u64) -> Self {
        Self {
            when: Instant::now() + Duration::from_millis(ms),
            registered: false,
            id,
        }
    }
}

impl MyFuture for Delay {
    type Output = ();

    fn poll(&mut self, waker: &Waker) -> PollState<Self::Output> {
        // 时间已到 → 完成
        if Instant::now() >= self.when {
            println!("task {} done", self.id);
            return PollState::Ready(());
        }

        // 只注册一次事件（防止重复 wake）
        if !self.registered {
            let dur = self.when - Instant::now();
            let reactor = Reactor::new();
            reactor.register_timer(dur, waker.clone());
            self.registered = true;
        }

        PollState::Pending
    }
}
```

注意 `registered` 字段的作用——**防止重复注册**。如果每次 `poll()` 都注册一次，会导致重复唤醒、任务被多次调度，甚至引发逻辑错误。

### 第五步：定义 Executor（任务调度器）

Executor 的职责：管理所有任务的生命周期——创建、调度、轮询、回收。

```rust
/// 任务类型: 封装一个可发送的 Future
type Task = Box<dyn MyFuture<Output = ()> + Send>;

/// 任务调度器：从就绪队列取任务，poll Future
pub struct Executor {
    /// 任务表，存储所有未完成的任务
    tasks: HashMap<usize, Task>,
    /// 就绪队列：等待被 poll 的任务 ID
    queue: Arc<Mutex<VecDeque<usize>>>,
    /// 下一个任务 ID
    next_id: usize,
}

impl Executor {
    pub fn new() -> Self {
        Self {
            tasks: HashMap::new(),
            queue: Arc::new(Mutex::new(VecDeque::new())),
            next_id: 0,
        }
    }

    /// 提交一个异步任务
    pub fn spawn<F>(&mut self, fut: F)
    where
        F: MyFuture<Output = ()> + Send + 'static,
    {
        let id = self.next_id;
        self.next_id += 1;
        self.tasks.insert(id, Box::new(fut));
        self.queue.lock().unwrap().push_back(id);
    }

    /// 运行调度循环，直到所有任务完成
    pub fn run(&mut self) {
        let current_thread = thread::current();

        loop {
            // 从就绪队列取一个任务
            // 注意：必须用独立代码块限定锁的生命周期
            // 否则 MutexGuard 的临时变量生命周期会延长到整个 match 块
            // 导致 park() 时仍持有锁，造成死锁
            let id = {
                let mut q = self.queue.lock().unwrap();
                q.pop_front()
            };

            let id = match id {
                Some(id) => id,
                None => {
                    if self.tasks.is_empty() {
                        // 所有任务已完成
                        break;
                    }
                    // 还有任务但都 Pending → park 等待 wake
                    thread::park();
                    continue;
                }
            };

            // 从任务表中取出任务
            let mut task = match self.tasks.remove(&id) {
                Some(t) => t,
                None => continue,
            };

            // 创建 Waker（与就绪队列共享）
            let waker = Waker {
                id,
                queue: self.queue.clone(),
                thread: current_thread.clone(),
            };

            // poll 任务
            match task.poll(&waker) {
                PollState::Ready(()) => {
                    // 完成 → 丢弃
                }
                PollState::Pending => {
                    // 未完成 → 放回
                    self.tasks.insert(id, task);
                }
            }
        }
    }
}
```

`run()` 方法的调度循环流程：

1. **取任务 ID**：从就绪队列中取出队首的任务 ID
2. **取任务**：在 HashMap 中根据 ID 取出 `Task` 实例
3. **创建 Waker**：为任务创建专属 Waker
4. **`poll()` 并处理结果**：完成则丢弃，未完成则放回

## 示例 & 踩坑

### 运行示例

```rust
fn main() {
    let mut exec = Executor::new();

    // 3 个并发延迟任务
    exec.spawn(Delay::new(1, 500));
    exec.spawn(Delay::new(2, 1000));
    exec.spawn(Delay::new(3, 1500));

    let start = Instant::now();
    exec.run();
    println!("非并发的耗时: 3s");
    println!("实际的并发耗时: {:?}", start.elapsed());
}
```

三个任务分别等待 500ms、1000ms、1500ms。由于是事件驱动的，总耗时约 1.5 秒（最长的那个任务的时间），而不是串行的 3 秒。

### 调度循环详细流程

```
1. Executor::spawn(Delay{500ms})  → task 0 加入就绪队列
   Executor::spawn(Delay{1000ms}) → task 1 加入就绪队列
   Executor::spawn(Delay{1500ms}) → task 2 加入就绪队列

2. Executor::run() 开始循环

3. 从队列取出 task 0 → poll → 时间未到
   → 注册 500ms 定时器到 Reactor
   → Reactor spawn 线程 A（sleep 500ms 后调用 waker0.wake()）
   → task 0 返回 Pending，放回 tasks

4. 从队列取出 task 1 → poll → 时间未到 → 注册 1000ms 定时器
5. 从队列取出 task 2 → poll → 时间未到 → 注册 1500ms 定时器

6. 队列为空，tasks 不为空 → thread::park()，Executor 休眠

7. 500ms 后，线程 A 醒来，调用 waker0.wake()
   → task 0 的 ID 被放入就绪队列
   → thread.unpark() 唤醒 Executor

8. Executor 醒来，poll task 0 → Ready → 移除

9. 1000ms 后，waker1.wake() → poll task 1 → Ready
10. 1500ms 后，waker2.wake() → poll task 2 → Ready

11. tasks 为空，退出循环
```

### 踩坑：Mutex 锁的生命周期

`run()` 方法中取任务 ID 时，必须用独立代码块限定锁的生命周期：

```rust
// 正确写法
let id = {
    let mut q = self.queue.lock().unwrap();
    q.pop_front()
};

// 错误写法：锁的生命周期可能延长到 park() 时
// let id = self.queue.lock().unwrap().pop_front();
```

如果锁的生命周期延长到 `thread::park()`，会导致**死锁**——Executor 持着锁休眠，Reactor 线程调用 `wake()` 时试图获取锁，永远拿不到。

## 本章完整代码

```rust
use std::collections::{HashMap, VecDeque};
use std::sync::{Arc, Mutex};
use std::thread::{self, Thread};
use std::time::{Duration, Instant};

// ================== 核心抽象 ==================

/// Future 的状态
pub enum PollState<T> {
    Ready(T),
    Pending,
}

/// 所有异步任务都必须实现这个 trait
pub trait MyFuture {
    type Output;
    fn poll(&mut self, waker: &Waker) -> PollState<Self::Output>;
}

// ================== Waker ==================

#[derive(Clone)]
pub struct Waker {
    id: usize,
    queue: Arc<Mutex<VecDeque<usize>>>,
    thread: Thread,
}

impl Waker {
    pub fn wake(&self) {
        self.queue.lock().unwrap().push_back(self.id);
        self.thread.unpark();
    }
}

// ================== Reactor ==================

#[derive(Clone)]
pub struct Reactor;

impl Reactor {
    pub fn new() -> Self {
        Self
    }

    pub fn register_timer(&self, dur: Duration, waker: Waker) {
        thread::spawn(move || {
            thread::sleep(dur);
            waker.wake();
        });
    }
}

// ================== Delay Future ==================

pub struct Delay {
    when: Instant,
    registered: bool,
    id: usize,
}

impl Delay {
    pub fn new(id: usize, ms: u64) -> Self {
        Self {
            when: Instant::now() + Duration::from_millis(ms),
            registered: false,
            id,
        }
    }
}

impl MyFuture for Delay {
    type Output = ();

    fn poll(&mut self, waker: &Waker) -> PollState<Self::Output> {
        if Instant::now() >= self.when {
            println!("task {} done", self.id);
            return PollState::Ready(());
        }

        if !self.registered {
            let dur = self.when - Instant::now();
            let reactor = Reactor::new();
            reactor.register_timer(dur, waker.clone());
            self.registered = true;
        }

        PollState::Pending
    }
}

// ================== Executor ==================

type Task = Box<dyn MyFuture<Output = ()> + Send>;

pub struct Executor {
    tasks: HashMap<usize, Task>,
    queue: Arc<Mutex<VecDeque<usize>>>,
    next_id: usize,
}

impl Executor {
    pub fn new() -> Self {
        Self {
            tasks: HashMap::new(),
            queue: Arc::new(Mutex::new(VecDeque::new())),
            next_id: 0,
        }
    }

    pub fn spawn<F>(&mut self, fut: F)
    where
        F: MyFuture<Output = ()> + Send + 'static,
    {
        let id = self.next_id;
        self.next_id += 1;
        self.tasks.insert(id, Box::new(fut));
        self.queue.lock().unwrap().push_back(id);
    }

    pub fn run(&mut self) {
        let current_thread = thread::current();

        loop {
            let id = {
                let mut q = self.queue.lock().unwrap();
                q.pop_front()
            };

            let id = match id {
                Some(id) => id,
                None => {
                    if self.tasks.is_empty() {
                        break;
                    }
                    thread::park();
                    continue;
                }
            };

            let mut task = match self.tasks.remove(&id) {
                Some(t) => t,
                None => continue,
            };

            let waker = Waker {
                id,
                queue: self.queue.clone(),
                thread: current_thread.clone(),
            };

            match task.poll(&waker) {
                PollState::Ready(()) => {}
                PollState::Pending => {
                    self.tasks.insert(id, task);
                }
            }
        }
    }
}

// ================== main ==================

fn main() {
    let mut exec = Executor::new();
    let start = Instant::now();

    exec.spawn(Delay::new(1, 500));
    exec.spawn(Delay::new(2, 1000));
    exec.spawn(Delay::new(3, 1500));

    exec.run();

    println!("非并发的耗时: 3s");
    println!("实际的并发耗时: {:?}", start.elapsed());
}
```

## 总结

这一章我们引入了事件驱动架构，实现了 Reactor-Executor 模型的完整闭环：

- **Waker**：通知器，将任务放回就绪队列并唤醒 Executor
- **Reactor**：事件监听者，等待外部事件就绪后触发 Waker
- **Executor**：任务调度器，管理就绪队列和任务生命周期
- **Delay Future**：利用事件驱动机制的异步延迟任务

相比上一章的盲目轮询，现在的 Runtime 完全是事件驱动的——不再猜什么时候 `poll`，而是等事件来了再 `poll`。但目前的运行时只支持单个 Future 的执行，下一章我们将实现**并发任务调度**，支持 JoinAll、Then、Select 等组合器。
