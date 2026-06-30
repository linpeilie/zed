# 测试、调试与质量门禁

Zed 仓库规模大，二次开发不能只依赖手工启动。应根据改动范围选择最小但有效的
测试组合。

## 常用测试命令

全 workspace：

```sh
cargo test --workspace
```

单 crate：

```sh
cargo test -p editor
cargo test -p project
cargo test -p workspace
```

如果改动只在某个 crate，优先跑单 crate 测试，再根据影响范围扩大。

## Clippy

项目规则要求使用脚本：

```sh
script/clippy
```

Windows：

```powershell
script\clippy.ps1
```

脚本行为：

- 没有传 `-p` 或 `--package` 时默认加 `--workspace`。
- 使用 release profile。
- 检查 all targets 和 all features。
- `--deny warnings`。

本地 Unix 脚本还会在工具存在时继续跑 `cargo machete`、`typos` 和 `buf` 检查。

## GPUI 测试注意点

根 `.rules` 对 GPUI 测试有明确要求：

- 需要 timeout、delay 或驱动 `run_until_parked()` 时，优先使用 GPUI executor
  timer。
- 使用 `cx.background_executor().timer(duration).await`，或在
  `TestAppContext` 中使用对应 background executor timer。
- 避免在依赖 GPUI scheduler 的测试中用 `smol::Timer::after(...)`。

原因是 `smol::Timer` 可能不被 GPUI scheduler 跟踪，导致测试 pump 时出现
“nothing left to run” 类型问题。

## Visual regression tests

macOS 文档说明视觉回归测试当前是 macOS-only，并需要 Screen Recording 权限。

运行：

```sh
cargo run -p zed --bin zed_visual_test_runner --features visual-tests
```

baseline 图片在 `crates/zed/test_fixtures/visual_tests/`，但被 gitignore。首次或
有意更新 UI 时用：

```sh
UPDATE_BASELINE=1 \
  cargo run -p zed --bin zed_visual_test_runner --features visual-tests
```

UI 改动如果影响布局、panel、主题或核心交互，应考虑视觉测试或截图验证。

## 性能测量

`docs/src/development.md` 记录了 `ZED_MEASUREMENTS`：

```sh
export ZED_MEASUREMENTS=1
cargo run --release &> version-a
script/histogram version-a version-b
```

用于比较帧渲染性能。

单元 benchmark 可用 `util_macros::perf`，然后运行：

```sh
cargo perf-test -p <crate>
```

该 alias 来自 `.cargo/config.toml`。

## Windows ETW

Windows 上 Zed 支持 ETW profiling。命令面板中有：

- `zed: record etw trace`
- `zed: record etw trace with heap tracing`
- `zed: save etw trace`
- `zed: cancel etw trace`

ETW trace 可能包含路径、注册表键、进程名等敏感信息，分享前要注意。

## 崩溃排查

根 `.rules` 记录了 crash investigation 入口：

- `.factory/prompts/crash/investigate.md`
- `.factory/prompts/crash/fix.md`
- `script/sentry-fetch <issue-id>`
- `script/crash-to-prompt <issue-id>`

主程序中 crash handler 初始化在 `crates/zed/src/main.rs`。Dev channel 默认不安装
crash handler，除非设置 `ZED_GENERATE_MINIDUMPS=true` 或 `1`。

## 文档检查

mdBook 文档预览：

```sh
script/generate-action-metadata
mdbook serve docs
```

Prettier 格式化：

```sh
cd docs && pnpm dlx prettier@3.5.0 . --write && cd ..
```

如果新增或修改 `docs/src` 中使用 `{#kb ...}`、`{#action ...}` 的文档，必须先
确保 action metadata 可生成。

本组 `docs/zh` 当前是独立文档，没有加入 mdBook `SUMMARY.md`。

## 改动范围与验证建议

| 改动类型               | 建议验证                                                   |
| ---------------------- | ---------------------------------------------------------- |
| 单个纯逻辑 crate       | `cargo test -p <crate>`                                    |
| Editor 行为            | `cargo test -p editor`，必要时加相关 project/language 测试 |
| Project/LSP            | `cargo test -p project`，检查语言 server 相关测试          |
| UI panel               | 对应 crate 测试 + 手工启动 + 必要时视觉测试                |
| Settings/keymap/action | 相关测试 + `script/generate-action-metadata`               |
| Extension host/API     | extension、extension_host 测试 + dev extension 手工安装    |
| Collab server          | collab 测试 + 本地 Postgres/foreman 流程                   |

## 依据文件

- `.rules`
- `.cargo/config.toml`
- `script/clippy`
- `script/clippy.ps1`
- `docs/src/development.md`
- `docs/src/development/macos.md`
- `docs/src/development/windows.md`
- `docs/README.md`
- `crates/zed/src/main.rs`
