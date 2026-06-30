# 协作、云服务与 AI 模块

Zed 的协作、账号、云服务和 AI 不是单一模块。它们跨客户端 UI、RPC、云端
collab 服务、模型 provider、agent runtime 和设置系统。二次开发时需要先确认
目标是“本地 UI 行为”、“客户端协议接入”还是“云端服务”。

## 协作后端

`crates/collab/README.md` 说明 `collab` crate 是 Zed 运行在
`https://collab.zed.dev` 的后端逻辑。Zed 客户端在通过 `https://zed.dev`
认证后，通过 websocket 连接协作后端。README 还说明 `zed.dev` 是另一个运行在
Vercel 的独立仓库。

因此，二次开发协作能力时不要假定当前仓库包含完整账号站点。

## collab server 启动

`crates/collab/src/main.rs` 支持：

```sh
collab version
collab serve api
collab serve collab
collab serve all
```

启动 `serve` 时会：

- 加载 `.env.toml`。
- 解析环境变量到 `Config`。
- 初始化 tracing 和 panic hook。
- 建立 Axum router。
- 根据 mode 设置 database、RPC routes、API routes。
- collab mode 会创建 server epoch 并启动 `collab::rpc::Server`。
- api mode 会周期性从 blob store 获取 extensions，并挂载 events/extensions API。

## collab 配置

`crates/collab/src/lib.rs` 中 `Config` 包含：

- `http_port`
- `database_url`
- `database_max_connections`
- LiveKit 配置
- blob store 配置
- Kinesis 配置
- `zed_environment`
- cloud internal API key
- checksum seed

`zed_dot_dev_url()` 按 environment 返回 development、staging 或 production URL。
`zed_cloud_url()` 也按 environment 切换。

## 客户端协作模块

客户端侧从当前仓库结构可见的相关 crate：

- `client`：Zed 客户端、telemetry、user store、proxy 等。
- `rpc`：协作 server 和客户端间消息。
- `collab_ui`：协作面板、channel view、通知等 UI。
- `channel`：channel store 和 channel buffer。
- `call`：语音/通话相关逻辑。
- `livekit_api`、`livekit_client`：LiveKit 接入。

`crates/zed/src/main.rs` 中会初始化 `channel::init(...)`、`call::init(...)`、
`notifications::init(...)` 和 `collab_ui::init(...)`。

## AI 文档中的功能分层

`docs/src/ai/overview.md` 把 AI 分成三块：

- Agents：Zed Agent、External Agents、Terminal Threads。
- Model access：Zed-hosted models、provider API keys、subscriptions、gateways、
  local models。
- Features：Agent Panel、Parallel Agents、Inline Assistant、Edit Prediction、
  Git commit generation。

从代码结构看，相关 crate 包括：

- `agent`
- `agent_ui`
- `agent_settings`
- `agent_servers`
- `agent_skills`
- `language_model`
- `language_model_core`
- `language_models`
- `language_models_cloud`
- provider crate，例如 `anthropic`、`open_ai`、`google_ai`、`bedrock`、
  `ollama`、`lmstudio`、`open_router`、`deepseek`、`mistral`、`x_ai`。

## Agent 核心

`crates/agent/src/agent.rs` 当前入口可见：

- agent thread、thread store、tool permissions、tools、sandboxing、templates。
- skill 加载，包含 project skills、global skills、builtin skills。
- MCP prompt 名称冲突处理。
- `ProjectState` 持有 `Project`、`ProjectContext`、skills、context server registry。
- `Session` 同时持有内部 `Thread` 和 ACP thread。

`crates/zed/src/main.rs` 初始化 AI 相关能力时，会设置 language model、
language models、ACP tools、web search、edit prediction registry、
agent UI 和 user agents md watcher。

## Agent Panel 的条件加载

`crates/zed/src/zed.rs` 中 `setup_or_teardown_ai_panel` 会检查
`DisableAiSettings` 和测试环境。AI 被禁用或测试环境下，Agent Panel 可能不会
注册。

因此 UI 或 deep link 代码不能假定 `AgentPanel` 永远存在。当前
`OpenRequestKind::AgentPanel` 分支中，如果找不到 panel，会记录 warning。

## 二开风险点

- 自建协作服务不等于复制完整 Zed Cloud。认证站点、账号和部分云能力不在当前
  仓库里。
- AI provider、token、subscription 和 hosted model 涉及线上服务边界，应先看
  设置和 client 层。
- Agent tools 和 sandboxing 涉及文件系统权限，改动时要查 `agent::sandboxing`
  和 tool permissions。
- 协作项目有本地 shared 和远端 collab 两种状态。改 `Project` 行为时要检查
  `ProjectClientState`。

## 依据文件

- `crates/collab/README.md`
- `crates/collab/src/main.rs`
- `crates/collab/src/lib.rs`
- `docs/src/collaboration/overview.md`
- `docs/src/ai/overview.md`
- `crates/agent/src/agent.rs`
- `crates/project/src/project.rs`
- `crates/zed/src/main.rs`
- `crates/zed/src/zed.rs`
