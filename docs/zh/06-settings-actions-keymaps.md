# 设置、Action 与快捷键

Zed 的用户配置、默认配置、action 和快捷键系统紧密关联。二次开发涉及命令、
菜单、快捷键、设置项或文档 keybinding 时，应一起检查。

## SettingsStore

`crates/settings/src/settings.rs` 中 `settings::init(cx)` 创建
`SettingsStore::new(cx, &default_settings())` 并设为全局状态。

默认设置和 keymap 来自 Rust embedded assets：

- `assets/settings/default.json`
- `assets/settings/default_semantic_token_rules.json`
- `assets/keymaps/default-macos.json`
- `assets/keymaps/default-windows.json`
- `assets/keymaps/default-linux.json`
- `assets/keymaps/vim.json`
- `assets/keymaps/specific-overrides*.json`

初始用户文件内容来自：

- `assets/settings/initial_user_settings.json`
- `assets/settings/initial_server_settings.json`
- `assets/settings/initial_local_settings.json`
- `assets/keymaps/initial.json`
- `assets/settings/initial_tasks.json`
- `assets/settings/initial_debug_tasks.json`
- `assets/settings/initial_local_debug_tasks.json`

二开新增设置时，需要同时考虑：

- settings content/schema 类型在哪个 crate。
- 默认值是否进 `assets/settings/default.json`。
- Settings UI 是否能编辑。
- 是否允许 project-level override。
- 文档和 JSON schema 是否需要更新。

## 设置分层

现有文档描述的设置优先级：

1. 默认设置。
2. 用户设置。
3. 项目设置。

对象设置会按属性合并，而不是整体替换。项目设置位于 `.zed/settings.json`。
并非所有设置都能在项目级生效，影响全局编辑器行为的设置通常只在用户设置中生效。

设置文件位置：

| 平台    | settings                                                              |
| ------- | --------------------------------------------------------------------- |
| macOS   | `~/.config/zed/settings.json`                                         |
| Linux   | `~/.config/zed/settings.json` 或 `$XDG_CONFIG_HOME/zed/settings.json` |
| Windows | `%APPDATA%\Zed\settings.json`                                         |

## Action 定义

Action 常见定义方式：

```rust
actions!(zed, [OpenDefaultSettings, OpenProjectSettingsFile]);
```

或：

```rust
#[derive(Action)]
#[action(namespace = workspace)]
pub struct Open {
    pub create_new_window: Option<bool>,
}
```

无参数 action 可直接在 keymap 中写字符串。有参数 action 通常写成数组：

```json
["workspace::ActivatePane", 0]
```

复杂 action 可写成 action 名和对象参数。

## Action 注册位置

不同 action 应注册在不同层级：

- 应用级：`cx.on_action(...)`，例如 `zed::init(cx)` 中的设置、日志、About。
- workspace 级：`workspace.register_action(...)`，例如打开文件、调整 pane、
  打开 URL。
- panel 或 view 级：在对应 entity 或 element 上注册。
- editor 级：通常在 `editor` crate 的初始化或元素处理里注册。

快捷键冲突时，Zed 会按 key context tree 决定优先级。更低层 context 胜出；
同层后定义的 binding 胜出。

## Keymap

用户 keymap 文件位置：

| 平台        | keymap                              |
| ----------- | ----------------------------------- |
| macOS/Linux | `~/.config/zed/keymap.json`         |
| Windows     | `~\AppData\Roaming\Zed/keymap.json` |

keymap 文件是 JSON array。每项可以有 `context` 和 `bindings`：

```json
[
  {
    "context": "ProjectPanel && not_editing",
    "bindings": {
      "o": "project_panel::Open"
    }
  }
]
```

上下文表达式支持：

- `&&`
- `||`
- `!`
- `(...)`
- `X > Y`

调试当前 key context 可用 `dev::OpenKeyContextView`。

## 生成 action 元数据

`crates/zed/src/main.rs` 有 `--dump-all-actions` 分支，会调用
`dump_all_gpui_actions()` 输出 action 元数据。`docs/README.md` 说明文档站
需要先运行：

```sh
script/generate-action-metadata
```

该脚本把 action manifest 写到 `crates/docs_preprocessor/actions.json`，供
文档预处理器校验 `{#kb ...}` 和 `{#action ...}`。

## 新增 action 的检查清单

- action 名称和 namespace 是否与现有风格一致。
- 是否需要参数 schema。
- 是否需要出现在 command palette。
- 是否需要默认快捷键。
- 是否需要 key context。
- 是否需要文档引用。
- 是否需要测试 keymap 解析或 action 行为。

## 依据文件

- `docs/src/configuring-zed.md`
- `docs/src/key-bindings.md`
- `docs/README.md`
- `crates/settings/src/settings.rs`
- `crates/zed/src/main.rs`
- `crates/zed/src/zed.rs`
- `crates/workspace/src/workspace.rs`
