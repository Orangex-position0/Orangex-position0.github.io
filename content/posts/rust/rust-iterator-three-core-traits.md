+++
title = 'Rust 迭代器的三个核心 Trait：Iterator、IntoIterator、FromIterator'
date = 2026-06-09T14:00:00+08:00
draft = false
description = '从 trait 定义到 for 循环语法糖，再到自定义集合的完整迭代器支持，看 Iterator、IntoIterator、FromIterator 三个 trait 如何协作'

categories = ['rust']
tags = ['rust', 'iterator']
+++

## 引言

Rust 标准库中，迭代器无处不在。`for` 循环、`map`/`filter`/`collect` 链式调用、`vec!.into_iter()` —— 这些日常操作背后，由三个 trait 协同支撑：

| Trait          | 职责             | 一句话         |
| -------------- | ---------------- | -------------- |
| `Iterator`     | 逐个产出元素     | "一个一个给我" |
| `IntoIterator` | 把集合变成迭代器 | "变成迭代器"   |
| `FromIterator` | 把迭代器变成集合 | "收集成集合"   |

三者可以理解为一个数据管线：`IntoIterator` 将集合拆成逐个元素的管线入口，`Iterator` 在管线上做加工，`FromIterator` 把管线出口重新打包为集合。

```text
集合 --<IntoIterator>--> 迭代器 --<Iterator(加工)>--> 迭代器 --<FromIterator>--> 集合
```

这篇重点讲三件事：

- 三个 trait 各自的定义和设计意图
- `for` 循环的解糖原理和 `collect()` 的底层机制
- 如何为自定义集合实现完整的迭代器支持

前置知识：需要你熟悉 Rust 的 trait、泛型、生命周期和所有权机制。

## IntoIterator：集合到迭代器的桥梁

`IntoIterator` 将容器转换为迭代器。

使用场景：让容器类型可直接用于 `for` 循环，或自定义容器委托已有迭代器实现。

### Trait 定义

```rust
trait IntoIterator {
    // 迭代器产生的元素类型
    type Item;
    // 转换后得到的迭代器类型
    type IntoIter: Iterator<Item = Self::Item>;

    // 将当前类型转换为迭代器
    fn into_iter(self) -> Self::IntoIter;
}
```

- `type Item` 迭代器产生的元素类型
- `type IntoIter` 转换后得到的迭代器类型，必须实现 `Iterator<Item = Self::Item>`
- `fn into_iter(self)` 消费 `self`，返回迭代器。注意参数是 `self` 不是 `&self`，调用 `into_iter()` 会拿走所有权

### for 循环的本质

Rust `for x in collection` 是语法糖，底层依赖 `IntoIterator` trait，不要求类型实现 `Iterator`：

```rust
// 你写的：
for val in &sv {
    println!("{val}");
}

// 编译器看到的（简化版）：
let mut iter = (&sv).into_iter();

while let Some(val) = iter.next() {
    println!("{val}");
}
```

编译器默认调用 `into_iter()` 获取迭代器，再不断调用 `next()` 获取元素。更完整的展开：

```rust
let values = vec![1, 2, 3, 4, 5];
{
    let result = match IntoIterator::into_iter(values) {
        mut iter => loop {
            let next;
            match iter.next() {
                Some(val) => next = val,
                None => break,
            };
            let x = next;
            let () = { println!("{x}"); };
        },
    };
    result
}
```

所以 `for x in values` 之后 `values` 不可用，所有权被 `into_iter(self)` 拿走了。

### 三种迭代形式

标准库中的集合类型通常提供三种迭代方式：

| 方法          | 语义         | 产出的元素类型 |
| ------------- | ------------ | -------------- |
| `iter()`      | 借用迭代     | `&T`           |
| `iter_mut()`  | 可变借用迭代 | `&mut T`       |
| `into_iter()` | 消费迭代     | `T`            |

集合类型会为 `&Collection`、`&mut Collection` 和 `Collection`（自身）分别实现 `IntoIterator`：

```rust
let mut values = vec![41];

// &Vec<T> 实现了 IntoIterator，委托给 iter()
for x in &values {
    // x: &i32
}

// &mut Vec<T> 实现了 IntoIterator，委托给 iter_mut()
for x in &mut values {
    // x: &mut i32
    *x += 1;
}

// Vec<T> 实现了 IntoIterator，委托给 into_iter()
for x in values {
    // x: i32，values 被消费
}
```

`for x in &values` 等价于 `for x in values.iter()`，`for x in &mut values` 等价于 `for x in values.iter_mut()`。

### Blanket Impl

标准库有一个关键的 blanket implementation：

```rust
impl<I: Iterator> IntoIterator for I {
    type Item = I::Item;
    type IntoIter = I;
    fn into_iter(self) -> I {
        self
    }
}
```

所有实现了 `Iterator` 的类型自动实现 `IntoIterator`，这意味着任何自定义迭代器都可以直接用在 `for` 循环中，接受 `impl IntoIterator` 参数的函数既可以传集合也可以传迭代器：

```rust
fn print_all<I: IntoIterator>(iterable: I) {
    for item in iterable {
        println!("{item:?}");
    }
}

let v = vec![1, 2, 3];
print_all(&v);          // 传 &Vec<i32>
print_all(v.iter());    // 传 std::slice::Iter<i32>
print_all(1..=5);       // 传 std::ops::RangeInclusive<i32>
```

## Iterator：迭代器的核心

实现 `Iterator` 的类型本身就是一个迭代器，它定义了许多 Combinator Methods 用来对元素进行加工。

使用场景：遍历元素、数据转换与过滤、链式数据处理。

### Trait 定义

```rust
pub trait Iterator {
    // 当前迭代器产生的元素类型
    type Item;

    // 获取下一个元素
    fn next(&mut self) -> Option<Self::Item>;

    // Combinator Methods，用于执行加工操作
    // fn map(...)
    // fn filter(...)
    // fn fold(...)
    // fn collect(...)
    // ...其余 76+ 个方法全部有默认实现，都建立在 next() 之上
}
```

- `type Item` 当前迭代器产生的元素类型
- `next()` 返回 `Some(Item)` 表示还有元素，`None` 表示结束。`&mut self` 说明每次调用会修改内部状态

Rust 的迭代器是惰性的，仅创建一个迭代器什么都不会发生，只有调用 `next()` 或消费适配器时才开始工作。

### Combinator Methods

这些默认方法可以分为两大类。

**Consuming Adapters**，消耗原迭代器，返回结果：

```rust
let v = vec![1, 2, 3];

let total: i32 = v.iter().sum();          // 求和
let count = v.iter().count();             // 计数
let v2: Vec<i32> = v.iter()
    .map(|x| x + 1)
    .collect();                           // 收集成集合
```

调用后迭代器被消耗，不能再次使用。

**Iterator Adapters**，不消耗原迭代器，返回新的迭代器：

```rust
let v = vec![1, 2, 3, 4, 5];

let result: Vec<i32> = v.iter()
    .filter(|&&x| x % 2 == 0)
    .map(|&x| x * 10)
    .collect();
assert_eq!(result, vec![20, 40]);
```

Iterator Adapters 也是惰性的，上面如果去掉 `.collect()`，`filter` 和 `map` 里的闭包不会执行，编译器会警告：

```
warning: unused `Map` that must be used
 = note: iterators are lazy and do nothing unless consumed
```

迭代器适配器都是泛型结构体，编译器在单态化之后能将整个迭代器链内联优化。Rust Book 第 13 章给出了一个基准测试，在《福尔摩斯探案集》全文中搜索单词，迭代器版本和手写 `for` 循环性能几乎相同，甚至略快。

### 自定义迭代器

两步就够了：定义一个结构体保存状态，然后为它实现 `Iterator`。一个从 1 数到 5 的迭代器：

```rust
struct Counter {
    count: usize,
}

impl Counter {
    fn new() -> Self {
        Counter { count: 0 }
    }
}

impl Iterator for Counter {
    type Item = usize;

    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;
        if self.count < 6 {
            Some(self.count)
        } else {
            None
        }
    }
}
```

实现了 `Iterator` 之后，所有 Combinator Methods 全部可用：

```rust
let sum: usize = Counter::new().sum();
assert_eq!(sum, 15);

let doubled: Vec<usize> = Counter::new().map(|x| x * 2).collect();
assert_eq!(doubled, vec![2, 4, 6, 8, 10]);
```

## FromIterator：迭代器到集合的桥梁

`FromIterator` 允许用迭代器中产出的元素流来构建一个具体类型。

使用场景：构建集合（`.collect()` 底层机制）、类型转换、错误处理（`Result` 实现了 `FromIterator`，可以实现 fail-fast）。

### Trait 定义

```rust
trait FromIterator<A>: Sized {
    // 从一个迭代器构建目标类型
    fn from_iter<T: IntoIterator<Item = A>>(iter: T) -> Self;
}
```

- `A` 元素类型
- 接受任何实现了 `IntoIterator<Item = A>` 的类型作为输入

### .collect() 的本质

`FromIterator` 是 `.collect()` 的底层支撑。`collect()` 的简化逻辑：

```rust
fn collect<B: FromIterator<Self::Item>>(self) -> B {
    FromIterator::from_iter(self)
}
```

```rust
let doubled: SortedVec<i32> = sv
    .into_iter()
    .filter(|&x| x > 1)
    .map(|x| x * 2)
    .collect();
```

对于上面的代码，之所以用 `.collect()` 就能生成 `SortedVec<i32>`，是因为你用 `: SortedVec<i32>` 显式指定了目标类型，编译器去找 `impl FromIterator<i32> for SortedVec<i32>`，并调用其 `from_iter()` 方法生成目标集合。

当 turbofish 嵌套过深时，`FromIterator::from_iter()` 的写法更清晰：

```rust
// 两种写法等价
let deque: VecDeque<i32> = (0..10).collect();
let deque = VecDeque::from_iter(0..10);
```

### 标准库中的实现

几乎所有标准库集合都实现了 `FromIterator`：

```rust
// Vec<T>
let v: Vec<i32> = (1..=5).collect();

// HashMap<K, V>
let map: HashMap<&str, i32> = [("a", 1), ("b", 2)].into_iter().collect();

// Result<Vec<T>, E> - 收集多个 Result
let results: Vec<Result<i32, &str>> = vec![Ok(1), Ok(2), Ok(3)];
let collected: Result<Vec<i32>, &str> = results.into_iter().collect();
assert_eq!(collected, Ok(vec![1, 2, 3]));

// Option<Vec<T>> - 收集多个 Option
let options: Vec<Option<i32>> = vec![Some(1), Some(2), Some(3)];
let collected: Option<Vec<i32>> = options.into_iter().collect();
assert_eq!(collected, Some(vec![1, 2, 3]));
```

`Result` 和 `Option` 的 `FromIterator` 实现让 `.collect()` 变成短路操作，遇到第一个 `Err` 或 `None` 就停止收集并返回。

## 总结

| Trait          | 核心方法                                                   | 职责               | 何时实现                                                  |
| -------------- | ---------------------------------------------------------- | ------------------ | --------------------------------------------------------- |
| `IntoIterator` | `fn into_iter(self) -> Self::IntoIter`                     | 集合转迭代器       | 为自定义集合实现（通常提供 `&T`、`&mut T`、`T` 三种版本） |
| `Iterator`     | `fn next(&mut self) -> Option<Self::Item>`                 | 加工迭代器中的元素 | 为自定义迭代器结构体实现                                  |
| `FromIterator` | `fn from_iter<I: IntoIterator<Item = A>>(iter: I) -> Self` | 迭代器转集合       | 为自定义集合实现，支持 `.collect()`                       |

## 实战：为自定义集合实现完整的迭代器支持

目标：为一个保持元素有序的 `SortedVec` 集合实现三个核心 trait，让它能用在 `for` 循环中，也能用 `.collect()` 构建。

先看骨架，试试自己能不能补完（提示在注释中）：

```rust
use std::iter::FromIterator;

#[derive(Debug)]
struct SortedVec<T> {
    data: Vec<T>,
}

impl<T: Ord> SortedVec<T> {
    fn new() -> Self {
        SortedVec { data: Vec::new() }
    }

    fn push(&mut self, item: T) {
        // 提示：用 iter().position() 找到第一个大于 item 的位置
        // 提示：用 Vec::insert 在该位置插入
        todo!()
    }
}

// 提示：为 SortedVec<T> 实现 IntoIterator
// - Item 类型是什么？
// - IntoIter 可以直接复用 Vec 的迭代器类型 std::vec::IntoIter<T>
// - into_iter 做什么？把 self.data 变成迭代器

// 提示：为 &'a SortedVec<T> 实现 IntoIterator（借用版本）
// - Item 类型是 &'a T
// - IntoIter 是 std::slice::Iter<'a, T>
// - into_iter 委托给 self.data.iter()

// 提示：为 SortedVec<T> 实现 FromIterator<T>
// - 遍历迭代器，对每个元素调用 push
// - 约束 T: Ord（因为 push 需要比较大小）

fn main() {
    let mut sv = SortedVec::from_iter([5, 1, 3, 2, 4]);
    sv.push(0);
    assert_eq!(sv.data, vec![0, 1, 2, 3, 4, 5]);

    for val in &sv {
        println!("{val}");
    }

    let doubled: SortedVec<i32> = sv
        .into_iter()
        .filter(|&x| x > 1)
        .map(|x| x * 2)
        .collect();
    assert_eq!(doubled.data, vec![4, 6, 8, 10]);
}
```

完整实现：

```rust
use std::iter::FromIterator;

#[derive(Debug)]
struct SortedVec<T> {
    data: Vec<T>,
}

impl<T: Ord> SortedVec<T> {
    fn new() -> Self {
        SortedVec { data: Vec::new() }
    }

    fn push(&mut self, item: T) {
        let pos = self.data.iter()
            .position(|x| x > &item)
            .unwrap_or(self.data.len());
        self.data.insert(pos, item);
    }
}

// IntoIterator（消费版本）
impl<T> IntoIterator for SortedVec<T> {
    type Item = T;
    type IntoIter = std::vec::IntoIter<T>;

    fn into_iter(self) -> Self::IntoIter {
        self.data.into_iter()
    }
}

// IntoIterator（借用版本）
impl<'a, T> IntoIterator for &'a SortedVec<T> {
    type Item = &'a T;
    type IntoIter = std::slice::Iter<'a, T>;

    fn into_iter(self) -> Self::IntoIter {
        self.data.iter()
    }
}

// FromIterator
impl<T: Ord> FromIterator<T> for SortedVec<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self {
        let mut sv = SortedVec::new();
        for item in iter {
            sv.push(item);
        }
        sv
    }
}

fn main() {
    let mut sv = SortedVec::from_iter([5, 1, 3, 2, 4]);
    sv.push(0);
    assert_eq!(sv.data, vec![0, 1, 2, 3, 4, 5]);

    for val in &sv {
        println!("{val}");
    }

    let doubled: SortedVec<i32> = sv
        .into_iter()
        .filter(|&x| x > 1)
        .map(|x| x * 2)
        .collect();
    assert_eq!(doubled.data, vec![4, 6, 8, 10]);
}
```

`SortedVec` 内部是 `Vec`，直接复用 `Vec` 的迭代器，不需要自定义 `Iterator` 实现（写法见上文 `Counter` 示例）。需要手写的是 `IntoIterator`（借用和消费两种版本）和 `FromIterator`（支持 `.collect()`）。

## References

- [The Rust Programming Language, Ch.13 - Iterators and Closures](https://doc.rust-lang.org/book/ch13-02-iterators.html)
- [std::iter::Iterator - Rust 标准库文档](https://doc.rust-lang.org/std/iter/trait.Iterator.html)
- [Rust Iterators Beyond the Basics - JetBrains Blog](https://blog.jetbrains.com/rust/2024/03/12/rust-iterators-beyond-the-basics-part-ii-key-aspects/)
- [IntoIterator - Comprehensive Rust (Google)](https://google.github.io/comprehensive-rust/iterators/intoiterator.html)
- [Rust Collections Explained - YouTube](https://www.youtube.com/watch?v=oiWATcjyUEI)
