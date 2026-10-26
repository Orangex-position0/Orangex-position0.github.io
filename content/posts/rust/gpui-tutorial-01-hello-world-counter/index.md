+++
title = 'GPUI Kit 教程 01：从 Hello World 到计数器'
date = 2026-10-26T10:39:49+08:00
lastmod = 2026-10-26T11:11:27+08:00
draft = false
image = 'cover.png'
description = '使用 GPUI Kit 创建第一个 Rust 桌面窗口，通过计数器理解 Render、Entity、点击回调与状态更新，并用实验区分修改状态和通知更新。'
categories = ['rust']
tags = ['rust', 'gpui', 'gpui-kit', 'gui']
series = 'GPUI 教程'
seriesWeight = 1
+++

本教程面向掌握 Rust 基础、第一次接触 GPUI 并想用它来开发 GUI 应用的开发者。你需要了解 struct、trait、泛型和闭包，但不需要有桌面 GUI 或前端开发经验。

本篇通过一个计数器，讲清楚三个问题：界面怎样描述，点击回调怎样修改状态，修改后的状态怎样显示到窗口中。

如果你用过 React、Vue 或 CSS，文中的前端类比可以辅助理解；这些类比不代表两个框架的实现机制相同。

## 1. 最终效果与学习目标

我们会先创建一个显示 `Hello, World!` 和按钮的窗口，点击按钮后在控制台输出消息；然后将它改成计数器，点击按钮后更新窗口中的数字。

完成后，你应该能够：

- 创建一个 View，通过 `Render` 描述界面
- 用元素的父子关系和布局 API 组织界面
- 在 View 中保存计数状态，并通过点击回调修改它
- 解释 `cx.new()`、`cx.listener()` 和 `cx.notify()` 在示例中的作用

本篇只讲清楚从按钮点击到界面更新的过程，不展开完整生命周期、共享状态、订阅和异步任务。

## 2. GPUI 与 GPUI Kit 的分工

GPUI 提供应用、窗口、状态管理、渲染和事件处理等基础能力。GPUI Kit 将 GPUI 与相关基础设施、组件库和资源整合为应用开发入口。

在本教程中：

| 代码                                       | 来自哪一层、负责什么                       |
| ------------------------------------------ | ------------------------------------------ |
| `Render`、`Entity`、`Context`、`div()`     | GPUI 的基础能力，通过 Kit 重新导出         |
| `Button`                                   | Kit 暴露的组件库，提供按钮样式和交互接口   |
| `application()`、`init()`、`open_window()` | Kit 提供或导出的启动入口与初始化、窗口封装 |

因此，示例只依赖 `gpui-kit`，并通过 `gpui_kit` 导入相关 API。理解状态更新时，我们讲的是 GPUI 的机制；使用 `Button` 时，我们用的是 Kit 暴露的现成组件。

## 3. 环境准备与创建项目

本教程从一个独立的 Rust 项目开始，不需要下载配套仓库，也不依赖本地的 GPUI Kit 源码。

### 3.1 安装 Rust 与平台工具

按当前 [GPUI Kit 安装指南](https://gpui-kit.com/docs/installation)，使用 Rust 1.92 或更新的稳定版；后续依赖可能要求更高版本。

桌面应用还需要对应平台的原生构建工具。以下要求来自官方安装指南：

| 平台                  | 准备工作                                                                                                                                                           |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Windows 10 或更新版本 | 安装 Visual Studio 2022 Build Tools 或 Community，勾选“使用 C++ 的桌面开发”、MSVC 和 Windows SDK；安装 CMake 并加入 PATH；使用 MSVC Rust 工具链，不使用 GNU 工具链 |
| macOS 15 或更新版本   | 安装 Xcode Command Line Tools，执行 `xcode-select --install`，再用 `xcode-select -p` 检查开发工具目录                                                              |
| Linux                 | 安装编译器和窗口、字体、图形等系统开发依赖；需要图形桌面会话与可用的 Vulkan 驱动                                                                                   |

Windows 用户还可以执行以下命令检查环境：

```bash
rustup show active-toolchain
cmake --version
```

工具链名称应包含 `msvc`，CMake 应输出版本号。

Linux 的依赖包名称因发行版而异。Ubuntu 24.04 可以按官方清单安装：

```bash
sudo apt update
sudo apt install -y gcc g++ clang pkg-config libfontconfig-dev libwayland-dev   libwebkit2gtk-4.1-dev libxkbcommon-x11-dev libx11-xcb-dev   libssl-dev libzstd-dev libasound2-dev vulkan-validationlayers libvulkan1
```

其中 `libasound2-dev` 用于可选的语音功能，本教程不启用该功能。安装 `libvulkan1` 不等于安装 GPU 驱动；运行时还需要可用的驱动，以及 Wayland 或 X11 图形会话。其他发行版请参照官方安装指南选择等效依赖。

### 3.2 创建独立项目

在你希望存放项目的位置执行：

```bash
cargo new gpui-hello --bin --edition 2024
cd gpui-hello
```

后文路径均相对于 `gpui-hello` 项目目录。

### 3.3 配置依赖

将项目的 `Cargo.toml` 替换为：

```toml
[package]
name = "gpui-hello"
version = "0.1.0"
edition = "2024"

[dependencies]
gpui-kit = "=0.8.0"
```

本教程固定使用 GPUI Kit `0.8.0`，其默认功能已包含组件库和图标资源，无需额外添加 GPUI 或资源包依赖。

`=0.8.0` 只固定 GPUI Kit 本身，不会固定所有间接依赖的版本。首次成功构建后，建议将应用项目生成的 `Cargo.lock` 一并纳入版本管理，方便后续复现依赖组合。

> 验证说明：本节安装方式和依赖版本已对照官方文档及 crates.io 信息核实，但本文尚未完成上述所有平台的实际构建与 GUI 验证。

## 4. 创建第一个 View

本节代码写在 `src/main.rs` 中，完整版本见第 4.3 节。

### 4.1 定义 View 并描述界面

在 GPUI 中，View 表示一个 UI 逻辑单元，封装了 UI 的状态和渲染逻辑。本篇用实现了 `Render` trait 的 Rust struct 来定义 View，并通过 `render()` 描述界面。稍后我们会用 `cx.new()` 将它交给 GPUI 管理。

先从没有字段的 `HelloWorld` 开始：

```rust
use gpui_kit::component::button::{Button, ButtonVariants};
use gpui_kit::*;

struct HelloWorld;

impl Render for HelloWorld {
    fn render(
        &mut self,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .size_full()
            .items_center()
            .justify_center()
            .gap_2()
            .child("Hello, World!")
            .child(
                Button::new("hello")
                    .primary()
                    .label("Click me")
                    .on_click(|_, _, _| {
                        println!("Clicked!");
                    }),
            )
    }
}
```

- `div()` 创建容器元素，`.child()` 添加子元素，两者共同构建 Element Tree。
- `flex()`, `size_full()` 等 Fluent API，用于定义布局和视觉属性，表达方式与 Tailwind CSS 的工具类相似
- `ButtonVariants` 将 `.primary()` 等按钮样式扩展方法引入作用域。`Button::new("hello")` 中的 `hello` 是按钮标识，屏幕上显示的文字由 `.label("Click me")` 设置。
- `render()` 根据 View 的当前状态返回元素描述。`Window` 提供窗口相关能力，`Context<Self>` 提供当前 Entity 的监听、通知等能力，返回值需实现 `IntoElement`。

### 4.2 启动应用并创建窗口

在 `main.rs` 的末尾添加入口函数：

```rust
fn main() {
    application()
        .with_assets(assets::Assets)
        .run(|cx| {
            init(cx);

            open_window(
                WindowOptions::default(),
                cx,
                |_, cx| cx.new(|_| HelloWorld),
            )
            .expect("Failed to open window");
        });
}
```

这段启动代码按顺序完成以下工作：

1. `application()` 创建应用入口，`with_assets(assets::Assets)` 注册 Kit 提供的资源来源。
2. `run()` 启动应用，并执行传入的初始化闭包。
3. `init(cx)` 初始化 Kit 启用的基础层和组件层，应在创建应用窗口前调用。
4. `open_window()` 用默认配置创建窗口，并调用闭包构建窗口内容。
5. `cx.new(|_| HelloWorld)` 创建由 GPUI 管理的 `HelloWorld`，返回 `Entity<HelloWorld>`。

这里的 `cx` 是应用级的 `App` 上下文，而 `render()` 中的 `cx` 是 `Context<HelloWorld>`。它们习惯上都叫 `cx`，但类型和可用能力并不完全相同。

### 4.3 Hello World 完整代码

使用第 3 节的依赖配置，将以下完整代码写入 `src/main.rs`：

```rust
use gpui_kit::component::button::{Button, ButtonVariants};
use gpui_kit::*;

struct HelloWorld;

impl Render for HelloWorld {
    fn render(
        &mut self,
        _window: &mut Window,
        _cx: &mut Context<Self>,
    ) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .size_full()
            .items_center()
            .justify_center()
            .gap_2()
            .child("Hello, World!")
            .child(
                Button::new("hello")
                    .primary()
                    .label("Click me")
                    .on_click(|_, _, _| {
                        println!("Clicked!");
                    }),
            )
    }
}

fn main() {
    application()
        .with_assets(assets::Assets)
        .run(|cx| {
            init(cx);

            open_window(
                WindowOptions::default(),
                cx,
                |_, cx| cx.new(|_| HelloWorld),
            )
            .expect("Failed to open window");
        });
}
```

在项目目录执行：

```bash
cargo run
```

窗口应显示 `Hello, World!` 和 `Click me` 按钮。点击按钮后，终端输出 `Clicked!`。

## 5. Element Tree

GPUI 的 UI 使用 Element Tree 来构建，呈现为树形结构。

`render()` 返回的是界面的元素描述。多个元素通过 `.child()` 形成父子关系，也就是 Element Tree：

```text
容器 div
├── 文本 Hello, World!
└── 按钮 Click me
```

![Element Tree 概念树状图（多层嵌套示意）](<GPUI Kit element tree diagram.png>)

`div()` 创建容器，`.child()` 添加子元素，其他链式方法配置布局和样式：

| API                 | 作用                             | CSS 类比                    |
| ------------------- | -------------------------------- | --------------------------- |
| `.flex()`           | 启用 Flexbox 布局                | `display: flex`             |
| `.flex_col()`       | 子元素纵向排列                   | `flex-direction: column`    |
| `.size_full()`      | 宽高占满父容器                   | `width: 100%; height: 100%` |
| `.items_center()`   | 交叉轴居中                       | `align-items: center`       |
| `.justify_center()` | 主轴居中                         | `justify-content: center`   |
| `.gap_2()`          | 使用框架的间距刻度设置子元素间距 | `gap`，数字不是像素值       |

本例使用 `.flex_col()`，主轴是竖直方向，交叉轴是水平方向。因此，`.justify_center()` 让内容上下居中，`.items_center()` 让内容左右居中。改成 `.flex_row()` 后，两个轴的方向也会交换。

试着将 `.flex_col()` 改为 `.flex_row()`，重新运行，观察文字与按钮是否从纵向排列变为横向排列，再恢复原来的代码。

> 前端类比：元素的父子结构类似 HTML，布局也采用熟悉的 Flexbox 概念。但这里创建的不是浏览器 DOM，链式调用也不是在编写 CSS，不能直接推导出浏览器中的全部行为。

## 6. 状态管理与事件处理

接下来，我们将 Hello World 改成计数器：在 View 中保存计数，点击按钮时递增，并更新窗口中的数字，以此学习 GPUI 的状态管理。

下面按状态定义、渲染与事件处理、应用入口三个部分讲解 Counter，最后提供完整代码。所有代码均位于 `src/main.rs`。

### 6.1 定义 Counter 状态

`Counter` 用于保存计数器的状态：

```rust
struct Counter {
    count: usize,
}
```

`count` 字段保存当前计数，作为 View 的状态在多次渲染之间保留。

### 6.2 根据状态渲染界面并处理点击

`Counter` 的完整 `Render` 实现如下：

```rust
impl Render for Counter {
    fn render(
        &mut self,
        _window: &mut Window,
        cx: &mut Context<Self>,
    ) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .size_full()
            .items_center()
            .justify_center()
            .gap_2()
            .child(format!("Count: {}", self.count))
            .child(
                Button::new("increment")
                    .primary()
                    .label("Increment")
                    .on_click(cx.listener(|this, _, _, cx| {
                        this.count += 1;
                        cx.notify();
                    })),
            )
    }
}
```

与 Hello World 相比，这里新增了状态读取和更新逻辑：

- `.child(format!("Count: {}", self.count))` 根据当前计数生成文字。
- `.on_click(cx.listener(...))` 注册与当前 Entity 关联的点击回调，回调中的 `this` 是 `&mut Counter`。
- `this.count += 1` 修改计数，`cx.notify()` 通知框架当前 Entity 有变化。

按钮无需直接修改文字元素，后续渲染会读取更新后的 `count`。**修改状态和通知更新是两件事，`notify()` 本身不会修改字段。**

Hello World 的点击回调有三个参数，而这里传给 `cx.listener()` 的闭包有四个参数：

```text
普通点击回调：          |event, window, app|
listener 内部的回调：  |this, event, window, cx|
```

`cx.listener()` 将当前 Entity 的访问接入点击回调。在本例中，`this` 是 `&mut Counter`，因此可以通过 `this.count` 修改计数；最后的 `cx` 是 `Context<Counter>`，可以用它通知当前 Entity 有变化。示例中的两个 `_` 分别忽略点击事件和窗口参数。

### 6.3 在入口中创建 Counter

应用入口与 Hello World 基本一致，只需让窗口内容改为 `Counter`：

```rust
fn main() {
    application()
        .with_assets(assets::Assets)
        .run(|cx| {
            init(cx);

            open_window(
                WindowOptions::default(),
                cx,
                |_, cx| cx.new(|_| Counter { count: 0 }),
            )
            .expect("Failed to open window");
        });
}
```

`cx.new(|_| Counter { count: 0 })` 创建初始计数为零的 Counter，返回 `Entity<Counter>`，其余启动流程不变。

### 6.4 完整代码与运行验证

依赖配置不变，将 `src/main.rs` 替换为以下完整代码：

```rust
use gpui_kit::component::button::{Button, ButtonVariants};
use gpui_kit::*;

struct Counter {
    count: usize,
}

impl Render for Counter {
    fn render(
        &mut self,
        _window: &mut Window,
        cx: &mut Context<Self>,
    ) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .size_full()
            .items_center()
            .justify_center()
            .gap_2()
            .child(format!("Count: {}", self.count))
            .child(
                Button::new("increment")
                    .primary()
                    .label("Increment")
                    .on_click(cx.listener(|this, _, _, cx| {
                        this.count += 1;
                        cx.notify();
                    })),
            )
    }
}

fn main() {
    application()
        .with_assets(assets::Assets)
        .run(|cx| {
            init(cx);

            open_window(
                WindowOptions::default(),
                cx,
                |_, cx| cx.new(|_| Counter { count: 0 }),
            )
            .expect("Failed to open window");
        });
}
```

在项目目录执行：

```bash
cargo run
```

检查点：初始显示 `Count: 0`；点击三次后应显示 `Count: 3`。

## 7. 理解 Entity 与状态更新流程

现在回看 `cx.new(|_| Counter { count: 0 })`：创建出来的 `Counter` 由 GPUI 管理，代码中得到的是 `Entity<Counter>` 句柄。

`Entity<T>` 是**指向 GPUI 托管状态对象的强句柄**。克隆句柄不会复制 `Counter`，多个句柄仍然引用同一个状态对象。

这里需要区分状态对象和访问它的句柄：`Counter` 保存计数，GPUI 托管这个对象，`Entity<Counter>` 则用于引用它。克隆句柄不会复制计数器，也不意味着可以任意取得 `&mut Counter`；访问和修改仍需经过框架提供的上下文与更新 API。本例中，`cx.listener()` 帮我们在点击回调里访问当前 Counter。

`Entity<T>` 不要求 `T` 一定是 View，也能管理非 UI 状态。在本例中，`Counter` 实现了 `Render`，所以它同时承担界面状态和渲染描述的职责。

本例的更新过程是：

```text
cx.new() 创建 Counter，初始 count 为 0
    ↓
render() 读取 count，描述文字和按钮
    ↓
用户点击按钮
    ↓
cx.listener() 创建的回调获得 Counter 的可变访问
    ↓
修改 count，并调用 cx.notify()
    ↓
框架安排相关界面更新，后续渲染读取新的 count
```

在这次状态驱动的更新中，变化的是 `Counter` 的字段和后续生成的元素描述，而不是每点击一次就重新执行 `cx.new()` 创建一个计数器。

可以先用这个心智模型理解示例：**Entity 管理需要保留的状态，`render()` 根据当前状态重新描述 Element Tree。** 这并不意味着所有元素内部状态每次都被彻底清空，也不意味着每次通知都会立即、同步执行一次 `render()`。

> 前端对比：这里与 React、Vue 相似的是状态驱动 UI 的思路。`Counter` 的字段不是 Vue 响应式属性，`cx.notify()` 也不是 React 的 `setState`：它不接收新状态，只负责通知变化。不要从元素树的相似性推断 GPUI 使用了 React 的虚拟 DOM 或 Hooks 机制。

## 8. 实验：修改状态不等于通知更新

为了区分状态修改和界面更新，我们分别在点击回调和 `render()` 中打印计数，对比有无通知时的行为。

### 8.1 添加实验日志

在第 6.4 节的 Counter 完整代码中，仅将 `impl Render for Counter { ... }` 整块替换为以下实现。导入、`struct Counter` 和 `main()` 保持不变。

修改点：添加两个 `println!()` 打印

```rust
impl Render for Counter {
    fn render(
        &mut self,
        _window: &mut Window,
        cx: &mut Context<Self>,
    ) -> impl IntoElement {
        println!("render: count = {}", self.count);

        div()
            .flex()
            .flex_col()
            .size_full()
            .items_center()
            .justify_center()
            .gap_2()
            .child(format!("Count: {}", self.count))
            .child(
                Button::new("increment")
                    .primary()
                    .label("Increment")
                    .on_click(cx.listener(|this, _, _, cx| {
                        this.count += 1;
                        println!("click: count = {}", this.count);
                        cx.notify(); // 对照实验时，仅注释这一行
                    })),
            )
    }
}
```

`click:` 日志记录回调修改后的计数，`render:` 日志记录渲染时读取的计数。

### 8.2 对比有无更新通知

1. 保留上述 `cx.notify()`，执行 `cargo run`，点击三次按钮，对比终端日志和窗口数字。
2. 关闭窗口，将上面标注的 `cx.notify()` 一行注释掉，其余代码不变。
3. 再次执行 `cargo run`，同样点击三次，对比日志和窗口数字。每次重新启动都会从 `count: 0` 开始。
4. 实验结束后，恢复 `cx.notify()`，删除两处 `println!` 日志，或直接恢复第 6.4 节的完整代码。

重点观察：点击回调是否仍然执行、字段是否继续变化，以及界面是否随之更新。不要假设一次点击严格对应一次 `render()` 调用。窗口出现悬停、按下等视觉变化，也不代表 Counter 的 `render()` 一定重新执行。这个实验应结合 `click:`、`render:` 日志和窗口数字一起判断，而不是只观察按钮有没有视觉反馈。

本实验用于理解“字段赋值不会自动表达更新意图”，而不是证明“去掉通知后界面在任何情况下都绝不更新”。

## 9. 练习：添加重置按钮

在递增按钮下方添加一个 `Reset` 按钮，使计数归零。保留现有的纵向布局，先独立完成，再展开提示或参考答案。

提示：参考递增按钮的写法，给同一个容器再添加一个按钮。两个按钮操作同一个 Counter，但需要使用不同的按钮标识。

验收标准：

- 递增三次后显示 `Count: 3`
- 点击重置后显示 `Count: 0`
- 再点击递增后显示 `Count: 1`
- 两个按钮使用不同标识

<details>
<summary>实现提示</summary>

添加另一个 `.child(Button::new("reset")...)`，用 `cx.listener()` 访问当前 `Counter`。回调中将 `this.count` 设为 `0`，再通知框架状态有变化。不需要创建新的 Entity，也不需要直接操作文字元素。

</details>

<details>
<summary>参考答案：完整代码</summary>

保持第 3.3 节的依赖配置不变，将以下代码完整替换到 `src/main.rs`，执行 `cargo run`，按本节验收标准检查结果。

```rust
use gpui_kit::component::button::{Button, ButtonVariants};
use gpui_kit::*;

struct Counter {
    count: usize,
}

impl Render for Counter {
    fn render(
        &mut self,
        _window: &mut Window,
        cx: &mut Context<Self>,
    ) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .size_full()
            .items_center()
            .justify_center()
            .gap_2()
            .child(format!("Count: {}", self.count))
            .child(
                Button::new("increment")
                    .primary()
                    .label("Increment")
                    .on_click(cx.listener(|this, _, _, cx| {
                        this.count += 1;
                        cx.notify();
                    })),
            )
            .child(
                Button::new("reset")
                    .label("Reset")
                    .on_click(cx.listener(|this, _, _, cx| {
                        this.count = 0;
                        cx.notify();
                    })),
            )
    }
}

fn main() {
    application()
        .with_assets(assets::Assets)
        .run(|cx| {
            init(cx);

            open_window(
                WindowOptions::default(),
                cx,
                |_, cx| cx.new(|_| Counter { count: 0 }),
            )
            .expect("Failed to open window");
        });
}
```

重置按钮使用独立标识 `reset`，通过 `cx.listener()` 访问同一个 Counter，将 `count` 设为 `0` 后调用 `cx.notify()`。它与递增按钮共用状态，不需要创建新的 Entity。

</details>

完成后，可以测试一下，试着回答：计数保存在哪里？回调怎样访问它？为什么仅修改字段还不足以明确通知框架更新？

## 10. 概念速查与参考资料

| 概念         | 在本例中的作用                              |
| ------------ | ------------------------------------------- |
| Application  | 应用启动与运行环境                          |
| Window       | 承载原生窗口与窗口相关操作                  |
| Entity<T>    | 引用由 GPUI 管理的状态对象的强句柄          |
| View         | 本例中保存 UI 状态、实现 `Render` 的 struct |
| Render       | 根据当前状态描述界面                        |
| Context<T>   | 当前 Entity 的上下文，提供监听、通知等能力  |
| Element Tree | `render()` 构建的元素父子结构               |
| Component    | 可复用的 UI 封装，本例使用 `Button`         |

Hello World 与 Counter 的完整代码分别见第 4.3 节和第 6.4 节，共用第 3.3 节的依赖配置。

继续学习与核对 API 时，可以参考：

- [GPUI Kit 安装指南](https://gpui-kit.com/docs/installation)：平台构建工具和系统依赖
- [GPUI Kit 入门指南](https://gpui-kit.com/docs/getting-started)：独立项目与启动流程
- [GPUI Kit 0.8.0 API 文档](https://docs.rs/gpui-kit/0.8.0/gpui_kit/)：本文所用版本的公开 API
- [GPUI Kit 0.8.0 依赖信息](https://crates.io/crates/gpui-kit/0.8.0)：发布版本与功能配置

在线指南可能随新版本更新；若它与本文存在差异，先确认正在阅读的文档版本，不要单独升级或覆盖 Kit 内部依赖的某个 GPUI 包。
