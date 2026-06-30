# 二次开发工作流

本文把前面文档整理成可执行的开发流程。先定位改动层级，再读源码、修改、验证。

## 先判断改动层级

| 目标                            | 优先入口                                                             |
| ------------------------------- | -------------------------------------------------------------------- |
| 改命令行参数或启动行为          | `crates/zed/src/main.rs`、`crates/cli`                               |
| 改窗口、pane、panel、status bar | `crates/zed/src/zed.rs`、`crates/workspace`                          |
| 改编辑行为                      | `crates/editor`                                                      |
| 改文件树或项目状态              | `crates/project`、`crates/worktree`、`crates/project_panel`          |
| 改语言支持                      | 扩展优先；内置语言看 `crates/languages`、`crates/language`           |
| 改 LSP                          | `crates/project/src/lsp_store*`、`crates/lsp`、对应 language adapter |
| 改设置                          | `crates/settings*`、`assets/settings`、`settings_ui`                 |
| 改快捷键/action                 | action 定义处、keymap assets、`crates/docs_preprocessor`             |
| 改扩展能力                      | `crates/extension`、`crates/extension_host`、`extension_api`         |
| 改协作                          | `crates/collab`、`crates/collab_ui`、`crates/rpc`、`crates/client`   |
| 改 AI                           | `crates/agent*`、`crates/language_model*`、provider crate            |

## 修改主程序功能

1. 在 `crates/zed/src/main.rs` 找初始化点。
2. 确认需要的全局状态是否已初始化。
3. 如果是 app-level action，看 `zed::init(cx)`。
4. 如果是 workspace-level action，看 `register_actions(...)`。
5. 如果涉及窗口创建，看 `build_window_options(...)` 和 `initialize_workspace(...)`。
6. 改完运行目标 crate 测试，必要时启动 `cargo run` 手工验证。

注意：不要随意调整初始化顺序。很多模块依赖 settings、client、language registry、
extension host 或 AppState 已存在。

## 新增或修改 UI

1. 找现有同类 UI。面板看 project、outline、terminal、git、debug、agent panel。
2. 优先复用 `ui` crate components。
3. 状态放在 `Entity<T>`，通过 `Render` 输出 element。
4. 用户操作用 action 或 element event handler。
5. 异步工作用 `Task`，UI 更新回到 GPUI context。
6. 涉及状态变化后调用 `cx.notify()`。

验证：

```sh
cargo test -p <target-crate>
cargo run
```

macOS UI 大改可跑视觉测试。

## 新增设置

1. 找设置类型定义，通常在 `settings_content` 或对应 feature crate。
2. 加默认值到 `assets/settings/default.json`。
3. 如果需要初始用户文件内容，检查 `assets/settings/initial_*.json`。
4. 如果需要 UI 编辑，检查 `settings_ui`。
5. 如果要支持项目设置，确认该设置允许 local/project scope。
6. 更新文档和 schema 相关输出。

验证：

```sh
cargo test -p settings
cargo test -p settings_ui
```

实际 crate 名和测试范围按改动位置调整。

## 新增 action 或快捷键

1. 定义 action，选择合适 namespace。
2. 在正确层级注册 handler。
3. 加默认 keymap 时检查平台差异。
4. 用 key context 限制作用范围。
5. 如果文档引用 keybinding/action，重新生成 action metadata。

验证：

```sh
script/generate-action-metadata
cargo test -p settings
cargo run
```

手工检查 command palette、keymap editor 和冲突行为。

## 新增语言能力

优先判断是否可通过扩展完成。扩展路径：

1. 创建 extension repository 或使用本仓库 `extensions/test-extension` 参考结构。
2. 写 `extension.toml`。
3. 加 `languages/<name>/config.toml`。
4. 加 Tree-sitter queries。
5. 如果有 LSP，实现 `language_server_command`。
6. 用 Install Dev Extension 本地安装。

需要改内置语言时：

1. 查 `crates/languages/src/<language>`。
2. 查 `crates/language` 的 config、query 和 registry 行为。
3. 查 `project::lsp_store` 和 adapter。

不要把语言 server 二进制直接塞进扩展。现有发布规则要求扩展下载或查找用户环境中
的 language server。

## 修改协作或云端能力

1. 明确改客户端、RPC 还是 collab server。
2. 客户端 UI 看 `collab_ui`、`channel`、`call`。
3. 协议看 `rpc` 和 `proto`。
4. server 看 `crates/collab/src/main.rs`、`api`、`rpc`、`db`。
5. 本地验证需要 Postgres、`script/bootstrap` 和 `foreman start`。

不要默认当前仓库包含完整 `zed.dev` 认证站点。`collab` README 明确说明认证站点是
另一个 repo。

## 修改 AI 或 Agent

1. UI 看 `agent_ui`。
2. 核心 thread/tools/sandbox 看 `agent`。
3. 模型 registry 和 provider 看 `language_model*` 与 provider crate。
4. 外部 agent 看 `agent_servers`、ACP 相关 crate。
5. MCP/context server 看 `context_server`、`project::context_server_store`。
6. 改 tool 权限时同步检查 sandboxing。

Agent Panel 可能因为 `disable_ai` 或测试环境不存在。调用前要处理缺失状态。

## PR 前检查清单

- 改动范围是否只覆盖目标功能。
- 是否遵守 `.rules` 中 Rust 风格：不随意 `unwrap()`、不吞掉错误、不建 `mod.rs`。
- 是否处理 async 错误并能向 UI 层反馈。
- 是否有测试或手工验证记录。
- UI 改动是否检查 light/dark、窄 pane、键盘和鼠标路径。
- 文档或 action 改动是否更新 metadata/文档。
- 新 crate 是否加入 workspace，并使用合适 license。
- 新扩展是否声明最小能力和 license。

## 依据文件

- `.rules`
- `CONTRIBUTING.md`
- `crates/zed/src/main.rs`
- `crates/zed/src/zed.rs`
- `crates/workspace/src/workspace.rs`
- `crates/editor/src/editor.rs`
- `crates/project/src/project.rs`
- `crates/language/src/language.rs`
- `crates/extension/src/extension_manifest.rs`
- `docs/src/extensions/developing-extensions.md`
- `docs/src/development/glossary.md`
