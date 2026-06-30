# 构建、运行与本地环境

本文整理二次开发前需要确认的本地环境、常用构建命令和平台差异。

## Rust 与 Cargo

仓库通过 `rust-toolchain.toml` 固定工具链：

```toml
[toolchain]
channel = "1.95.0"
profile = "minimal"
components = [ "rustfmt", "clippy", "rust-analyzer", "rust-src" ]
```

同时配置了三个 target：

- `wasm32-wasip2`：扩展使用。
- `wasm32-unknown-unknown`：GPUI web 目标使用。
- `x86_64-unknown-linux-musl`：remote server 使用。

根 `.cargo/config.toml` 设置了全局 `rustflags`：

```toml
rustflags = ["-C", "symbol-mangling-version=v0", "--cfg", "tokio_unstable"]
```

Windows 目标还额外设置 `windows_slim_errors` 和静态 CRT。不要随意用
`RUSTFLAGS` 覆盖这些配置。现有 Windows 文档说明，覆盖后可能导致链接或
构建错误。

## 主程序构建

根 `Cargo.toml` 的 `default-members` 是 `crates/zed`，主程序 crate 的
二进制入口是 `crates/zed/src/main.rs`。

常用命令：

```sh
cargo run
cargo run --release
cargo test --workspace
```

如果只验证某个 crate，优先缩小范围：

```sh
cargo test -p editor
cargo test -p project
```

项目规则要求做 Clippy 时使用脚本：

```sh
script/clippy
```

Windows 下有 PowerShell 版本：

```powershell
script\clippy.ps1
```

脚本默认在没有 `-p` 或 `--package` 时检查整个 workspace，并使用
`--release --all-targets --all-features -- --deny warnings`。

## 平台依赖

macOS：

- 安装 `rustup`。
- 安装 Xcode 和 Xcode command line tools。
- 安装 `cmake`。
- 视觉回归测试只在 macOS 文档中说明，且需要 Screen Recording 权限。

Linux：

- 安装 `rustup`。
- 运行 `script/linux` 安装系统库，或者按该脚本手动安装。
- 开发调试可以直接 `cargo run`。
- 发布/打包需要同时考虑 `cli` 和 `zed` 两个二进制。

Windows：

- 安装 `rustup`。
- 安装 Visual Studio 或 Build Tools，包含 C++ build tools、Spectre
  mitigated libs 和 Windows SDK。
- 安装 CMake。
- 文档要求至少有 Windows 10 SDK 2104，也给出了 VS 组件清单。
- 若只装 Build Tools，需要从开发者 shell 启动，让编译环境变量生效。

## CLI crate

`crates/cli/README.md` 给出的本地测试方式是先构建主二进制，再运行 CLI：

```sh
cargo build -p zed
cargo run -p cli -- --zed ./target/debug/zed.exe
```

这个流程用于验证 `cli` crate 对主程序二进制的调用。非 Windows 平台把
二进制路径换成对应产物路径。

## 文档本地预览

`docs/README.md` 说明文档站使用 mdBook，并依赖动作元数据：

```sh
script/generate-action-metadata
mdbook serve docs
```

如果 action 或 keybinding 发生变化，需要重新生成动作元数据。

格式化命令按现有文档为：

```sh
cd docs && pnpm dlx prettier@3.5.0 . --write && cd ..
```

本组 `docs/zh` 文档当前没有接入 mdBook 目录，也没有使用 `{#kb ...}` 这类
预处理语法。

## 协作服务本地运行

`crates/collab/README.md` 说明 `collab` crate 是 Zed 在
`https://collab.zed.dev` 运行的后端逻辑。它需要 Postgres。

本地协作开发流程：

```sh
script/bootstrap
foreman start
script/zed-local -2
```

`script/bootstrap` 会设置 `zed` Postgres 数据库并填充默认用户。README 说明
该脚本需要访问 GitHub API。

## 新 crate

仓库有 `script/new-crate`，它会在 `crates/<name>` 下生成 `Cargo.toml`、
`src/<name>.rs` 和许可证链接。脚本末尾仍提示需要手动把新 crate 加入
workspace。

根 `.rules` 还要求：

- 新 crate 优先用 `[lib] path = "...rs"`，避免默认 `lib.rs`。
- 不创建 `mod.rs` 路径。
- 新一方 crate 默认 GPL，必要时可用 Apache。

## 依据文件

- `rust-toolchain.toml`
- `.cargo/config.toml`
- `Cargo.toml`
- `crates/zed/Cargo.toml`
- `docs/src/development/macos.md`
- `docs/src/development/linux.md`
- `docs/src/development/windows.md`
- `docs/README.md`
- `script/clippy`
- `script/clippy.ps1`
- `script/new-crate`
- `crates/cli/README.md`
- `crates/collab/README.md`
