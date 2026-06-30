# 扩展系统

Zed 扩展是 Git 仓库，核心是 `extension.toml` manifest。扩展可以提供语言、
主题、调试器、snippets、MCP servers 等能力。带过程逻辑的扩展用 Rust 编写并
编译成 WebAssembly。

## 扩展能力

现有扩展文档列出的能力包括：

- Languages
- Debuggers
- Themes
- Icon Themes
- Snippets
- MCP Servers

`crates/extension/src/extension_manifest.rs` 当前 manifest 结构还包含：

- `language_servers`
- `context_servers`
- `slash_commands`
- `debug_adapters`
- `debug_locators`
- `language_model_providers`
- `capabilities`

当前 extension schema version 支持范围由 `schema_version_range()` 给出，
上限是 `CURRENT_SCHEMA_VERSION = SchemaVersion(1)`。

## Manifest 基本结构

基础字段：

```toml
id = "my-extension"
name = "My extension"
version = "0.0.1"
schema_version = 1
authors = ["Your Name <you@example.com>"]
description = "Example extension"
repository = "https://github.com/your-name/my-zed-extension"
```

语言扩展通常还会包含：

```toml
[grammars.my-language]
repository = "https://github.com/example/tree-sitter-my-language"
rev = "..."

[language_servers.my-language-server]
name = "My Language LSP"
languages = ["My Language"]
```

`GrammarManifestEntry` 的字段名是 `rev`，并用 serde alias 兼容 `commit`。
`LanguageServerManifestEntry` 中单数 `language` 已标记为 deprecated，新增扩展
应使用复数 `languages`。

## Rust/WASM 扩展入口

扩展代码实现 `zed_extension_api::Extension`，并用宏注册：

```rust
zed::register_extension!(MyExtension);
```

`extensions/proto/src/proto.rs` 示例实现了：

- `language_server_command`
- `language_server_workspace_configuration`
- `language_server_initialization_options`

`extensions/glsl/src/glsl.rs` 示例展示了：

- 优先用 `worktree.which("glsl_analyzer")` 查找系统二进制。
- 找不到时通过 GitHub release 下载。
- 设置 language server installation status。
- 缓存下载后的 binary path。
- 从 `LspSettings::for_worktree(...)` 读取 LSP 配置。

## Extension crate 与 host

`crates/extension/src/extension.rs` 提供基础 trait 和 proxy：

- `Extension`
- `WorktreeDelegate`
- `ProjectDelegate`
- `KeyValueStoreDelegate`
- `ExtensionHostProxy`
- manifest、capability、events 类型

`extension::init(cx)` 初始化 extension events，并设置默认全局
`ExtensionHostProxy`。

`crates/extension_host/src/extension_host.rs` 提供 `ExtensionStore`。从当前入口可见，
它维护：

- proxy
- builder
- extension index
- filesystem 和 HTTP client
- installed/staging/index 路径
- outstanding operations
- Wasm host 和 Wasm extensions
- remote clients

`ExtensionIndex` 跟踪 extensions、themes、icon themes 和 languages。

## 能力限制

扩展能力由 capability system 管理。用户可通过
`granted_extension_capabilities` 限制扩展能力。

当前文档列出的能力包括：

- `process:exec`
- `download_file`
- `npm:install`

`ExtensionManifest::allow_exec(...)` 会检查 manifest 中是否声明了对应
`process:exec` capability。扩展如果要执行进程，应在 manifest 中声明最小权限，
而不是依赖全局放开。

## 仓库内扩展

`extensions/README.md` 说明，本仓库内扩展主要是 Zed 团队为便于维护而放在
主仓库中的扩展。它们使用与其他 Zed 扩展相同的 `zed_extension_api`。

当前内置扩展示例：

- `glsl`：GLSL grammar 和 `glsl_analyzer` LSP。
- `html`：HTML language support。
- `proto`：Protocol Buffers grammar 和多个 LSP。
- `test-extension`：测试扩展。

## 二开建议

- 能通过扩展实现的语言、主题、debugger、MCP 能力，优先用扩展实现。
- 只有需要改核心编辑器行为、通用 API、扩展 host 或内置分发策略时，才改
  主仓库核心 crate。
- 扩展需要 Rust 时，注意 WASM 环境限制。现有文档明确说明 `std::env::var`
  不会像本机进程那样工作，应使用 extension API 和 `Worktree` 方法。
- 发布扩展前要满足 license 和 manifest 规则。

## 依据文件

- `docs/src/extensions/developing-extensions.md`
- `docs/src/extensions/capabilities.md`
- `docs/src/extensions/languages.md`
- `extensions/README.md`
- `extensions/glsl/extension.toml`
- `extensions/glsl/src/glsl.rs`
- `extensions/proto/extension.toml`
- `extensions/proto/src/proto.rs`
- `crates/extension/src/extension.rs`
- `crates/extension/src/extension_manifest.rs`
- `crates/extension_host/src/extension_host.rs`
