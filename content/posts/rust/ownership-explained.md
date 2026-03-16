+++
title = 'Rust 所有权系统详解'
date = 2026-03-16T12:00:00+08:00
draft = false
description = '理解 Rust 最核心的特性：所有权系统'

categories = ['rust']
tags = ['rust', 'ownership', 'memory-management', 'fundamentals']
series = 'Rust 入门系列'
seriesWeight = 1
+++

## 什么是所有权

所有权（Ownership）是 Rust 最独特的特性，它使 Rust 无需垃圾回收就能保证内存安全。

## 所有权规则

1. Rust 中每个值都有一个所有者（owner）
2. 值在同一时间只能有一个所有者
3. 当所有者离开作用域，值将被丢弃

## 示例代码

```rust
{
    let s = String::from("hello");  // s 是所有者
    // 使用 s
}  // s 离开作用域，内存被自动释放
```

## 移动语义

```rust
let s1 = String::from("hello");
let s2 = s1;  // s1 的所有权移动到 s2
// println!("{}", s1);  // 错误！s1 已失效
```

## 总结

掌握所有权是学习 Rust 的关键，它带来：
- 内存安全，无需 GC
- 高性能，无运行时开销
- 明确的代码意图
