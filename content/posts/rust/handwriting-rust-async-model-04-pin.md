+++
title = '手写 Rust Async Model（四）：自引用和 Pin'
date = 2026-04-26T13:00:00+08:00
draft = false
description = '为什么 async 需要固定内存地址？从自引用结构的问题出发，理解 PhantomPinned 和 Pin 的设计'
categories = ['rust']
tags = ['rust', 'async', 'self-referential']
series = '手写 Rust Async Model'
seriesWeight = 4
+++

## 背景 & 问题：async 为何需要 Pin？

前面三章我们手写了 Future 和相应的运行时，但有一个关键的安全问题一直没提——`async fn` 编译后的 Future，本质是一个"状态机结构体"，而这个状态机**可能形成自引用结构**。

什么是自引用结构？就是结构体内部的某个字段**引用了自身的另一个字段**：

```
旧地址 A
  ├── a (值: 42)
  └── b → 指向 a 的地址

move 后变成新地址 B
  ├── a (值: 42)     ← 数据被拷贝到了新位置
  └── b → 仍指向旧地址 A.a  ❌ 悬垂指针！
```

在 Rust 中，大多数类型可以在内存中自由移动（move）。但自引用结构一旦被 move，内部指针就会失效，变成**悬垂指针**——这是内存安全的灾难。

`async fn` 编译后的状态机就可能出现这种情况：某个 `.await` 点暂停后，局部变量的引用可能被保存在结构体字段中，形成自引用。如果 Future 被 move，这些引用就失效了。

解决方案：**使用 `Pin` 固定内存地址，禁止 move**。

## 方案：PhantomPinned + Pin

Rust 的解决方案分两步：

1. **`PhantomPinned`**：标记类型为 `!Unpin`，告诉编译器"这个类型不能安全移动"
2. **`Pin<T>`**：一个包装器，保证被包装的值在内存中不会被移动

### PhantomPinned 和 !Unpin

`PhantomPinned` 是标准库提供的**零大小标记类型**，它唯一的作用是让包含它的类型自动变成 `!Unpin`：

```rust
use std::marker::PhantomPinned;

struct MaybeSelfRef {
    a: usize,
    b: Option<NonNull<usize>>, // b 可能指向 a（自引用！）
    _pin: PhantomPinned,       // 让这个类型变成 !Unpin
}
```

`!Unpin` 是一个 marker trait 的否定形式，含义是"这个类型不能安全地被 Pin 解引用"。一旦结构体包含 `PhantomPinned`，编译器就会阻止你用安全代码将它的值从 `Pin` 中取出。

## 实现步骤

### 第一步：复现自引用问题

先用一段代码直观展示自引用结构被 move 后会发生什么：

```rust
use std::marker::PhantomPinned;
use std::ptr::NonNull;

#[derive(Debug)]
struct MaybeSelfRef {
    a: usize,
    // b 可能指向 a 的地址（自引用！）
    b: Option<NonNull<usize>>,
    _pin: PhantomPinned,
}

impl MaybeSelfRef {
    fn new(value: usize) -> Self {
        Self {
            a: value,
            b: None,
            _pin: PhantomPinned,
        }
    }
}

fn main() {
    let mut x = MaybeSelfRef::new(42);
    let mut y = MaybeSelfRef::new(99);

    // 建立 x 的自引用：b 指向 a
    unsafe {
        let a_ptr = NonNull::new_unchecked(&mut x.a as *mut usize);
        x.b = Some(a_ptr);
    }

    println!("建立自引用前:");
    println!("  x.a = {}, 通过 x.b 读取的值 = {}", x.a,
        unsafe { x.b.unwrap().as_ref() });

    // 危险！swap 交换了 x 和 y 的内存内容
    std::mem::swap(&mut x, &mut y);

    println!("\nswap 后:");
    println!("  x.a = {}, 通过 x.b 读取的值 = {}", x.a,
        unsafe { x.b.unwrap().as_ref() });
    println!("  x.b 指向的值 ({}) ≠ x.a ({}) — 悬垂指针！",
        unsafe { *x.b.unwrap().as_ref() }, x.a);
}
```

运行结果：

```
建立自引用前:
  x.a = 42, 通过 x.b 读取的值 = 42

swap 后:
  x.a = 42, 通过 x.b 读取的值 = 99
  x.b 指向的值 (99) ≠ x.a (42) — 悬垂指针！
```

`std::mem::swap` 把 x 和 y 的内存内容互换了。x 的 `a` 被换成了原来的 42，但 `b` 仍然指向原来的地址——而那个地址现在存的是 y 的数据。**自引用被破坏了**。

### 第二步：用 Pin 解决问题

`Pin<T>` 是一个包装器，它保证被包装的值不会被 move。最常见的方式是 `Box::pin()`，将值固定在堆上：

```rust
use std::pin::Pin;

fn main() {
    // Box::pin 将值固定在堆上，之后无法被移动
    let mut pinned = Box::pin(MaybeSelfRef::new(42));

    // 建立自引用
    pinned.as_mut().init();

    println!("建立自引用后:");
    println!("  a = {}", pinned.a);

    // 通过 b 修改 a
    pinned.as_mut().set_a_via_b(100);

    println!("  通过 b 修改后 a = {}", pinned.a);
    println!("  ✅ 自引用仍然有效！");

    // 试图移动会编译失败：
    // let moved = pinned; // ❌ 编译错误！MaybeSelfRef: !Unpin
}
```

因为 `MaybeSelfRef` 包含 `PhantomPinned`，它是 `!Unpin` 的。这意味着你不能用安全代码把它从 `Pin` 中取出来，也就无法 move 它。**自引用永远安全**。

### 第三步：实现安全的自引用操作

在 `Pin` 保护下，我们可以安全地操作自引用结构：

```rust
impl MaybeSelfRef {
    /// 建立自引用（必须在 Pin 保护下调用）
    fn init(self: Pin<&mut Self>) {
        // 安全性：因为 Self 是 !Unpin，调用者保证 self 不会被移动
        unsafe {
            let this = self.get_unchecked_mut();
            let a_ptr = NonNull::new_unchecked(&mut this.a as *mut usize);
            this.b = Some(a_ptr);
        }
    }

    /// 通过自引用 b 修改 a 的值
    fn set_a_via_b(self: Pin<&mut Self>, new_val: usize) {
        unsafe {
            let this = self.get_unchecked_mut();
            if let Some(ptr) = this.b {
                *ptr.as_ptr() = new_val;
            }
        }
    }
}
```

注意 `self: Pin<&mut Self>` 签名——它要求调用者必须持有 `Pin<&mut Self>`，从而保证调用期间值不会被 move。

## 示例 & 踩坑

### Pin + Future 的关系

整个 Rust async runtime 的安全保证链如下：

```
Future（状态机）
   ↓
可能包含自引用
   ↓
必须 Pin 固定内存位置
   ↓
Executor poll(Pin<&mut T>)
   ↓
保证 poll 过程中不会 move
```

这就是为什么标准库的 `Future` trait 签名是：

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
```

而不是我们手写版本的 `fn poll(&mut self, ...)`。`Pin<&mut Self>` 保证 Future 在 `poll` 过程中不会被移动，自引用始终有效。

### 为什么不直接禁止所有 move？

只有**可能自引用**的类型才需要 `Pin`。大多数普通类型（如 `i32`、`Vec<T>`、不含 `PhantomPinned` 的结构体）是 `Unpin` 的，可以自由 move。`Pin` 只对 `!Unpin` 类型起限制作用，对 `Unpin` 类型是透明的。

### 本章完整代码

```rust
use std::marker::PhantomPinned;
use std::pin::Pin;
use std::ptr::NonNull;

/// 模拟 async 状态机中的自引用结构
#[derive(Debug)]
struct MaybeSelfRef {
    a: usize,
    // b 可能指向 a 的地址（自引用！）
    b: Option<NonNull<usize>>,
    // PhantomPinned 使这个类型变成 !Unpin
    _pin: PhantomPinned,
}

impl MaybeSelfRef {
    fn new(value: usize) -> Self {
        Self {
            a: value,
            b: None,
            _pin: PhantomPinned,
        }
    }

    /// 建立自引用（必须在 Pin 保护下调用）
    fn init(self: Pin<&mut Self>) {
        // 安全性：因为 Self 是 !Unpin，调用者保证 self 不会被移动
        unsafe {
            let this = self.get_unchecked_mut();
            let a_ptr = NonNull::new_unchecked(&mut this.a as *mut usize);
            this.b = Some(a_ptr);
        }
    }

    /// 通过自引用 b 修改 a 的值
    fn set_a_via_b(self: Pin<&mut Self>, new_val: usize) {
        unsafe {
            let this = self.get_unchecked_mut();
            if let Some(ptr) = this.b {
                *ptr.as_ptr() = new_val;
            }
        }
    }
}

fn main() {
    // === 1. 问题演示：自引用 + swap = 悬垂指针 ===
    println!("=== 自引用问题演示 ===");

    let mut x = MaybeSelfRef::new(42);
    let mut y = MaybeSelfRef::new(99);

    // 建立 x 的自引用：b 指向 a
    unsafe {
        let a_ptr = NonNull::new_unchecked(&mut x.a as *mut usize);
        x.b = Some(a_ptr);
    }

    println!("建立自引用前:");
    println!("  x.a = {}, 通过 x.b 读取的值 = {}", x.a,
        unsafe { x.b.unwrap().as_ref() });

    // 危险！swap 交换了 x 和 y 的内存内容
    // x.b 仍然指向旧地址，但那个地址现在属于 y
    std::mem::swap(&mut x, &mut y);

    println!("\nswap 后:");
    println!("  x.a = {}, 通过 x.b 读取的值 = {}", x.a,
        unsafe { x.b.unwrap().as_ref() });
    println!("  ⚠️  x.b 指向的值 ({}) ≠ x.a ({}) — 悬垂指针！",
        unsafe { *x.b.unwrap().as_ref() }, x.a);

    println!();

    // === 2. 解决方案：Pin 固定内存位置 ===
    println!("=== Pin 解决方案 ===");

    // Box::pin 将值固定在堆上，之后无法被移动
    let mut pinned = Box::pin(MaybeSelfRef::new(42));

    // 建立自引用
    pinned.as_mut().init();

    println!("建立自引用后:");
    println!("  a = {}", pinned.a);

    // 通过 b 修改 a
    pinned.as_mut().set_a_via_b(100);

    println!("  通过 b 修改后 a = {}", pinned.a);
    println!("  ✅ 自引用仍然有效！");

    // 试图移动会编译失败：
    // let moved = pinned; // ❌ 编译错误！MaybeSelfRef: !Unpin
}
```

## 总结

这一章我们解决了 async 编程中一个根本性的安全问题：

- **自引用结构**：结构体内部字段引用自身其他字段，move 后指针失效
- **`PhantomPinned`**：零大小标记类型，让包含它的类型变为 `!Unpin`
- **`Pin<T>`**：禁止 `!Unpin` 类型被 move 的包装器，保证内存地址不变
- **`Pin + Future`**：标准库 `Future` 的 `poll()` 签名使用 `Pin<&mut Self>`，确保状态机在 poll 过程中不会被移动

`Pin` 是 Rust 异步安全的基石——没有它，编译器就无法保证 `async fn` 生成的状态机在多次 `poll` 之间自引用始终有效。下一章我们将揭示 `async/await` 语法糖的本质，看看编译器是如何将异步代码转换为状态机的。
