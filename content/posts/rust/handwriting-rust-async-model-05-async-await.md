+++
title = '手写 Rust Async Model（五）：async/await 揭秘'
date = 2026-04-26T14:00:00+08:00
draft = false
description = '手写多步状态机并与 async/await 版本对比，揭示编译器语法糖背后的真相'
categories = ['rust']
tags = ['rust', 'async', 'state-machine', 'compiler']
series = '手写 Rust Async Model'
seriesWeight = 5
+++

## 背景 & 问题：async/await 到底做了什么？

前面四章我们手写了 Future Trait、Reactor-Executor 模型、组合器和 Pin。但日常使用 Rust 异步时，我们几乎从不手写 `poll()`——只需要 `async` 和 `await` 两个关键字：

```rust
async fn do_something() -> String {
    some_async_op().await;
    another_async_op().await;
    "done".to_string()
}
```

这两个关键字到底做了什么？编译器在背后生成了什么代码？

答案：**`async/await` 是语法糖，编译器会将它转换为一个状态机**。这一章我们手写一个状态机，然后与 `async/await` 版本对比，彻底搞清楚它的本质。

## 方案：手写状态机 vs async/await

我们要实现的功能很简单：等待 100ms，打印一条消息，再等待 100ms，再打印一条消息，然后返回一个字符串。两步异步操作，对应两个 `.await` 点。

分别用两种方式实现：

1. **手写状态机**：手动定义枚举状态和 `poll()` 逻辑
2. **`async/await`**：用语法糖写出等价逻辑

对比两者的行为，揭示编译器做了什么。

## 实现步骤

### 第一步：定义基础 Future（SimpleDelay）

两种版本都需要一个简单的延迟 Future 作为基础：

```rust
/// 一个简单的延迟 Future
struct SimpleDelay {
    when: Instant,
}

impl SimpleDelay {
    fn new(ms: u64) -> Self {
        Self {
            when: Instant::now() + Duration::from_millis(ms),
        }
    }
}

impl Future for SimpleDelay {
    type Output = ();

    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Self::Output> {
        if Instant::now() >= self.when {
            Poll::Ready(())
        } else {
            Poll::Pending
        }
    }
}
```

### 第二步：手写多步状态机

手动定义一个四状态的状态机，模拟编译器生成的代码：

```rust
/// 手写的多步状态机（模拟 async fn 编译结果）
struct HandWrittenFuture {
    state: HandWrittenState,
}

enum HandWrittenState {
    /// 初始状态，还没开始
    Start,
    /// 等待第一个延迟
    WaitFirst(SimpleDelay),
    /// 等待第二个延迟
    WaitSecond(SimpleDelay),
    /// 已完成
    Done,
}
```

四个状态对应异步函数的执行阶段：

- `Start`：函数体开始执行
- `WaitFirst`：第一个 `.await` 点，等待 `SimpleDelay::new(100)`
- `WaitSecond`：第二个 `.await` 点，等待 `SimpleDelay::new(100)`
- `Done`：函数执行完毕

`poll()` 实现：

```rust
impl Future for HandWrittenFuture {
    type Output = String;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = unsafe { self.get_unchecked_mut() };

        loop {
            match &mut this.state {
                HandWrittenState::Start => {
                    println!("[手写版] 开始");
                    this.state = HandWrittenState::WaitFirst(SimpleDelay::new(100));
                }
                HandWrittenState::WaitFirst(delay) => {
                    match Pin::new(delay).poll(cx) {
                        Poll::Pending => return Poll::Pending,
                        Poll::Ready(()) => {
                            println!("[手写版] 第一步完成");
                            this.state = HandWrittenState::WaitSecond(SimpleDelay::new(100));
                        }
                    }
                }
                HandWrittenState::WaitSecond(delay) => {
                    match Pin::new(delay).poll(cx) {
                        Poll::Pending => return Poll::Pending,
                        Poll::Ready(()) => {
                            println!("[手写版] 第二步完成");
                            this.state = HandWrittenState::Done;
                            return Poll::Ready("完成".to_string());
                        }
                    }
                }
                HandWrittenState::Done => panic!("Future polled after completion"),
            }
        }
    }
}
```

注意 `loop` + `match` 的设计模式：

- **立即推进**：`loop` 让无等待的状态转换（如 `Start → WaitFirst`）一气呵成，避免不必要的 `Pending` 返回
- **按需暂停**：子 Future 未就绪时返回 `Pending`，控制权交还 Executor，下次 `poll` 从当前状态继续
- **状态内嵌**：枚举变体直接存储子 Future 实例（如 `WaitFirst(SimpleDelay)`），状态在 `poll` 之间自然保留

### 第三步：async/await 版本

同样的逻辑，用 `async/await` 写出：

```rust
/// 同样的逻辑，用 async/await 写出
async fn async_version() -> String {
    println!("[async版] 开始");

    SimpleDelay::new(100).await;
    println!("[async版] 第一步完成");

    SimpleDelay::new(100).await;
    println!("[async版] 第二步完成");

    "完成".to_string()
}
```

就这么简单。编译器在背后做了什么？

1. 分析 `async fn` 函数体，识别所有 `.await` 点
2. 为每个 `.await` 点生成一个枚举变体（状态）
3. 生成状态机结构体和 `poll()` 方法

**`async/await` 本质就是语法糖**，让你用同步风格的代码写出异步逻辑，降低开发者的心智负担。

## 示例 & 踩坑

### 两个版本逐行对比

| 手写状态机 | async/await 版本 | 说明 |
|---|---|---|
| `HandWrittenState::Start` | `async fn async_version()` 的函数体开始 | 第一个状态 |
| `println!("[手写版] 开始")` | `println!("[async版] 开始")` | 初始状态下的同步代码 |
| `WaitFirst(SimpleDelay::new(100))` | `SimpleDelay::new(100).await` | 创建子 Future 并开始等待 |
| `HandWrittenState::WaitFirst(delay)` | 第一个 `.await` 点 | 暂停点 |
| `Pin::new(delay).poll(cx)` 返回 `Pending` | `.await` 让出控制权 | 子 Future 未完成 |
| `Pin::new(delay).poll(cx)` 返回 `Ready(())` | `.await` 恢复执行 | 子 Future 完成 |
| `WaitSecond(SimpleDelay::new(100))` | `SimpleDelay::new(100).await` | 第二个暂停点 |
| `return Poll::Ready("完成")` | `"完成".to_string()` | 函数返回 |

### .await 的本质

`.await` 在编译后的代码中做了三件事：

1. **调用子 Future 的 `poll()`**
2. 若返回 `Ready(val)`，将值提取出来，继续执行后续代码
3. 若返回 `Pending`，保存当前状态机的一切状态，将控制权交还给 Executor

> 注意：`.await` 是一个**非阻塞等待**，它不会让当前线程停下来。它更像一个检查点——"子 Future 好了吗？好了就继续，没好就暂停"。

### 运行两种版本

```rust
fn main() {
    println!("=== 手写状态机版 ===");
    let start = Instant::now();
    let result = mini_block_on(HandWrittenFuture::new());
    println!("结果: {}，耗时: {:?}\n", result, start.elapsed());

    println!("=== async/await 版 ===");
    let start = Instant::now();
    let result = mini_block_on(async_version());
    println!("结果: {}，耗时: {:?}\n", result, start.elapsed());

    println!("两种写法的行为完全一致！");
}
```

### 最小 Executor

为了运行这两个 Future，需要一个最小化的 Executor：

```rust
/// 创建一个 no-op Waker（仅用于演示，不会真正唤醒）
fn noop_waker() -> Waker {
    fn noop_clone(_: *const ()) -> RawWaker {
        noop_raw_waker()
    }
    fn noop(_: *const ()) {}

    fn noop_raw_waker() -> RawWaker {
        RawWaker::new(std::ptr::null(), &VTABLE)
    }

    static VTABLE: RawWakerVTable = RawWakerVTable::new(noop_clone, noop, noop, noop);

    unsafe { Waker::from_raw(noop_raw_waker()) }
}

/// 最小化 block_on（busy poll，仅用于演示）
fn mini_block_on<F: Future>(future: F) -> F::Output {
    let mut future = Box::pin(future);
    let waker = noop_waker();
    let mut cx = Context::from_waker(&waker);

    loop {
        match future.as_mut().poll(&mut cx) {
            Poll::Ready(val) => return val,
            Poll::Pending => thread::sleep(Duration::from_millis(10)),
        }
    }
}
```

> 注意：这个 `mini_block_on` 使用了 busy polling，仅用于演示。生产环境请使用第二章的事件驱动版本。

## 本章完整代码

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll, RawWaker, RawWakerVTable, Waker};
use std::thread;
use std::time::{Duration, Instant};

// ================== 手写状态机版本 ==================

/// 一个简单的延迟 Future
struct SimpleDelay {
    when: Instant,
}

impl SimpleDelay {
    fn new(ms: u64) -> Self {
        Self {
            when: Instant::now() + Duration::from_millis(ms),
        }
    }
}

impl Future for SimpleDelay {
    type Output = ();

    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Self::Output> {
        if Instant::now() >= self.when {
            Poll::Ready(())
        } else {
            Poll::Pending
        }
    }
}

/// 手写的多步状态机（模拟 async fn 编译结果）
struct HandWrittenFuture {
    state: HandWrittenState,
}

enum HandWrittenState {
    Start,
    WaitFirst(SimpleDelay),
    WaitSecond(SimpleDelay),
    Done,
}

impl HandWrittenFuture {
    fn new() -> Self {
        Self {
            state: HandWrittenState::Start,
        }
    }
}

impl Future for HandWrittenFuture {
    type Output = String;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = unsafe { self.get_unchecked_mut() };

        loop {
            match &mut this.state {
                HandWrittenState::Start => {
                    println!("[手写版] 开始");
                    this.state = HandWrittenState::WaitFirst(SimpleDelay::new(100));
                }
                HandWrittenState::WaitFirst(delay) => {
                    match Pin::new(delay).poll(cx) {
                        Poll::Pending => return Poll::Pending,
                        Poll::Ready(()) => {
                            println!("[手写版] 第一步完成");
                            this.state = HandWrittenState::WaitSecond(SimpleDelay::new(100));
                        }
                    }
                }
                HandWrittenState::WaitSecond(delay) => {
                    match Pin::new(delay).poll(cx) {
                        Poll::Pending => return Poll::Pending,
                        Poll::Ready(()) => {
                            println!("[手写版] 第二步完成");
                            this.state = HandWrittenState::Done;
                            return Poll::Ready("完成".to_string());
                        }
                    }
                }
                HandWrittenState::Done => panic!("Future polled after completion"),
            }
        }
    }
}

// ================== async/await 版本 ==================

/// 同样的逻辑，用 async/await 写出
async fn async_version() -> String {
    println!("[async版] 开始");

    SimpleDelay::new(100).await;
    println!("[async版] 第一步完成");

    SimpleDelay::new(100).await;
    println!("[async版] 第二步完成");

    "完成".to_string()
}

// ================== 最小 Executor ==================

fn noop_waker() -> Waker {
    fn noop_clone(_: *const ()) -> RawWaker {
        noop_raw_waker()
    }
    fn noop(_: *const ()) {}

    fn noop_raw_waker() -> RawWaker {
        RawWaker::new(std::ptr::null(), &VTABLE)
    }

    static VTABLE: RawWakerVTable = RawWakerVTable::new(noop_clone, noop, noop, noop);

    unsafe { Waker::from_raw(noop_raw_waker()) }
}

fn mini_block_on<F: Future>(future: F) -> F::Output {
    let mut future = Box::pin(future);
    let waker = noop_waker();
    let mut cx = Context::from_waker(&waker);

    loop {
        match future.as_mut().poll(&mut cx) {
            Poll::Ready(val) => return val,
            Poll::Pending => thread::sleep(Duration::from_millis(10)),
        }
    }
}

// ================== main ==================

fn main() {
    println!("=== 手写状态机版 ===");
    let start = Instant::now();
    let result = mini_block_on(HandWrittenFuture::new());
    println!("结果: {}，耗时: {:?}\n", result, start.elapsed());

    println!("=== async/await 版 ===");
    let start = Instant::now();
    let result = mini_block_on(async_version());
    println!("结果: {}，耗时: {:?}\n", result, start.elapsed());

    println!("两种写法的行为完全一致！");
}
```

## 总结

这一章我们通过手写状态机和 `async/await` 版本的对比，揭示了语法糖背后的真相：

- **`async fn`** 编译后生成一个状态机结构体，每个 `.await` 点对应一个枚举变体
- **`.await`** 本质是调用子 Future 的 `poll()`，`Ready` 则继续，`Pending` 则暂停并保存状态
- **`loop` + `match`** 是编译器生成的 `poll()` 方法的标准模式
- **`async/await` 就是语法糖**——它让你用同步风格写出异步代码，但编译器做了所有繁重的状态机管理工作

到这里，我们已经从最底层的 Future Trait 一直走到了 `async/await` 语法糖。最后一章，我们将把手写的模型与生产级运行时 Tokio 做一个全面对比，看看还差多远。
