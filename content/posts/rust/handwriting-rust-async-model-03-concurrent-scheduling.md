+++
title = '手写 Rust Async Model（三）：并发任务调度'
date = 2026-04-26T12:00:00+08:00
draft = false
description = '实现 JoinAll、Then、Select 三种 Future 组合器，掌握并发任务调度的核心模式'
categories = ['rust']
tags = ['rust', 'async']
series = '手写 Rust Async Model'
seriesWeight = 3
+++

## 背景 & 问题：单个 Future 不够用

上一章我们实现了一个完整的 Reactor-Executor 模型，但还有一个问题没解决——真实场景中经常需要**多个 Future 协作**：

- 多个 async task 同时存在，需要等待全部完成
- IO/timer 的完成顺序是乱序的
- CPU 需要复用（不能一个 task 占一个线程）

解决方案：构建**并发任务调度系统**，实现最基础的三种组合模式：

- **JoinAll**：等待全部完成
- **Then**：串联执行
- **Select**：竞速取最快

## 方案：Future 组合器

组合器（Combinator）本身也是一个 Future，它内部持有子 Future，通过特定的策略来驱动子 Future 的 `poll()`。

### Timer Future：带返回值的延迟

第二章的 `Delay` 返回 `()`，但组合器需要子 Future 返回有意义的值。所以先引入 `Timer`——一个类似 `Delay` 但返回 `String` 消息的 Future：

```rust
/// 延迟指定时间后返回消息的 Future
pub struct Timer {
    when: Instant,
    registered: bool,
    message: String,
}

impl Timer {
    pub fn new(ms: u64, message: impl Into<String>) -> Self {
        Self {
            when: Instant::now() + Duration::from_millis(ms),
            registered: false,
            message: message.into(),
        }
    }
}

impl MyFuture for Timer {
    type Output = String;

    fn poll(&mut self, waker: &Waker) -> PollState<Self::Output> {
        if Instant::now() >= self.when {
            let msg = self.message.clone();
            println!("{} done", msg);
            return PollState::Ready(msg);
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
```

## 实现步骤

### JoinAll：等待所有任务完成

`JoinAll` 接收一组 Future，等待全部完成后，将所有结果收集到一个 `Vec` 中返回。

```rust
/// 等待所有子 Future 完成
pub struct JoinAll<F: MyFuture> {
    /// 存储子 Future 及其完成结果
    /// (已完成的值, 子 Future)
    items: Vec<(Option<F::Output>, F)>,
}

impl<F: MyFuture> JoinAll<F> {
    pub fn new(futures: Vec<F>) -> Self {
        Self {
            items: futures.into_iter().map(|f| (None, f)).collect(),
        }
    }
}
```

为什么用 `Option` 存储结果？因为子 Future 的完成顺序不确定，要等所有 `Option` 都为 `Some` 才说明全部完成。

`poll` 实现：

```rust
impl<F: MyFuture> MyFuture for JoinAll<F> {
    type Output = Vec<F::Output>;

    fn poll(&mut self, waker: &Waker) -> PollState<Self::Output> {
        let mut pending_count = 0;

        for (result, fut) in self.items.iter_mut() {
            if result.is_some() {
                continue; // 跳过已完成的 Future
            }

            match fut.poll(waker) {
                PollState::Ready(val) => {
                    *result = Some(val);
                }
                PollState::Pending => {
                    pending_count += 1;
                }
            }
        }

        if pending_count == 0 {
            // 所有子 Future 都已完成，收集结果
            let results: Vec<F::Output> = self
                .items
                .iter_mut()
                .map(|(r, _)| r.take().unwrap())
                .collect();
            PollState::Ready(results)
        } else {
            PollState::Pending
        }
    }
}
```

每次调用 `poll()`，`JoinAll` 会遍历所有子项，驱动未完成的子 Future，直到全部完成才返回结果。

使用示例：

```rust
let results = block_on(JoinAll::new(vec![
    Timer::new(300, "任务A"),
    Timer::new(200, "任务B"),
    Timer::new(100, "任务C"),
]));
println!("全部完成: {:?}", results);
// 输出顺序取决于完成顺序，可能是 ["任务C", "任务B", "任务A"]
```

### Then：链式调用

`Then` 使用串行模式：前一个 Future 的输出作为后一个 Future 的输入。

```rust
/// 链式调用：第一个 Future 完成后，用其结果启动第二个
pub struct Then<F, NextF, C>
where
    F: MyFuture,
    NextF: MyFuture,
    C: FnOnce(F::Output) -> NextF,
{
    first: Option<F>,
    second: Option<NextF>,
    callback: Option<C>,
}

impl<F, NextF, C> Then<F, NextF, C>
where
    F: MyFuture,
    NextF: MyFuture,
    C: FnOnce(F::Output) -> NextF,
{
    pub fn new(first: F, callback: C) -> Self {
        Self {
            first: Some(first),
            second: None,
            callback: Some(callback),
        }
    }
}
```

`poll()` 实现：

```rust
impl<F, NextF, C> MyFuture for Then<F, NextF, C>
where
    F: MyFuture,
    NextF: MyFuture,
    C: FnOnce(F::Output) -> NextF,
{
    type Output = NextF::Output;

    fn poll(&mut self, waker: &Waker) -> PollState<Self::Output> {
        // 1. poll 第一个 Future
        if let Some(mut first) = self.first.take() {
            match first.poll(waker) {
                PollState::Ready(val) => {
                    // 第一个完成，用闭包生成第二个 Future
                    let cb = self.callback.take().unwrap();
                    self.second = Some(cb(val));
                }
                PollState::Pending => {
                    // 第一个未完成，放回去等待下次 poll
                    self.first = Some(first);
                    return PollState::Pending;
                }
            }
        }

        // 2. Poll 第二个 Future
        match self.second.as_mut() {
            Some(second) => second.poll(waker),
            None => unreachable!("Then polled after completion"),
        }
    }
}
```

使用示例：

```rust
let result = block_on(Then::new(
    Timer::new(500, "第一步"),
    |msg| Timer::new(300, format!("{} → 第二步", msg)),
));
println!("最终结果: {}", result);
// 输出: 最终结果: 第一步 → 第二步
```

### Select：竞速

`Select` 同时运行两个 Future，任意一个完成即返回。常用于超时控制、竞速请求等场景。

首先定义结果标记枚举：

```rust
/// 竞速结果的标记
pub enum Either<T1, T2> {
    First(T1),
    Second(T2),
}
```

`Select` 结构和实现：

```rust
/// 竞速：任一 Future 完成即返回
pub struct Select<F1, F2> {
    f1: Option<F1>,
    f2: Option<F2>,
}

impl<F1: MyFuture, F2: MyFuture> Select<F1, F2> {
    pub fn new(f1: F1, f2: F2) -> Self {
        Self {
            f1: Some(f1),
            f2: Some(f2),
        }
    }
}

impl<F1: MyFuture, F2: MyFuture> MyFuture for Select<F1, F2> {
    type Output = Either<F1::Output, F2::Output>;

    fn poll(&mut self, waker: &Waker) -> PollState<Self::Output> {
        // 先 poll 第一个 Future
        if let Some(mut f1) = self.f1.take() {
            if let PollState::Ready(val) = f1.poll(waker) {
                return PollState::Ready(Either::First(val));
            }
            self.f1 = Some(f1); // 未完成则放回去
        }

        // 再 poll 第二个 Future
        if let Some(mut f2) = self.f2.take() {
            if let PollState::Ready(val) = f2.poll(waker) {
                return PollState::Ready(Either::Second(val));
            }
            self.f2 = Some(f2); // 未完成，放回去
        }

        // 两个都未完成
        PollState::Pending
    }
}
```

每次 `poll()` 时，先尝试 `poll` 第一个 Future，若 `Ready` 则立即返回；否则 `poll` 第二个 Future；两个都 `Pending` 则返回 `Pending`。

使用示例：

```rust
match block_on(Select::new(
    Timer::new(1000, "慢任务"),
    Timer::new(200, "快任务"),
)) {
    Either::First(msg) => println!("胜出(左): {}", msg),
    Either::Second(msg) => println!("胜出(右): {}", msg),
}
// 输出: 胜出(右): 快任务
```

## 示例 & 踩坑

### 三种组合器对比

| 组合器 | 语义 | 返回值 | 典型场景 |
|---|---|---|---|
| `JoinAll` | 等待全部完成 | `Vec<F::Output>` | 并发请求多个 API，全部返回后聚合结果 |
| `Then` | 串联执行 | `NextF::Output` | 先获取 token，再用 token 请求用户数据 |
| `Select` | 竞速取最快 | `Either<F1::Output, F2::Output>` | 超时控制、多源竞速、请求取消 |

### 简化版 block_on

本章的组合器演示只需要运行单个 Future，所以使用简化版的 `block_on`，不需要第二章 Executor 的完整任务管理：

```rust
/// 简化版 block_on：事件驱动地执行单个 Future
fn block_on<F: MyFuture>(mut future: F) -> F::Output {
    let queue = Arc::new(Mutex::new(VecDeque::new()));
    let current_thread = thread::current();

    loop {
        let waker = Waker {
            id: 0,
            queue: queue.clone(),
            thread: current_thread.clone(),
        };

        match future.poll(&waker) {
            PollState::Ready(val) => return val,
            PollState::Pending => {
                // 没有 wake 信号就 park 等待
                thread::park();
            }
        }
    }
}
```

相比第二章的 Executor：

- 不需要任务 ID 管理：只有一个 Future，固定为 `id: 0`
- 不需要 HashMap：只有一个 Future，直接作为局部变量持有
- 不需要就绪队列的销毁逻辑：`park()` 被 `unpark()` 唤醒后，直接重新 `poll` 那唯一的 Future

### 当前模型的局限

- **没有 work-stealing**：任务绑定在创建它的线程上，不能跨线程迁移
- **负载不均衡**：某些线程忙，某些线程闲

生产级运行时（如 tokio）使用**工作窃取算法**解决这个问题——空闲线程可以从忙碌线程"偷"任务。

## 本章完整代码

```rust
use std::collections::VecDeque;
use std::sync::{Arc, Mutex};
use std::thread::{self, Thread};
use std::time::{Duration, Instant};

// ================== 核心抽象 ==================

pub enum PollState<T> {
    Ready(T),
    Pending,
}

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

// ================== Timer Future ==================

/// 延迟指定时间后返回消息的 Future
pub struct Timer {
    when: Instant,
    registered: bool,
    message: String,
}

impl Timer {
    pub fn new(ms: u64, message: impl Into<String>) -> Self {
        Self {
            when: Instant::now() + Duration::from_millis(ms),
            registered: false,
            message: message.into(),
        }
    }
}

impl MyFuture for Timer {
    type Output = String;

    fn poll(&mut self, waker: &Waker) -> PollState<Self::Output> {
        if Instant::now() >= self.when {
            let msg = self.message.clone();
            println!("{} done", msg);
            return PollState::Ready(msg);
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

// ================== 组合器 ==================

/// 等待所有子 Future 完成
pub struct JoinAll<F: MyFuture> {
    items: Vec<(Option<F::Output>, F)>,
}

impl<F: MyFuture> JoinAll<F> {
    pub fn new(futures: Vec<F>) -> Self {
        Self {
            items: futures.into_iter().map(|f| (None, f)).collect(),
        }
    }
}

impl<F: MyFuture> MyFuture for JoinAll<F> {
    type Output = Vec<F::Output>;

    fn poll(&mut self, waker: &Waker) -> PollState<Self::Output> {
        let mut pending_count = 0;

        for (result, fut) in self.items.iter_mut() {
            if result.is_some() {
                continue;
            }

            match fut.poll(waker) {
                PollState::Ready(val) => {
                    *result = Some(val);
                }
                PollState::Pending => {
                    pending_count += 1;
                }
            }
        }

        if pending_count == 0 {
            let results: Vec<F::Output> = self
                .items
                .iter_mut()
                .map(|(r, _)| r.take().unwrap())
                .collect();
            PollState::Ready(results)
        } else {
            PollState::Pending
        }
    }
}

/// 链式调用：第一个 Future 完成后，用其结果启动第二个
pub struct Then<F, NextF, C>
where
    F: MyFuture,
    NextF: MyFuture,
    C: FnOnce(F::Output) -> NextF,
{
    first: Option<F>,
    second: Option<NextF>,
    callback: Option<C>,
}

impl<F, NextF, C> Then<F, NextF, C>
where
    F: MyFuture,
    NextF: MyFuture,
    C: FnOnce(F::Output) -> NextF,
{
    pub fn new(first: F, callback: C) -> Self {
        Self {
            first: Some(first),
            second: None,
            callback: Some(callback),
        }
    }
}

impl<F, NextF, C> MyFuture for Then<F, NextF, C>
where
    F: MyFuture,
    NextF: MyFuture,
    C: FnOnce(F::Output) -> NextF,
{
    type Output = NextF::Output;

    fn poll(&mut self, waker: &Waker) -> PollState<Self::Output> {
        if let Some(mut first) = self.first.take() {
            match first.poll(waker) {
                PollState::Ready(val) => {
                    let cb = self.callback.take().unwrap();
                    self.second = Some(cb(val));
                }
                PollState::Pending => {
                    self.first = Some(first);
                    return PollState::Pending;
                }
            }
        }

        match self.second.as_mut() {
            Some(second) => second.poll(waker),
            None => unreachable!("Then polled after completion"),
        }
    }
}

/// 竞速结果的标记
pub enum Either<T1, T2> {
    First(T1),
    Second(T2),
}

/// 竞速：任一 Future 完成即返回
pub struct Select<F1, F2> {
    f1: Option<F1>,
    f2: Option<F2>,
}

impl<F1: MyFuture, F2: MyFuture> Select<F1, F2> {
    pub fn new(f1: F1, f2: F2) -> Self {
        Self {
            f1: Some(f1),
            f2: Some(f2),
        }
    }
}

impl<F1: MyFuture, F2: MyFuture> MyFuture for Select<F1, F2> {
    type Output = Either<F1::Output, F2::Output>;

    fn poll(&mut self, waker: &Waker) -> PollState<Self::Output> {
        if let Some(mut f1) = self.f1.take() {
            if let PollState::Ready(val) = f1.poll(waker) {
                return PollState::Ready(Either::First(val));
            }
            self.f1 = Some(f1);
        }

        if let Some(mut f2) = self.f2.take() {
            if let PollState::Ready(val) = f2.poll(waker) {
                return PollState::Ready(Either::Second(val));
            }
            self.f2 = Some(f2);
        }

        PollState::Pending
    }
}

// ================== 简化版 block_on ==================

fn block_on<F: MyFuture>(mut future: F) -> F::Output {
    let queue = Arc::new(Mutex::new(VecDeque::new()));
    let current_thread = thread::current();

    loop {
        let waker = Waker {
            id: 0,
            queue: queue.clone(),
            thread: current_thread.clone(),
        };

        match future.poll(&waker) {
            PollState::Ready(val) => return val,
            PollState::Pending => {
                thread::park();
            }
        }
    }
}

// ================== main ==================

fn main() {
    // === JoinAll：等待所有任务 ===
    println!("=== JoinAll ===");
    let results = block_on(JoinAll::new(vec![
        Timer::new(300, "任务A"),
        Timer::new(200, "任务B"),
        Timer::new(100, "任务C"),
    ]));
    println!("全部完成: {:?}", results);
    println!();

    // === Then：链式调用 ===
    println!("=== Then ===");
    let result = block_on(Then::new(
        Timer::new(500, "第一步"),
        |msg| Timer::new(300, format!("{} → 第二步", msg)),
    ));
    println!("最终结果: {}", result);
    println!();

    // === Select：竞速 ===
    println!("=== Select ===");
    match block_on(Select::new(
        Timer::new(1000, "慢任务"),
        Timer::new(200, "快任务"),
    )) {
        Either::First(msg) => println!("胜出(左): {}", msg),
        Either::Second(msg) => println!("胜出(右): {}", msg),
    }
}
```

## 总结

这一章我们实现了三种核心 Future 组合器：

- **JoinAll**：并发执行多个 Future，等待全部完成后收集结果
- **Then**：串行执行两个 Future，前一个的输出作为后一个的输入
- **Select**：竞速执行，任一 Future 完成即返回

组合器本身也是 Future，它们通过持有子 Future 并在 `poll()` 中驱动子 Future 的方式工作。这三种模式覆盖了异步编程中最常见的并发控制场景。

但目前的 Future 还有一个根本性的安全问题没解决——`async fn` 编译后的 Future 可能形成**自引用结构**，一旦被 move 就会出问题。下一章我们将引入 `Pin` 来解决这个问题。
