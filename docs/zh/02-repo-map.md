# 仓库结构与模块地图

Zed 是一个 Rust workspace。当前根 `Cargo.toml` 中有 236 个 `crates/`
成员和 4 个内置扩展成员。不要把它当作单一二进制项目修改；二次开发通常
需要先找到功能所属 crate，再看该 crate 如何在 `crates/zed` 中注册。

## 顶层目录

| 路径          | 用途                                             |
| ------------- | ------------------------------------------------ |
| `crates/`     | 主体 Rust workspace crate。                      |
| `extensions/` | 仓库内维护的扩展示例和官方维护扩展。             |
| `docs/`       | mdBook 用户文档和本文档所在目录。                |
| `assets/`     | 默认设置、keymap、图标、主题等资源。             |
| `script/`     | 构建、检查、发布、协作服务、文档等脚本。         |
| `tooling/`    | workspace 工具 crate，例如 `xtask`、性能、合规。 |
| `.cargo/`     | Cargo 别名、目标平台 rustflags 和环境变量。      |

## 主程序层

| crate                    | 主要职责                                                              |
| ------------------------ | --------------------------------------------------------------------- |
| `zed`                    | 桌面应用二进制入口，集中初始化全局服务和 UI 模块。                    |
| `cli`                    | `zed` 命令行入口。Linux 打包文档说明发布时需要 CLI 和 editor 二进制。 |
| `paths`                  | 应用名、配置目录、数据目录等路径约定。                                |
| `release_channel`        | Stable、Preview、Nightly、Dev 等渠道信息。                            |
| `session`                | 应用 session 信息。                                                   |
| `zlog`、`ztracing`       | 日志与 tracing 初始化。                                               |
| `crashes`、`reliability` | crash handler、hang detection 和可靠性相关逻辑。                      |

## UI 与窗口层

| crate                             | 主要职责                                                      |
| --------------------------------- | ------------------------------------------------------------- |
| `gpui`                            | Zed 使用的 Rust UI 框架。                                     |
| `gpui_platform`                   | 平台窗口、文本和渲染后端。                                    |
| `ui`                              | Zed UI primitives 和 components。                             |
| `workspace`                       | 窗口根视图、pane、dock、panel、status bar、workspace 持久化。 |
| `component`、`component_preview`  | UI component 支撑和预览。                                     |
| `title_bar`、`platform_title_bar` | 标题栏相关 UI。                                               |
| `panel`、`sidebar`、`picker`      | 通用 UI 构件。                                                |

## 编辑与项目层

| crate                | 主要职责                                                 |
| -------------------- | -------------------------------------------------------- |
| `editor`             | 核心 `Editor` 类型。代码编辑器和输入框都大量使用它。     |
| `multi_buffer`       | 多 buffer 编辑抽象。搜索、引用跳转等会打开 multibuffer。 |
| `project`            | Project、Worktree 和各类 store 的聚合。                  |
| `worktree`           | 本地或远程文件树、文件监听、路径条目。                   |
| `fs`                 | 文件系统抽象。                                           |
| `language`           | 语言配置、Tree-sitter、diagnostics 和 registry。         |
| `lsp`                | Language Server Protocol JSON-RPC 进程通信。             |
| `languages`          | 内置语言注册和 grammar 加载。                            |
| `language_extension` | 扩展提供的语言和 LSP 接入。                              |

## 功能模块

| 功能       | 相关 crate                                                |
| ---------- | --------------------------------------------------------- |
| 搜索与导航 | `search`、`file_finder`、`project_symbols`、`outline`     |
| Git        | `git`、`git_ui`、`git_hosting_providers`                  |
| 终端与任务 | `terminal`、`terminal_view`、`task`、`tasks_ui`           |
| Debugger   | `dap`、`dap_adapters`、`debugger_ui`、`debugger_tools`    |
| 设置       | `settings`、`settings_content`、`settings_ui`             |
| 主题       | `theme`、`theme_settings`、`theme_extension`              |
| 扩展       | `extension`、`extension_api`、`extension_host`            |
| 协作       | `collab`、`collab_ui`、`channel`、`call`、`rpc`、`client` |
| AI         | `agent`、`agent_ui`、`language_model*`、各 provider crate |

## 代码入口规律

多数功能 crate 会提供一个 `init(cx)` 或类似函数，然后在
`crates/zed/src/main.rs` 中集中调用。例如主启动链路中会调用
`editor::init(cx)`、`workspace::init(...)`、`project_panel::init(cx)`、
`terminal_view::init(cx)`、`git_ui::init(cx)`、`agent_ui::init(...)` 等。

要修改一个功能，常见追踪路径是：

1. 在 `crates/zed/src/main.rs` 找到该 crate 的初始化点。
2. 打开该 crate 的公开入口文件，例如 `crates/editor/src/editor.rs`。
3. 查 `actions!`、`register_action`、`Render`、`EventEmitter` 和 store 类型。
4. 查测试文件和同 crate 中的 `test-support` feature。

## 内置扩展

`extensions/README.md` 说明，仓库内 `extensions/` 包含 Zed 团队维护的一些
扩展。当前 workspace member 是：

- `extensions/glsl`
- `extensions/html`
- `extensions/proto`
- `extensions/test-extension`

其他官方维护扩展在独立组织或扩展仓库中。

## 依据文件

- `Cargo.toml`
- `README.md`
- `CONTRIBUTING.md`
- `crates/zed/src/main.rs`
- `crates/zed/Cargo.toml`
- `docs/src/development/glossary.md`
- `extensions/README.md`
