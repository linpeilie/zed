# GPUI 与界面开发

Zed 的界面建立在 GPUI 上。改 UI 前，需要先理解 GPUI 的 state、view、
element、action 和异步任务模型。

## GPUI 的定位

`crates/gpui/README.md` 描述 GPUI 是一个 hybrid immediate and retained mode、
GPU accelerated 的 Rust UI framework。Zed 是 GPUI 的主要使用者。

一个 GPUI app 从 `Application` 开始。独立 GPUI 示例会在
`Application::run()` 回调里通过 `App::open_window()` 创建窗口。Zed 在
`crates/zed/src/main.rs` 中创建 `Application`，并在 `app.run(...)` 中注册
所有全局服务和 UI 模块。

## Context 类型

现有 glossary 和 `.rules` 对上下文的解释一致：

| 类型                 | 用途                                                       |
| -------------------- | ---------------------------------------------------------- |
| `App`                | 根上下文，持有全局状态和 entity map，只在 UI 线程使用。    |
| `Context<T>`         | 更新 `Entity<T>` 时的上下文，可访问当前 entity 和 `App`。  |
| `AsyncApp`           | 异步上下文中持有的 app 访问入口。                          |
| `AsyncWindowContext` | 窗口相关异步任务中使用。                                   |
| `Window`             | 窗口状态、focus、action dispatch、输入状态和直接绘制入口。 |

代码中函数参数通常按 `window, cx` 顺序出现。没有窗口时只传 `cx`。

## Entity 与生命周期

`Entity<T>` 是 GPUI 管理的强引用句柄。常用操作：

- `read(cx)` 读取。
- `update(cx, |this, cx| ...)` 修改。
- `update_in(cx, |this, window, cx| ...)` 在窗口上下文中修改。
- `downgrade()` 得到 `WeakEntity<T>`。

`WeakEntity<T>` 的更新返回 `anyhow::Result`，因为 entity 可能已释放。

规则中明确提醒：在 entity update 的闭包里要使用闭包参数中的 `cx`，不要继续
使用外层 `cx`，避免多重 borrow 问题。也要避免在 entity 正被更新时再次更新
同一个 entity，否则可能 panic。

## Render 与 Element

实现 `Render` 的 state 可以渲染成 element tree。`Entity<T>` 中的 `T`
如果实现 `Render`，通常可以被视为 view。

常见 element 构建方式来自 `gpui` 和 `ui::prelude::*`，例如 `div()`、
`h_flex()`、`.child(...)`、`.when(...)`。样式 API 接近 Tailwind。

如果一个类型只是构造后立即转为 element，可以实现 `RenderOnce`，并使用
`#[derive(IntoElement)]`。

## Zed UI crate

`crates/ui/src/ui.rs` 说明 `ui` crate 提供 Zed UI primitives 和 components，
并导出：

- `components::*`
- `prelude::*`
- `styles::*`
- animation extension traits

二次开发 UI 时，优先复用 `ui` crate 组件和现有面板中的模式，不要直接在
业务 crate 中重新发明按钮、菜单、输入框或状态样式。

## Workspace、Pane、Panel、Dock

`docs/src/development/glossary.md` 对 UI 结构给出术语：

- `Workspace` 是窗口根。
- `Center` 是窗口中间区域，分成多个 `Pane`。
- `Pane` 放 editor、multibuffer、terminal 等 item。
- `Panel` 是实现 `Panel` trait 的 entity，可放在 dock 中。
- `Dock` 可位于 left、right、bottom，包含一个或多个 panel。
- `Editor` 不实现 `Panel`。

当前 `initialize_panels` 加载 project、outline、terminal、git、collab、
debug 和 agent panel。

## Action 与事件

Action 可以来自键盘、command palette 或代码分发。定义方式包括：

- `actions!(namespace, [ActionName])`
- `#[derive(Action)]`

处理方式包括：

- `cx.on_action(...)`
- element 上的 `.on_action(...)`
- workspace 上的 `register_action(...)`
- `window.dispatch_action(...)`

Entity 事件通过 `EventEmitter<EventType>` 声明，并使用 `cx.emit(event)` 发出。
其他 entity 可用 `cx.subscribe(...)` 订阅。订阅返回 `Subscription`，通常存入
结构体字段，drop 时自动取消。

## 异步任务

GPUI 提供 foreground 和 background executor：

- `cx.spawn(...)` 在 UI 线程调度异步任务。
- `cx.background_spawn(...)` 在后台线程池调度任务。
- `Task<R>` 是已经开始或计划运行的 future。

如果 `Task` 被 drop，任务会取消。需要长期运行时应：

- await 它；
- 或 `detach()` / `detach_and_log_err(cx)`；
- 或存进结构体字段。

UI 更新应回到 `cx.update(...)` 或 `update_in(...)`。

## 改 UI 的入口建议

- 新面板：看 `workspace::Panel`、现有 `project_panel`、`outline_panel`、
  `terminal_view`。
- 新状态栏项：看 `initialize_workspace` 中 `workspace.status_bar().update(...)`
  的注册方式。
- 新 command/action：先定义 action，再决定注册在 app、workspace、editor
  还是某个 panel。
- 新通用组件：优先放在合适的现有 crate。只有确实通用时才考虑 `ui`。

## 依据文件

- `.rules`
- `docs/src/development/glossary.md`
- `crates/gpui/README.md`
- `crates/ui/src/ui.rs`
- `crates/zed/src/zed.rs`
- `crates/workspace/src/workspace.rs`
