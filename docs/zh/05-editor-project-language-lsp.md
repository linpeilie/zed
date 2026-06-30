# 编辑器、项目、语言与 LSP

Zed 的编辑体验主要由 `editor`、`project`、`language`、`lsp`、`worktree`
和 `multi_buffer` 等 crate 协同完成。

## Editor

`crates/editor/src/editor.rs` 的 crate-level 文档说明：

- `Editor` 是 Zed 中 editor-related state 和 UI 的核心类型。
- 它用于代码编辑器，也用于很多输入框。
- 它有 single line、multiline 和 fixed height 等使用形态。
- `element` 子模块负责渲染。
- `display_map` 负责把文本拆成逻辑块，维护坐标映射和显示变换元数据。
- Vim 模式在 `vim` crate 中包裹 `Editor` 并覆盖部分行为。

`Editor` 依赖大量 project 和 language 能力，例如 completion、code action、
diagnostics、inlay hints、semantic tokens、git hunk、debugger inline value、
edit prediction 等。

## Project

`crates/project/src/project.rs` 的 `Project` 注释说明它是语义感知的 entity，
关联一个或多个 `Worktree`。它负责：

- tasks
- LSP
- collab queries
- worktree state 同步
- 用 `ProjectEntryId` 和 `ProjectPath` 映射 worktree entry

当前 `Project` 结构体持有或关联这些 store：

- `WorktreeStore`
- `BufferStore`
- `LspStore`
- `GitStore`
- `TaskStore`
- `DapStore`
- `BreakpointStore`
- `ContextServerStore`
- `AgentServerStore`
- `ToolchainStore`
- `SnippetProvider`
- `ProjectEnvironment`

`ProjectClientState` 当前有三种模式：

- `Local`：单人本地项目。
- `Shared`：多人模式，但项目仍在本机。
- `Collab`：加入远端共享项目。

二开时不要把 `Project` 当作单纯文件树。它是文件、buffer、LSP、任务、协作、
debugger 和环境信息的聚合点。

## Worktree 与 ProjectPath

Glossary 定义：

- `Project` 是一个或多个 `Worktree`。
- `Worktree` 表示本地或远程文件。

`ProjectPath` 包含：

```rust
pub struct ProjectPath {
    pub worktree_id: WorktreeId,
    pub path: Arc<RelPath>,
}
```

因此项目内路径通常不是单独的绝对路径，而是 `worktree_id + relative path`。
涉及打开文件、重命名、搜索结果、diagnostics、LSP 位置转换时，需要保留这个
结构。

## Buffer

Glossary 把 `Buffer` 定义为内存中的文件表示，同时包含 syntax tree、git
status、diagnostics 等相关数据。`Editor` 通常围绕 `Buffer` 或 `MultiBuffer`
工作。

`ProjectItem` trait 提供从 `ProjectPath` 打开某类项目 item 的抽象。打开文件、
图片、diff 或其他 item 时，可先查目标类型是否实现了 `ProjectItem`。

## Language

`crates/language/src/language.rs` 的 crate-level 文档说明：

- `language` crate 提供 Zed 大量语言相关能力。
- 主要类型包括 `Language`、`Grammar`、`LanguageRegistry`。
- 它使用 Tree-sitter 为 editor 提供语法相关范围与 outline。
- 它暴露 `LanguageConfig`，描述括号、行注释等语言构造。
- 现实文件可能混合多种语言，例如 HTML，因此 Zed 不把一个文件简单绑定到
  单一语言。

该 crate 还导出 text diff、diagnostic、runnable、toolchain、modeline 等能力。

## LSP

`crates/lsp/src/lsp.rs` 负责 Language Server Protocol 的 JSON-RPC 通信。
从当前入口可见：

- `LanguageServerBinary` 表示可启动的语言服务器命令、参数和环境变量。
- `LanguageServerBinaryOptions` 控制是否允许 PATH 查找、下载二进制和预发布版本。
- `LanguageServer` 持有 server id、请求 id、outbound channel、notification
  handler、response handler、process handle、server capabilities 等。
- 默认 LSP 初始化/获取 server 等待时间常量是
  `DEFAULT_LSP_REQUEST_TIMEOUT_SECS = 120`。
- server shutdown timeout 是 5 秒。

`Project` 中的 `LspStore` 是 Zed 项目层管理语言服务器、buffer 注册、diagnostics、
semantic tokens、inlay hints 等功能的核心入口。

## 语言支持的两条路径

内置语言：

- 在 `crates/languages/src` 下维护。
- 由 `languages::init(...)` 注册。
- `LanguageRegistry` 持有并按需要加载。

扩展语言：

- 在扩展的 `languages/<language>/config.toml` 中声明语言元数据。
- 在 `extension.toml` 中声明 grammars 和 language servers。
- Rust/WASM 扩展代码实现 `language_server_command` 等方法。
- 由 `language_extension::init(...)` 接入 workspace 中的 `LspStore`。

## 常见二开路径

改编辑器交互：

1. 查 `crates/editor/src/editor.rs` 的 action 或输入处理模块。
2. 查是否需要 project、language 或 LSP 数据。
3. 修改后跑对应 editor 测试，必要时加 GPUI 测试。

改语言支持：

1. 如果能通过扩展完成，优先走扩展。
2. 内置语言改动看 `crates/languages/src/<language>`。
3. LSP 行为看 `project::lsp_store` 和对应 adapter。
4. Tree-sitter query 改动要检查 highlight、outline、indent、injection 等文件。

改项目/文件树行为：

1. 看 `project` 和 `worktree`。
2. 注意 `ProjectPath`、`WorktreeId`、远程/协作模式。
3. UI 表现通常在 `project_panel` 或 `workspace`。

## 依据文件

- `docs/src/development/glossary.md`
- `crates/editor/src/editor.rs`
- `crates/project/src/project.rs`
- `crates/language/src/language.rs`
- `crates/lsp/src/lsp.rs`
- `docs/src/extensions/languages.md`
- `extensions/README.md`
