+++
title = 'ConcurrentHashMap 原理分析'
date = 2026-03-16T11:05:04+08:00
draft = false
description = '深入理解 Java 并发容器 ConcurrentHashMap 的实现原理'

# 主分类
categories = ['java']

# 标签（支持多个）
tags = ['java', 'concurrency', 'source-code', 'interview']

# 系列文章
series = 'Java 并发编程系列'
seriesWeight = 1

# 阅读时间（可选，Hugo 会自动计算）
# readingTime = 10
+++

## 概述

`ConcurrentHashMap` 是 Java 并发包中最重要的并发容器之一，本文将深入分析其实现原理。

## 核心特点

1. **线程安全**：无需额外同步措施即可在多线程环境下使用
2. **高性能**：采用分段锁/ CAS 机制，减少锁竞争
3. **弱一致性**：迭代器不保证强一致性，但不会抛出 `ConcurrentModificationException`

## JDK 1.7 实现：分段锁

### Segment 结构

```java
static final class Segment<K,V> extends ReentrantLock {
    transient volatile HashEntry<K,V>[] table;
    // ...
}
```

## JDK 1.8 实现：CAS + synchronized

### 核心改进

- 取消 Segment 分段，直接使用 Node 数组
- 使用 CAS + synchronized 实现更细粒度的锁
- 红黑树优化链表过长问题

## 总结

ConcurrentHashMap 的演进体现了 Java 在并发领域的持续优化。
