# 资料基线与可信边界

本文说明这组中文二次开发文档读了哪些现有资料，以及哪些地方需要继续
按目标功能追源码。

## 已确认的仓库事实

- 当前工作区根目录存在 `.codegraph/`。主界面、主编辑器和 AI 分析按项目规则先使用
  CodeGraph 定位，再用本地源码核对关键实现。
- 根 `Cargo.toml` 使用 Cargo workspace，`default-members = ["crates/zed"]`。
- 根 `Cargo.toml` 当前列出 236 个 `crates/` workspace member 和 4 个
  `extensions/` workspace member。
- `rust-toolchain.toml` 固定 Rust 工具链为 `1.95.0`，profile 为
  `minimal`，组件包含 `rustfmt`、`clippy`、`rust-analyzer` 和 `rust-src`。
- `docs` 使用 mdBook，源码目录是 `docs/src`，自定义预处理器是
  `crates/docs_preprocessor`。
- 你要求输出到 `docs/zh`，所以本组文档作为独立中文二开资料存在，
  当前没有加入 `docs/src/SUMMARY.md`。

## 已读的主要入口

仓库与规则：

- `README.md`
- `CONTRIBUTING.md`
- `.rules`
- `Cargo.toml`
- `.cargo/config.toml`
- `rust-toolchain.toml`
- `docs/.rules`
- `docs/AGENTS.md`
- `docs/README.md`
- `docs/book.toml`
- `docs/src/SUMMARY.md`

开发与用户文档：

- `docs/src/development.md`
- `docs/src/development/macos.md`
- `docs/src/development/linux.md`
- `docs/src/development/windows.md`
- `docs/src/development/glossary.md`
- `docs/src/running-testing.md`
- `docs/src/configuring-zed.md`
- `docs/src/key-bindings.md`
- `docs/src/extensions/developing-extensions.md`
- `docs/src/extensions/capabilities.md`
- `docs/src/extensions/languages.md`
- `docs/src/collaboration/overview.md`
- `docs/src/ai/overview.md`

主程序与核心 crate：

- `crates/zed/Cargo.toml`
- `crates/zed/src/main.rs`
- `crates/zed/src/zed.rs`
- `crates/workspace/src/workspace.rs`
- `crates/workspace/src/multi_workspace.rs`
- `crates/workspace/src/dock.rs`
- `crates/workspace/src/pane_group.rs`
- `crates/workspace/src/pane.rs`
- `crates/workspace/src/status_bar.rs`
- `crates/workspace/src/toolbar.rs`
- `crates/workspace/src/item.rs`
- `crates/workspace/src/workspace_settings.rs`
- `crates/title_bar/src/title_bar.rs`
- `crates/title_bar/src/title_bar_settings.rs`
- `crates/editor/src/editor.rs`
- `crates/editor/src/element.rs`
- `crates/editor/src/display_map.rs`
- `crates/editor/src/input.rs`
- `crates/editor/src/items.rs`
- `crates/multi_buffer/src/multi_buffer.rs`
- `crates/project/src/project.rs`
- `crates/language/src/language.rs`
- `crates/lsp/src/lsp.rs`
- `crates/gpui/README.md`
- `crates/settings/src/settings.rs`
- `crates/ui/src/ui.rs`
- `crates/extension/src/extension.rs`
- `crates/extension/src/extension_manifest.rs`
- `crates/extension_host/src/extension_host.rs`

扩展、协作与 AI：

- `extensions/README.md`
- `extensions/glsl/extension.toml`
- `extensions/glsl/src/glsl.rs`
- `extensions/proto/extension.toml`
- `extensions/proto/src/proto.rs`
- `crates/collab/README.md`
- `crates/collab/src/main.rs`
- `crates/collab/src/lib.rs`
- `crates/agent/src/agent.rs`
- `crates/agent/src/native_agent_server.rs`
- `crates/agent/src/thread.rs`
- `crates/agent/src/tools.rs`
- `crates/agent/src/tool_permissions.rs`
- `crates/agent/src/sandboxing.rs`
- `crates/agent/src/templates.rs`
- `crates/agent/src/templates/system_prompt.hbs`
- `crates/agent_settings/src/agent_settings.rs`
- `crates/agent_settings/src/agent_profile.rs`
- `crates/agent_ui/src/agent_ui.rs`
- `crates/agent_ui/src/agent_panel.rs`
- `crates/agent_ui/src/conversation_view.rs`
- `crates/agent_ui/src/conversation_view/thread_view.rs`
- `crates/agent_ui/src/conversation_view/message_queue.rs`
- `crates/agent_ui/src/message_editor.rs`
- `crates/agent_ui/src/inline_assistant.rs`
- `crates/agent_ui/src/buffer_codegen.rs`
- `crates/agent_ui/src/terminal_codegen.rs`
- `crates/acp_thread/src/connection.rs`
- `crates/acp_thread/src/acp_thread.rs`
- `crates/agent_servers/src/agent_servers.rs`
- `crates/language_model/src/language_model.rs`
- `crates/language_model/src/registry.rs`
- `crates/language_model/src/request.rs`
- `crates/language_model_core/src/language_model_core.rs`
- `crates/language_model_core/src/request.rs`
- `crates/language_models/src/language_models.rs`
- `crates/prompt_store/src/prompts.rs`
- `crates/prompt_store/src/prompt_store.rs`
- `crates/zed/src/zed/edit_prediction_registry.rs`
- `crates/edit_prediction/src/edit_prediction.rs`
- `crates/edit_prediction_ui/src/edit_prediction_ui.rs`

脚本：

- `script/clippy`
- `script/clippy.ps1`
- `script/new-crate`
- `crates/cli/README.md`
- `crates/icons/README.md`

## 可信边界

这些文档只把上述文件中确认过的内容写成事实。对于没有展开阅读的 crate，
只按 crate 名、公开导出、启动注册点或现有文档描述其职责，不推断内部实现。

二次开发时，如果目标是某个具体功能，例如 Git 面板、终端、远程开发或某个
语言 server，需要继续阅读对应 crate 的入口、测试和调用方。本文档给出追踪
方向，但不替代针对目标改动的代码阅读。

## 复核方法

常用本地命令：

```sh
rg "关键类型或函数名" crates docs extensions
rg --files crates | sort
cargo metadata --no-deps --format-version 1
```

仓库根目录存在 `.codegraph/` 时，应优先使用项目规则中要求的 CodeGraph
工具定位代码，再补充 `rg`。
