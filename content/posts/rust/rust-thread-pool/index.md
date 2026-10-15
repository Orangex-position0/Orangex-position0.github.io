+++
title = 'Rust 并发编程实战：从零实现简易线程池'
date = 2026-10-15T14:37:12+08:00
lastmod = 2026-10-15T14:37:12+08:00
draft = false
image = 'cover.png'
description = '使用 Rust 标准库逐步实现简易线程池，理解闭包任务传递、Arc 与 Mutex 共享接收端、有界队列背压及优雅关闭。'
categories = ['rust']
tags = ['rust', 'thread-pool', 'concurrency']
+++

# 用 Rust 实现简易线程池：任务传递、共享接收端与优雅关闭

最近学习 Rust 时，发现一个项目非常适合用来学习 Rust 并发编程：手写简易线程池。原因是我在 The Rust Book 的最后一个练习中发现其编写了一个线程池，虽然是简易版本，但非常适合用来学习相关并发知识。

Prerequisite：要求你已经了解 Rust 的所有权、闭包和 trait，能看懂 `Arc<Mutex<_>>` 的写法

本文只使用标准库，围绕三个问题逐步实现：

1. 闭包任务如何安全地从提交线程转移到工作线程？
2. 多个 worker 如何共享接收端，并在取出任务后并发执行？
3. 线程池关闭时，如何处理完已提交的任务，再等待所有 worker 退出？

这是理解并发机制的教学实现，不是生产级线程池。我们暂时保留 `unwrap()`，把任务 panic、创建失败等边界放到文末讨论。

## 如何跟着代码走

后续代码块如果标注了函数或字段位置，只替换对应部分，不要把它们直接追加到文件末尾。完整可运行版本见第十步。

## 第一步：使用方案

这个简易版本的 ThreadPool 先提供两个函数：

- 构造函数 `new`：传入固定的工作线程数量
- 执行函数 `execute`：传入一个闭包，表示交给工作线程执行的任务

先写出期望的调用方式。这段代码需要配合下一步的接口骨架才能编译。

```rust
fn main() {
    let pool = ThreadPool::new(4);
    let message = String::from("hello from a job");
    pool.execute(move || println!("{message}"));
}
```

这里用 `move` 将 `message` 的所有权交给闭包，让任务拥有打印时需要的数据。我们的目标是让 `execute()` 提交任务，由池中的某个 worker 执行，而不是为每次提交都创建一个新线程。

## 第二步：补齐 ThreadPool 骨架

先定义一个 `ThreadPool` Struct，然后为其实现刚才提到的两个函数：

- 第一个是构造函数，需要注意补充前置校验，传入的线程数必须为正数
- 第二个是执行任务函数，暂时只写 API 契约，不实现内部

将下面的定义放在 `main()` 后面，与第一步的代码组合成当前阶段的完整程序：

```rust
pub struct ThreadPool;

impl ThreadPool {
    pub fn new(size: usize) -> Self {
        assert!(size > 0, "ThreadPool must have at least 1 worker");
        Self
    }

    pub fn execute<F>(&self, _f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        // 暂时只检查传入闭包的类型，不执行任务。
    }
}
```

这个阶段的程序还不会创建线程，也不会打印任务中的消息。`execute()` 接收闭包后直接将其丢弃，参数名 `_f` 的下划线表示暂时不使用它。先让调用方式和类型约束成立，后续再补上提交与执行逻辑。中间阶段可能出现未使用字段的警告，因为后续功能还没有补齐。

这里我们创建一个泛型 `F`，并为其添加了 trait bound。接下来分别解释一下其含义：

- `FnOnce()` 表示闭包可以被消费并调用一次，这里的 `()` 表示不接收参数。线程池只需要对每个任务调用一次，因此使用这个约束，也允许任务在执行时移出捕获的数据。实现了 `Fn` 或 `FnMut` 的闭包也满足 `FnOnce`，所以它们同样可以作为任务提交
- `Send` 表示闭包可以安全地在线程之间转移所有权。这里需要把整个闭包从提交任务的线程交给工作线程，因此闭包捕获的数据也必须满足相应的跨线程传递要求
- `'static` 表示闭包不能携带短于 `'static` 的借用，因为任务可能在 `execute()` 返回后才执行。它不要求闭包一直存活到程序结束；闭包自己拥有的 `String` 等数据也可以满足这个约束

这里要区分 `FnOnce` 和 `move`：`move` 决定闭包按值捕获使用到的外部变量，`FnOnce` 约束闭包如何被调用。按值捕获不代表调用时一定会把数据移出去，所以 `move` 闭包也可能实现 `Fn` 或 `FnMut`。另外，`move` 捕获一个引用时，得到的仍然是引用，并不会自动让它满足 `'static`。

### 先预测：加了 `move`，就一定满足 `'static` 吗？

暂时用下面的 `main()` 替换第一步的调用示例，保留当前的 ThreadPool 骨架。先不要编译，判断：闭包已经加了 `move`，能通过 `execute()` 的类型检查吗？

```rust
// 预测实验：预期无法编译。
fn main() {
    let pool = ThreadPool::new(4);
    let message = String::from("hello from a job");
    let message_ref = &message;
    pool.execute(move || println!("{message_ref}"));
}
```

再运行 `cargo check`，检查编译器指出的是哪个借用。

这里移进闭包的是 `message_ref` 这个引用，原来的 `String` 仍属于 `main()`。这个引用借用的是局部变量，不能满足 `execute()` 要求的 `'static`。即使此时的骨架还不执行任务，编译器也会检查方法声明中的约束。

删除 `let message_ref = &message;`，将提交语句恢复为：

```rust
pool.execute(move || println!("{message}"));
```

再运行 `cargo check`，这次可以通过。闭包拥有 `String`，不再借用 `main()` 中的局部数据。满足 `'static` 不意味着这个 `String` 永远不会销毁；它会随持有它的任务一起被正常释放。

继续下一步前，恢复第一步的 `main()`。

## 第三步：创建 Worker

写好了 ThreadPool 骨架，现在要考虑一个问题：ThreadPool 还没有创建任何的工作线程？

先给 ThreadPool 增加保存 worker 的字段，并同步修改构造函数。用下面的结构体替换第二步的空结构体，用新的 `new()` 替换旧构造函数；`execute()` 暂时保持不变。

```rust
use std::thread;

pub struct ThreadPool {
    workers: Vec<Worker>,
}

impl ThreadPool {
    pub fn new(size: usize) -> Self {
        assert!(size > 0, "ThreadPool must have at least 1 worker");
        let mut workers = Vec::with_capacity(size);
        for id in 0..size {
            workers.push(Worker::new(id));
        }
        Self { workers }
    }
    // 保留第二步的 execute()
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize) -> Self {
        let thread = thread::spawn(|| {});
        Self { id, thread }
    }
}
```

`Worker` 保存两项元数据：`id` 是工作线程编号，`thread` 是线程句柄。此时工作线程执行空闭包后就会退出，还不会等待或处理任务。

另外，`Vec::with_capacity(size)` 只预分配空间，不会创建 worker。真正创建工作线程的是循环里调用的 `Worker::new()`。

### 线程句柄

Rust 的 OS Thread 会存在一个线程句柄，用于作为一个间接访问的工具。通过工作线程的句柄，我们可以在不了解其底层逻辑的前提下，就能管理线程的完成等待、并获取返回结果。

## 第四步：建立任务通道

当前问题：`execute` 和工作线程之间没有联系，而线程句柄用于等待线程结束，不能向线程发送任务。所以我们需要一条从提交者到 worker 的传递路径

解决思路：用一个任务队列连接两端，让提交的任务能够通过队列传递给工作线程。Rust 中可使用标准库的 `mpsc::channel()`，线程池保存发送端，worker 后续通过接收端获取任务。

但这里还有一个问题：被传递的任务不能用 ThreadPool 或 Worker 来表示，需要再创建一个专门的类型。暂时用空 struct 来表示队列中一个待执行的任务。

将文件顶部的导入替换为下面的 `use`，新增占位类型 `Job`，并替换 ThreadPool 的结构体和构造函数。保留已有的 `execute()` 和 Worker 定义。此阶段 `_receiver` 尚未交给 worker，会在构造函数返回时被丢弃，通道还不能用于提交任务。

```rust
use std::{sync::mpsc, thread};

struct Job;

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

impl ThreadPool {
    pub fn new(size: usize) -> Self {
        assert!(size > 0);
        let (sender, _receiver) = mpsc::channel::<Job>();
        let mut workers = Vec::with_capacity(size);
        for id in 0..size {
            workers.push(Worker::new(id));
        }
        Self { workers, sender }
    }
    // execute 和 Worker 暂时不变。
}
```

- `ThreadPool` 增加一个字段 `sender`，用于存放发送者，它们负责将任务发给 channel，工作线程可从 channel 中取任务
- 那相应的，`ThreadPool` 的构造函数中也要创建 channel

## 第五步：让多个 Worker 共享任务接收端

背景问题：每个 worker 都需要从同一个队列取任务，那需要依次传入 receiver 吗？

先做一个预期无法编译的尝试：把第四步构造函数中的 `_receiver` 改名为 `receiver`，再修改循环和 `Worker::new()`，直接把接收端传进去。

```rust
// ThreadPool::new 内：
for id in 0..size {
    workers.push(Worker::new(id, receiver));
}

// Worker::new 的尝试：
fn new(id: usize, receiver: mpsc::Receiver<Job>) -> Self {
    let thread = thread::spawn(move || {
        let _ = receiver.recv();
    });
    Self { id, thread }
}
```

然后你会发现编译失败：第一次循环已经把 receiver 的所有权移进了 `Worker::new`，后续循环不能再使用它。标准库的 `mpsc::Receiver<T>` 也没有实现 `Clone`，不能通过克隆接收端来解决。

我们希望多个 worker 共同使用同一个接收端，谁取到任务就由谁执行，每个任务只交给一个 worker，而不是广播给所有 worker。

这里需要解决两个问题：

1. 多个 worker 如何共同持有 receiver？使用 `Arc` 共享所有权
2. 多个 worker 如何安全地访问 receiver？`Receiver<T>` 没有实现 `Sync`，不能仅靠 `Arc` 让多个线程共享访问。使用 `Mutex` 保护接收端，同一时间只允许一个 worker 调用 `recv()`

因此，这里使用 `Arc<Mutex<mpsc::Receiver<Job>>>`：`Arc` 让多个 worker 共同持有接收端，`Mutex` 让它们轮流接收任务。克隆 `Arc` 只会增加对同一个对象的引用计数，不会复制接收端或队列中的任务。

下面改为可编译的版本：替换文件顶部的导入，以及 ThreadPool 和 Worker 各自的 `new()`。这些方法分别放在对应的 `impl` 块内，其余代码保持不变。

```rust
use std::{sync::{mpsc, Arc, Mutex}, thread};

// ThreadPool::new：
pub fn new(size: usize) -> Self {
    assert!(size > 0);
    let (sender, receiver) = mpsc::channel::<Job>();
    let receiver = Arc::new(Mutex::new(receiver));
    let mut workers = Vec::with_capacity(size);
    for id in 0..size {
        workers.push(Worker::new(id, Arc::clone(&receiver)));
    }
    Self { workers, sender }
}

// Worker::new：
fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Self {
    let thread = thread::spawn(move || {
        let _ = receiver.lock().unwrap().recv();
    });
    Self { id, thread }
}
```

## 第六步：优化 Job，实现 execute 函数

背景问题：线程任务需要以闭包的形式传入，但现在占位用的 `Job` 空结构体还不能保存闭包，所以需要修改。

分析问题：每个闭包表达式都有自己的匿名类型和大小，接口需要接收不同类型的闭包，但队列需要统一的消息格式。

解决思路：让 `Job` 表示统一的任务类型，而 `execute()` 只需要接收具体闭包，再装箱为统一的任务类型即可。

用下面的类型别名替换 `struct Job;`，再替换 ThreadPool 的 `execute()` 方法。这个阶段只完成发送，worker 仍然只接收一次并丢弃任务，因此还不能用第一步的示例验证任务执行。

```rust
type Job = Box<dyn FnOnce() + Send + 'static>;

// ThreadPool::execute：
pub fn execute<F>(&self, f: F)
where
    F: FnOnce() + Send + 'static,
{
    let job: Job = Box::new(f);
    self.sender.send(job).unwrap();
}
```

这里用 `type` 关键字建立一个类型别名，为什么这样写：

| 部分             | 作用                                     |
| ---------------- | ---------------------------------------- |
| `FnOnce()`       | 消费闭包执行一次，支持移出捕获的数据     |
| `Send`           | 闭包能够从提交线程转移给 worker          |
| `'static`        | 不携带可能提前失效的短生命周期借用       |
| `Box<dyn ...>`   | 堆分配与类型擦除，让不同闭包进入同一队列 |
| `type Job = ...` | 给长类型取别名，不创建新的独立类型       |

那 `execute()` 的处理流程就是：

1. 调用者传入闭包
2. 将闭包类型装箱并转为统一类型 `Job`
3. 将任务所有权交给 channel
4. worker 后续从 channel 中接收任务并调用（下一步实现）

## 第七步：让 Worker 持续接收 channel 的消息并执行

背景问题：当前的 worker 只接收一次消息，而且还不调用任务。提交入口已经能发送闭包了，但任务还是会被丢弃。

解决思路：在工作线程中实现一个循环：“接收任务 -> 执行 -> 再次等待”。锁只保护接收过程，不覆盖任务执行，不然其他 worker 无法取任务。

替换 `Worker::new()` 中创建线程的语句，保留后面的 `Self { id, thread }`。

```rust
// Worker::new
let thread = thread::spawn(move || loop {
    let job = receiver.lock().unwrap().recv().unwrap();
    println!("worker {id} executing");
    job();
});
```

- 关于 `let job ...`：
    - `lock()` 用于获取互斥锁，若其他 worker 持有锁就等待
    - 第一个 `unwrap()` 用于取出 `MutexGuard`，通过它来访问内部的 Receiver
    - `recv()` 用于从 channel 中取出一个任务；队列为空时，持有锁并等待任务
    - 第二个 `unwrap()` 用于取出收到的 `Job`
    - 所以其职责就是**取出任务**，并存入变量 job
- 后面的 `job()` 是在调用前面 job 变量中的闭包，等于**执行这个任务**
- 所以取任务是加锁的、串行的；但执行任务可以并行

### 锁到底在哪里释放？

`lock()` 返回的 `MutexGuard` 持有锁，guard 被销毁时才会解锁。在上面的写法中，guard 是临时值，会在 `let job = ...;` 这条语句结束时销毁。`job` 保存的是接收到的闭包，不是 guard，所以执行 `job()` 时已经不再持锁，其他 worker 可以继续取任务。

如果把 guard 存到变量里，并让它一直活到任务执行结束，情况就不同了：

```rust
// 对照写法：替换 Worker::new 中的 spawn，故意让锁覆盖任务执行。
let thread = thread::spawn(move || loop {
    let guard = receiver.lock().unwrap();
    let job = guard.recv().unwrap();
    println!("worker {id} executing");
    job();
    drop(guard); // 执行完任务后才解锁。
});
```

这段代码即使创建多个 worker，任务仍然只能串行执行：一个 worker 执行任务时，其他 worker 都拿不到锁，无法接收下一个任务。显式写出 `drop(guard)` 是为了标明解锁位置；如果删掉它，guard 也要等到本次循环体结束才会销毁，并不会因为后面没有再使用就提前解锁。

### 验证任务是否完成

先给 `main()` 增加完成通知，验证 worker 是否能持续接收并完成任务。这个断言检查本次运行收到的完成通知，不证明任务之间存在并发。

```rust
fn main() {
    let pool = ThreadPool::new(2);
    let (done_tx, done_rx) = mpsc::channel();
    for i in 0..4 {
        let done_tx = done_tx.clone();
        pool.execute(move || {
            println!("job {i}");
            done_tx.send(i).unwrap();
        });
    }
    drop(done_tx);
    let mut completed: Vec<_> = done_rx.iter().collect();
    completed.sort_unstable();
    assert_eq!(completed, vec![0, 1, 2, 3]);
}
```

### 对照观察并发与串行

先不运行代码，分别对前面的两种 worker 写法作出预测：**一个任务还没有结束时，另一个 worker 能取到任务并开始执行吗？** 找出每种写法的解锁位置，再给出判断。

然后分别使用这两种写法，运行下面的 `main()`。每个任务打印开始和结束，中间休眠一段时间，方便观察执行是否重叠。

```rust
fn main() {
    let pool = ThreadPool::new(2);
    let (done_tx, done_rx) = mpsc::channel();
    for i in 0..4 {
        let done_tx = done_tx.clone();
        pool.execute(move || {
            println!("job {i}: starting");
            std::thread::sleep(std::time::Duration::from_millis(200));
            println!("job {i}: finished");
            done_tx.send(i).unwrap();
        });
    }
    drop(done_tx);
    let mut completed: Vec<_> = done_rx.iter().collect();
    completed.sort_unstable();
    assert_eq!(completed, vec![0, 1, 2, 3]);
}
```

观察两个现象：

- **先解锁再执行**：通常能看到一个任务尚未结束，另一个任务已经开始。这说明任务的执行区间发生了重叠
- **持锁执行**：一个任务打印 `finished` 后，下一个任务才能打印 `starting`。任务执行期间一直占着锁，其他 worker 无法取任务

线程调度不保证固定的日志顺序，也不保证每次运行都能观察到重叠。一次没有看到重叠，不能据此判定代码只能串行；持锁版本只能串行，则可以从锁的范围直接推导出来。这里的休眠用于展示执行重叠，不是 CPU 并行计算的性能测试。

实验后，恢复先解锁再执行任务的 worker 写法，再继续下一步。

当前版本尚未处理通道关闭，接收端断开后 `recv().unwrap()` 可能使 worker panic。下一步再解决 worker 如何正常退出的问题。

## 第八步：通过 Drop 完成优雅关闭

背景问题：线程池离开作用域时，如何等待所有 worker 结束，实现 graceful shutdown 呢？

初步想法：先添加下面这个预期无法编译的 `Drop` 实现，看看直接等待线程会遇到什么问题。

```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        for worker in &mut self.workers {
            worker.thread.join().unwrap();
        }
    }
}
```

但是你会发现编译失败，因为循环只有 `&mut Worker`（可变借用），但 `JoinHandle::join(self)` 消费句柄，不能从借用的 worker 中直接把字段 move 出来。那替换成循环呢？

```rust
for worker in self.workers.drain(..) {
    println!("joining worker {}", worker.id);
    worker.thread.join().unwrap();
}
```

用这段循环替换上面 `drop()` 的函数体后能编译，但在任务正常完成的情况下，运行时会等待不退出。因为 sender 还存活，worker 会继续等待新任务；而 `join()` 只等待，不主动停止线程，最终导致关闭过程阻塞。

再次分析问题原因：worker 在等待新任务，主线程却在等待 worker 结束。需要谁进行什么操作后，才能打破这个等待？

解决方案：**关闭任务入口，让 worker 知道不会再有新任务，然后让它们处理完队列，再结束等待并退出**（Graceful Shutdown 的思路）

整个流程

1. 停止提交，关闭任务入口
2. worker 继续处理队列中的剩余任务
3. 队列为空且不会有新任务，worker 结束等待并退出
4. 主线程等待所有 worker 退出

落实到 Rust 代码，就是**先断开发送端，再等待 worker**。下面是分别位于不同位置的替换片段，不是可以整体粘贴到文件末尾的代码。其中 `Option::take()` 把 sender 从字段中取出，留下 `None`，随后显式销毁取出的发送端。

通道必须等所有发送端都被销毁，才会在剩余任务被取完后返回接收错误。本实现不向外暴露或克隆任务发送端，所以销毁池内的 sender 就能关闭任务入口。这个关闭过程仍然要求任务最终能够返回；如果任务一直阻塞，`drop(pool)` 也会一直等待。

```rust
// ThreadPool 字段：
sender: Option<mpsc::Sender<Job>>,

// new 的返回表达式：
Self { workers, sender: Some(sender) }

// execute 的发送语句：
self.sender.as_ref().unwrap().send(job).unwrap();

// Worker::new 中替换 spawn：
let thread = thread::spawn(move || loop {
    let result = receiver.lock().unwrap().recv();
    match result {
        Ok(job) => {
            println!("worker {id} executing");
            job();
        }
        Err(_) => {
            println!("worker {id} shutting down");
            break;
        }
    }
});

// 替换 Drop 实现：
impl Drop for ThreadPool {
    fn drop(&mut self) {
        // 使用 Option::take() 返回原来的 Some(sender)
        drop(self.sender.take());
        // 使用 drain() 将 worker 取出来，并将所有权也转出来
        for worker in self.workers.drain(..) {
            println!("joining worker {}", worker.id);
            worker.thread.join().unwrap();
        }
    }
}
```

## 第九步：有界队列

背景问题：如果任务提交速度长期高于处理速度，无界队列中的待执行任务就会不断积压，增加内存占用。

解决思路：改为有界队列，队列满时发送端会阻塞，让提交者等待 worker 腾出空位。这种通过阻塞提交来限制任务继续涌入的机制，称为背压。Rust 中可以使用 `mpsc::sync_channel(capacity)` 创建有界通道。

```rust
// ThreadPool 字段：
sender: Option<mpsc::SyncSender<Job>>,

// new 内创建 channel：
let (sender, receiver) = mpsc::sync_channel::<Job>(size * 2);
```

这里把等待队列容量设为 worker 数量的两倍，只是教学示例的配置，不是通用的最优值。已经被 worker 取出的任务不占队列位置，所以这个容量不等于线程池能容纳的任务总数。

### 有界队列改变了 execute 的行为

`execute()` 的函数签名没有变化，但现在调用它可能因为队列已满而阻塞。调用者不能再把它理解成“提交后立即返回”。另外，`execute()` 返回只表示任务已交给通道，不表示任务已经执行完成。

有界队列限制的是排队的任务数量，不是内存字节数。每个闭包捕获的数据大小不同，即使任务数量固定，也不能据此给出总内存占用的固定上限。

还需要注意一种死锁场景：如果任务能够向同一个线程池继续提交任务，队列又已经满了，提交操作就会阻塞。如果所有 worker 都阻塞在提交上，就没有 worker 能继续取任务、腾出队列位置。这个简易版本不处理这种情况，使用时应避免从 worker 内向同一个池进行可能阻塞的提交。

下一步的完整示例会记录每次 `execute()` 的耗时。队列满时，后续提交可能明显变慢，这展示的是提交端等待空位，而不是 `execute()` 在等待自己提交的任务完成。具体哪次提交发生阻塞、阻塞多久，取决于当时的线程调度和任务进度，不应写成固定的日志顺序或耗时断言。

## 第十步：最终完整实现

下面是完整程序，可以直接替换 `src/main.rs`，然后在项目根目录执行 `cargo run`。

示例先提交十个任务，再直接销毁线程池，最后核对完成通知。这样可以检查：即使调用者没有提前等待通知，`drop(pool)` 仍会等待任务和 worker 结束。提交耗时用于观察背压，不作为性能结论。

```rust
use std::{
    sync::{Arc, Mutex, mpsc},
    thread,
    time::{Duration, Instant},
};

type Job = Box<dyn FnOnce() + Send + 'static>;

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: Option<mpsc::SyncSender<Job>>,
}

impl ThreadPool {
    /// 创建包含 size 个 worker 的线程池。
    ///
    /// # Panics
    /// size 为 0 或操作系统无法创建线程时 panic。
    pub fn new(size: usize) -> Self {
        assert!(size > 0, "ThreadPool must have at least 1 worker");
        let (sender, receiver) = mpsc::sync_channel::<Job>(size * 2);
        let receiver = Arc::new(Mutex::new(receiver));
        let mut workers = Vec::with_capacity(size);
        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }
        Self {
            workers,
            sender: Some(sender),
        }
    }

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job: Job = Box::new(f);
        self.sender.as_ref().unwrap().send(job).unwrap();
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        // 断开通道，让 worker 读完队列后退出。
        drop(self.sender.take());
        for worker in self.workers.drain(..) {
            println!("joining worker {}", worker.id);
            worker.thread.join().unwrap();
        }
    }
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Self {
        let thread = thread::spawn(move || {
            loop {
                // 本语句结束时解锁；执行 job 时不持锁。
                let result = receiver.lock().unwrap().recv();
                match result {
                    Ok(job) => {
                        println!("worker {id} executing");
                        job();
                    }
                    Err(_) => {
                        println!("worker {id} shutting down");
                        break;
                    }
                }
            }
        });
        Self { id, thread }
    }
}

fn main() {
    let pool = ThreadPool::new(2); // 两个 worker，等待队列容量为四。
    let (done_tx, done_rx) = mpsc::channel();
    for i in 0..10 {
        let done_tx = done_tx.clone();
        println!("submitting job {i}");
        let started = Instant::now();
        pool.execute(move || {
            println!("job {i}: starting");
            thread::sleep(Duration::from_millis(100));
            println!("job {i}: finished");
            done_tx.send(i).unwrap();
        });
        println!("submitted job {i}, execute took {:?}", started.elapsed());
    }
    drop(done_tx);
    drop(pool); // 等待所有任务与 worker 完成。

    let mut completed: Vec<_> = done_rx.iter().collect();
    completed.sort_unstable();
    assert_eq!(completed, (0..10).collect::<Vec<_>>());
    println!("All jobs completed; all workers joined");
}
```

## 扩展

若你想将这个简易版 ThreadPool 扩展为生产实现，还需要实现：

- **错误处理**：`send`、`lock`、`join` 使用 `unwrap()`，没有公开的错误契约。
- **任务 panic 隔离**：任务 panic 会终止 worker，减少池的容量。执行任务时已解锁，因此任务 panic 本身通常不会使 receiver 锁中毒；但 `join().unwrap()` 仍会让关闭过程 panic，在另一次 panic 的栈展开中可能导致二次 panic 并终止进程。
- **创建失败清理**：`thread::spawn` 失败会 panic，可以研究返回 `Result` 的 `thread::Builder::spawn`，以及部分 worker 已创建后的清理。
- **配置和监控**：容量写死为 worker 数量的两倍，没有任务结果 handle、活动任务计数、超时、取消或 worker 补建。

## 学完以后，应该能解释什么

先不看下面的答案，试着独立回答：

1. 多个 worker 共同持有 receiver，为什么只用 `Arc` 还不够？
2. guard 后面没有再被使用，是否就已经释放锁？本文的两种写法各在哪里解锁？
3. 关闭时只调用 `join()`，为什么可能一直等待？应该先做什么？

### 对照答案

1. `Arc` 只解决共享所有权，不会让内部类型自动获得线程安全的共享访问能力。`Receiver<Job>` 没有实现 `Sync`，这里还需要 `Mutex` 让多个 worker 轮流访问接收端。任务本身则作为装箱闭包经 channel 转移，受 `Send` 和 `'static` 约束
2. 没有再使用 guard，不代表它已经销毁。本文的链式接收写法中，临时 guard 在 `let job = ...;` 语句结束时销毁；对照写法中的具名 guard 则在 `drop(guard)` 时销毁，删去显式 `drop` 后会在本次循环体结束时销毁。执行任务前解锁，才能避免这把锁将任务执行串行化
3. `join()` 只等待线程结束，不会通知线程停止。sender 仍存活时，worker 可能一直等待新任务。因此要先销毁任务发送端，让 worker 取完剩余任务、收到通道断开的错误并退出，再通过 `join()` 等待它们结束

这个线程池适合观察同步闭包任务的调度过程。它没有实现执行顺序保证，也不会让彼此依赖、可能互相等待的任务自动避免死锁。

但本文只介绍了简易版线程池，若你还想进一步学习，可以参考两个主流库：

- **Rayon**：面向数据并行和 CPU 密集型计算，提供并行迭代器，并使用工作窃取调度。适合比较这里的共享任务队列与工作窃取的区别
- **Tokio**：异步运行时，调度的是实现 `Future` 的任务。适合学习大量 I/O 等待如何通过异步机制组织；不能把这里的阻塞闭包原样当作普通异步任务使用。阻塞工作需要考虑 `spawn_blocking` 等机制，CPU 密集型计算还应单独考虑并发度限制

# References

- [The Rust Book 21.2](https://doc.rust-lang.org/stable/book/ch21-02-multithreaded.html)
