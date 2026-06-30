# Codex 研究知识库索引

本 topic 针对 [`openai/codex`](https://github.com/openai/codex) 做代码分析。该仓库是 OpenAI Codex CLI、SDK 与 app-server 等开源组件的主要源码入口；分析以本 topic 内的源码 submodule 为准，官方文档用于补充产品语境和用户可见行为。

## 仓库信息

- 上游仓库：<https://github.com/openai/codex>
- submodule 路径：[`./codex`](./codex)
- 初始化分支：`main`
- 初始化 commit：`6d2168f06ae275d5e1f73cabf935d2bcc8549998`
- 递归子模块：无
- 依赖初始化：未执行
- 技术栈线索：Rust workspace（`codex-rs/Cargo.toml`、`codex-rs/Cargo.lock`）、pnpm workspace（`package.json`、`pnpm-lock.yaml`）、Python SDK/runtime（`sdk/python/pyproject.toml`、`sdk/python-runtime/pyproject.toml`）

## 初始化状态

当前已完成：

- 创建分析目录和权威参考资料索引。
- 引入源码 submodule。
- 记录本地 submodule commit、分支和递归子模块状态。
- 将 `notes/code-ana_codex/codex/**` 加入 `.vscode/settings.json` 的 `files.watcherExclude`。

当前暂不初始化依赖。后续若为了类型跳转或本地构建需要依赖，应优先使用冻结、只读、不开安装脚本的命令，例如在源码目录中执行 `pnpm install --frozen-lockfile --ignore-scripts`，或在 `codex-rs/` 中执行 `cargo fetch --locked`。

## 文档列表

### [`codex-agent-core-report.md`](./codex-agent-core-report.md)

Codex 交互式 agent core 主链路报告。覆盖默认 `codex` 入口到 TUI/app-server、`thread/start` 与 `turn/start`、core SQ/EQ、`submission_loop`、`RegularTask`/`run_turn`、Responses stream、工具执行、exec policy、approval/sandbox、以及 core event 回传到 TUI 的端到端链路。

### [`codex-sandbox-approvals.md`](./codex-sandbox-approvals.md)

Codex sandbox 与权限边界专题报告。覆盖 sandbox 的产品语义、`PermissionProfile` 权限模型、approval policy 分工、shell 工具执行链路、macOS Seatbelt、Linux bubblewrap/seccomp、Windows restricted token/ACL/WFP，以及 network proxy / network approval 的关系。

### [`codex-session-scheduling.md`](./codex-session-scheduling.md)

Codex session 执行调度专题报告。覆盖 thread/session/turn/task 的关系、SQ/EQ 队列、`submission_loop` 分发、`SessionTask` 与 active turn 状态机、`run_turn` 内层循环、steer/interrupt/input queue/mailbox/idle 自动任务、app-server listener/unload，以及 multi-agent/subagent 并发限流。

### [`codex-multi-agent.md`](./codex-multi-agent.md)

Codex multi-agent / subagent 机制专题报告。覆盖 V1 `multi_agent_v1` 与 V2 `collaboration` 工具族、`AgentControl` / `AgentRegistry` / `AgentPath` / mailbox 架构、spawn/fork/resume 生命周期、runtime override 继承、V2 residency 与 execution limiter、模型可见 tool specs / prompt hints / `AgentMessage`、app-server collab item、hook、配置和 `spawn_agents_on_csv` 批处理扩展。

### [`codex-goal-mode.md`](./codex-goal-mode.md)

Codex Goal 模式专题报告。覆盖 Goal 为何要把超长 step loop 从单 turn 连续工具闭环提升为跨 turn durable workflow、Goal 与普通非 goal 模式的差异、持久化 `ThreadGoal` 状态机、`get_goal`/`create_goal`/`update_goal` 工具、Goal 特有的 hidden steering `user` role message、idle 自动续跑、完成/阻塞审计、token/time 记账、预算与错误防循环、resume 恢复顺序、长 objective 附件化，以及 Goal 与 compaction 兼容性的 FAQ。

### [`codex-responses-api.md`](./codex-responses-api.md)

Codex Responses API 使用与 Prompt Cache 设计专题报告。覆盖 `run_turn` 到 `ModelClientSession::stream` 的请求链路、Responses request 结构、stream event 映射、HTTP SSE / WebSocket transport、`previous_response_id` 增量请求、Responses Lite、reasoning encrypted content、remote compaction、`prompt_cache_key` scope、稳定前缀与 `cached_input_tokens` 账本。

### [`codex-prompt-system.md`](./codex-prompt-system.md)

Codex Prompt 系统与 Compaction 策略专题报告。覆盖 base/model instructions、developer/user context fragments、工具 specs、collaboration/goal/review/realtime/memory 等特殊 prompt、普通 turn prompt 组装链路、Responses Lite 布局、local/remote/remote v2/token-budget compaction，以及 compaction prompt 模板与 replacement history。

### [`ref/INDEX.md`](./ref/INDEX.md)

官方或权威资料入口。当前记录上游仓库、官方 Codex 文档、Codex CLI 文档、开源组件说明、README、贡献与安装资料等，供后续分析按需加载。

## 后续分析入口

源码引入后，可优先补充：

1. `codex-rs-architecture.md`：`codex-rs` workspace、core/TUI/CLI/app-server/protocol 等 crate 的职责边界。
2. `codex-tool-runtime.md`：shell、patch、file read/write、MCP、browser/app 等工具能力如何注册、审批、执行和回传。
3. `codex-customization-surface.md`：`AGENTS.md`、skills、plugins、MCP、hooks、config 与 memory 在运行时上下文中的位置。
