# 启动流程与应用状态

主程序入口是 `crates/zed/src/main.rs`。二次开发如果涉及全局服务、窗口创建、
命令行行为、初始化顺序或跨模块状态，应先读这个文件。

## 二进制入口

`crates/zed/Cargo.toml` 定义：

```toml
[[bin]]
name = "zed"
path = "src/main.rs"
```

`main.rs` 中还有一个编译期断言，要求二进制名和 `paths::APP_NAME_LOWERCASE`
保持一致。fork 如果要改应用名，需要同时检查 `crates/paths/src/paths.rs`。

## 启动前置处理

`main()` 开始后先处理这些早期分支：

- Linux sandbox helper 模式：`sandbox::run_sandbox_launcher_if_invoked()`。
- Unix root 运行保护：`util::prevent_root_execution()`。
- 命令行参数解析：`Args::parse()`。
- 非 Windows 的 `--askpass` 模式。
- `--crash-handler` minidump crash server 模式。
- Windows ETW trace record 模式。
- `--printenv` 和 `--dump-all-actions`。
- `--user-data-dir` 自定义数据目录。
- Windows askpass 程序路径设置。

这些分支会在进入主 UI 初始化前返回。改 CLI 行为时不要只看 `cli` crate，
也要看这里的 `Args`。

## 路径、日志和版本

启动会调用 `init_paths()` 创建以下目录：

- config
- extensions
- languages
- debug adapters
- database
- logs
- temp
- hang traces

之后初始化 `zlog` 和 `ztracing`。如果 stdout 是 TTY，日志输出到 stdout；
否则写到 `paths::log_file()`。

版本信息来自：

- `CARGO_PKG_VERSION`
- 可选 `ZED_BUILD_ID`
- 可选 `ZED_COMMIT_SHA`
- `release_channel::RELEASE_CHANNEL`

`--system-specs` 会在 UI 初始化前输出系统信息并退出。

## Application 与全局资源

`build_application()` 根据 `gpui_platform::current_platform(false)` 创建 GPUI
`Application`。只有设置 `ZED_EXPERIMENTAL_A11Y=1` 时使用
`Application::with_platform(platform)`，否则使用
`Application::new_inaccessible(platform)`。

进入 `app.run(move |cx| { ... })` 后，启动链路设置和初始化大量全局资源：

- `db::AppDatabase`
- trusted worktrees
- menu 和 `zed_actions`
- release channel
- `gpui_tokio`
- settings 和 zlog settings
- HTTP client
- 全局 `Fs`
- Git hosting provider registry
- `OpenListener`
- extension host proxy
- `Client::production`
- `LanguageRegistry`
- `NodeRuntime`
- `UserStore`
- `WorkspaceStore`

这些资源随后被组合进 `workspace::AppState`：

```rust
AppState {
    languages,
    client,
    user_store,
    fs,
    build_window_options,
    workspace_store,
    node_runtime,
    session,
}
```

`AppState` 是很多 UI 和功能模块拿到共享服务的入口。

## 模块初始化顺序

`main.rs` 中的初始化顺序很长。二开时需要注意，很多 crate 的 `init` 假定
前面的全局状态已经存在。

从当前入口可见，核心顺序包括：

1. `settings::init(cx)`。
2. extension 基础设施：`extension::init(cx)`、`extension_host::init(...)`。
3. 语言和扩展语言：`languages::init(...)`、`language_extension::init(...)`。
4. `zed::init(cx)` 注册应用 action。
5. `project::Project::init(&client, cx)`。
6. client、feature flags、auto update、DAP、theme。
7. command palette、AI/model、agent、REPL。
8. `editor::init(cx)`。
9. `workspace::init(app_state.clone(), cx)`。
10. 导航、project panel、outline panel、tasks、search、vim、terminal。
11. collaboration、Git、preview、onboarding、settings UI、extensions UI。
12. `initialize_workspace(app_state.clone(), cx)`。

如果新增全局功能，优先放在与依赖最接近的位置，而不是文件末尾。

## OpenListener 与打开请求

`OpenListener` 接收：

- CLI connection
- `zed://` URL
- 文件路径或 diff 路径
- extension deep link
- agent panel deep link
- settings deep link
- git clone 和 commit view 请求

初始参数会转换成 `RawOpenRequest`。启动时如果没有明确打开路径，会走
`restore_or_create_workspace(...)` 恢复上次工作区或创建空窗口。

二次开发如果新增 deep link，应看：

- `crates/zed/src/zed/open_listener.rs`
- `OpenRequestKind`
- `handle_open_request(...)`

## 窗口与 workspace 初始化

`crates/zed/src/zed.rs` 提供：

- `build_window_options(...)`
- `initialize_workspace(...)`
- `initialize_panels(...)`
- `register_actions(...)`

`build_window_options` 设置 titlebar、窗口装饰、最小尺寸、app id 和平台图标。

`initialize_workspace` 会：

- 绑定最后一个窗口关闭时的退出行为。
- 初始化 cursor hide mode。
- 观察新 `MultiWorkspace`，注册关闭处理和 sidebar。
- 观察新 `Workspace`，初始化 pane、文件 watcher、GPU 信息、status bar 和 panels。
- 注册 workspace 级 actions。

`initialize_panels` 当前加载：

- Project Panel
- Outline Panel
- Terminal Panel
- Git Panel
- Collaboration Panel
- Debug Panel
- Agent Panel

Agent Panel 会受 `DisableAiSettings` 和测试环境影响。不要假定所有 panel 在所有
配置下都存在。

## 应用级 action

`zed::init(cx)` 注册应用级 action，例如：

- 打开设置文件、keymap 文件、tasks 文件。
- 打开默认 settings、默认 keymap、默认 semantic token rules。
- 打开 licenses。
- About 窗口。
- quit、hide、fullscreen、test panic/crash 等。

窗口或 workspace 相关 action 则在 `register_actions(...)` 中注册。

## 依据文件

- `crates/zed/Cargo.toml`
- `crates/zed/src/main.rs`
- `crates/zed/src/zed.rs`
- `docs/src/development/glossary.md`
