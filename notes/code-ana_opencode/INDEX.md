# opencode 研究知识库索引

本 topic 针对 [`anomalyco/opencode`](https://github.com/anomalyco/opencode) 做代码分析，源码作为 submodule 放在 [`./opencode`](./opencode)。当前报告聚焦 opencode 作为 AI 编码代理的核心工作链路：session loop、LLM streaming、工具调用、权限、plan/coding 切换与 subagent；暂不展开配置系统、企业版和 TUI。

## 文档列表

### [`opencode-agent-core-report.md`](./opencode-agent-core-report.md)

主研究报告，重点覆盖：

- 当前公开 HTTP/SDK prompt 路径如何进入 `SessionPrompt`
- `SessionPrompt.runLoop` 如何组织 user message、assistant message、compaction、subtask、agent/model、tools 和 continuation
- `SessionProcessor` 如何把 LLM stream 映射为 text/reasoning/tool/session state
- `LLM.stream` 如何选择 native runtime 或 AI SDK，并统一转换成 `LLMEvent`
- V2 durable runner 的 prompt admission、wake/run coordinator、session runner、tool settlement 与 continuation
- 当前旧版路径和 V2 迁移目标的边界

适合先读，用来建立 opencode coding agent 主干执行链路。

### [`opencode-tools-plan-subagents.md`](./opencode-tools-plan-subagents.md)

工具、plan 模式与 subagent 专题报告，重点覆盖：

- `build`、`plan`、`general`、`explore` 等 agent 如何通过权限和 prompt 形成不同工作模式
- plan 模式如何通过 synthetic reminder 和 `plan_exit` 切换到 build
- 工具 registry 如何被包装成 AI SDK tools
- 权限系统如何在 allow/ask/deny 之间决策
- tool part 生命周期如何被 processor 持久化
- `task` 工具如何创建 child session 并递归调用同一套 prompt loop
- V2 typed tool registry 与当前旧版工具系统的差异

适合理解 opencode 为什么能从“回答问题”升级为“持续读写代码、调用工具、委派子任务的编码代理”。

### [`opencode-multi-agent.md`](./opencode-multi-agent.md)

multi-agent 能力与设计专题报告，重点覆盖：

- opencode 的 multi-agent 为什么更像 task/subagent session delegation，而不是自治 agent swarm
- primary agent、subagent、agent profile、mode、description、model、permission 之间的关系
- `task` 工具如何作为 multi-agent 调用边界，创建或复用 child session
- 模型自动委派、用户 `@subagent` 手动委派、`permission.task` 和 `bypassAgentCheck` 的差异
- parent/child session 在 transcript、tool state、permission、result 回流上的隔离与共享
- foreground/background subagent、`task_id` resume、background result synthetic prompt 注入
- opencode multi-agent 设计背后的上下文经济、权限边界和工程取舍

适合理解 opencode 如何把多代理能力实现为可审计、可取消、可权限约束的父子任务委派机制。

### [`opencode-edit-tool.md`](./opencode-edit-tool.md)

`edit` 工具专题研究报告，重点覆盖：

- `edit` 作为局部代码修改能力，如何连接 read、permission、diff、formatter、watcher、LSP 和后续迭代
- 旧版 `edit` 的参数、执行链路、diff metadata、权限审批和写入后诊断反馈
- 旧版 fuzzy replacer pipeline 如何处理 LLM 常见复制/缩进/转义/上下文偏差
- V2 `edit` 如何收缩为 exact-edit leaf，并通过 typed schema、permission-first、conditional commit 固定核心语义
- `edit` 与 `write`、`apply_patch`、`read` 的边界，以及典型 coding 任务中的使用情形
- 从 prompt-heavy 工具工作流迁移到 runtime-heavy 工具核心的设计取舍

适合理解 opencode 如何把“读懂代码后做一个小而具体的修改”工程化成可审批、可诊断、可继续迭代的工具协议。

### [`opencode-session-v1-v2.md`](./opencode-session-v1-v2.md)

SessionV1 与 SessionV2 深度对比报告，重点覆盖：

- V1 的 session info、message/part、coarse-grained event 模型
- V2 的 prompt admission、session input queue、domain event、projection、message reducer、execution coordinator、context epoch
- 从 V1 到 V2 的核心变化：从可变 transcript 到 durable agent runtime
- V2 新抽象之间的关系图
- 设计哲学变化：event-sourced thinking、runtime-first、durable work queue、provider-neutral canonical message
- 为什么当前代码仍保留 V1/V2 双轨迁移痕迹

适合在理解主执行链路后，深入把握 opencode session runtime 的演进方向。

### [`opencode-v2-layered-architecture.md`](./opencode-v2-layered-architecture.md)

SessionV2 抽象分层总览，重点覆盖：

- V2 从 public API 到 runner、event、projection、read model 的整体层次图
- public API 为何很薄，runtime layer 为何很厚
- Session facade、SessionInput、SessionExecution、SessionRunner、SessionContextEpoch、ToolRegistry、SessionProjector 的分工
- Location layer 如何作为 coding agent 的横切边界
- 当前 V2 已完成与仍在迁移中的部分

适合作为继续逐层深挖 V2 的总入口。

### [`opencode-v2-input-execution.md`](./opencode-v2-input-execution.md)

V2 输入生命周期与执行控制专题，重点覆盖：

- prompt 从直接写 user message 变成 `admit -> wake -> promote`
- `SessionInput.Admitted`、`delivery=steer/queue`、`session_input` 表的语义
- prompt admission 与 runner promotion 的关系
- `SessionExecution` 的 `wake/resume/interrupt`
- `SessionRunCoordinator` 如何保证同一 session 单 active drain、wake 合并、run 升级和 interrupt 边界
- 为什么这一层把 request-owned promise 改造成 session-owned execution lane

适合理解 V2 如何处理长任务、用户中途输入、排队和中断。

### [`opencode-v2-events-projection.md`](./opencode-v2-events-projection.md)

V2 领域事件、projection 与 read model 专题，重点覆盖：

- `SessionEvent` 的 durable/ephemeral 事件体系
- text/reasoning/tool/compaction 生命周期如何被记录
- `SessionMessage` 作为 canonical read model 的设计
- `SessionMessageUpdater` 如何像 reducer 一样把 event 折叠成 message
- `SessionProjector` 如何同时处理 V1/V2 双轨投影
- `SessionStore` 与 `SessionHistory` 如何给 API 和 runner 提供上下文

适合理解 V2 为什么从状态同步转向 event-sourced thinking。

### [`opencode-v2-runner-context-tools.md`](./opencode-v2-runner-context-tools.md)

V2 runner、context 与 tool settlement 专题，重点覆盖：

- `SessionRunner.run` 的 open activity loop 与 `MAX_STEPS`
- `runTurnAttempt` 如何组装一次 provider turn
- `SessionContextEpoch` 如何维护 system baseline、revision、replacement
- `SessionHistory` 与 `toLLMMessages` 如何决定模型看到什么
- `SessionRunnerModel` 如何从 catalog 解析 provider-neutral model
- `createLLMEventPublisher` 如何把 LLM stream 变成 session events
- `ToolRegistry.materialize/settle`、typed tool、ToolOutputStore、PermissionV2 的协作
- compaction 如何成为 runner 的上下文维护事件

适合理解 V2 agent turn 的真实执行内核。

### [`opencode-prompt-system.md`](./opencode-prompt-system.md)

opencode 提示词系统深度研究，重点覆盖：

- V2 core runner 与旧 `SessionPrompt` 路径中 system prompt 的组成差异
- `ContextEpoch`、AGENTS.md、skill guidance、reference guidance 如何进入系统上下文
- `bash`、`edit`、`write`、`read`、`grep/glob`、`todowrite`、`skill`、`question` 等常用工具的提示词和结果渲染设计
- MCP tools/prompts/resources 如何被纳入工具、command 和附件上下文
- skill 系统如何采用“索引优先、按需加载”的提示词策略
- opencode prompt 设计的方法论、coding task 组织方式、magic 机制与记忆系统边界

适合理解 opencode 如何把提示词、工具定义、权限、上下文源和运行时事件组织成一个 coding agent。

## 当前结论摘要

opencode 的 agent 主干是一个 session continuation loop。plan、coding、subagent 共享同一套循环，差异主要来自 agent profile、权限规则、工具集合和 synthetic prompt。旧版公开路径仍以 `packages/opencode/src/session/prompt.ts` 为核心；V2 core 已经实现更 durable 的 prompt queue、runner、domain events、projection 和 tool settlement，是迁移中的目标架构。

## 建议阅读顺序

1. 先读 [`opencode-agent-core-report.md`](./opencode-agent-core-report.md)，理解用户 prompt 如何变成 agent loop。
2. 再读 [`opencode-tools-plan-subagents.md`](./opencode-tools-plan-subagents.md)，理解 plan/coding/subagent 和工具权限如何组成行动能力。
3. 读 [`opencode-multi-agent.md`](./opencode-multi-agent.md)，进一步理解 task/subagent session delegation 的能力边界与设计取舍。
4. 读 [`opencode-edit-tool.md`](./opencode-edit-tool.md)，深入理解最核心的局部代码修改工具如何工作。
5. 接着读 [`opencode-session-v1-v2.md`](./opencode-session-v1-v2.md)，理解 V1/V2 session runtime 的抽象差异和设计哲学。
6. 然后读 [`opencode-v2-layered-architecture.md`](./opencode-v2-layered-architecture.md)，建立 V2 分层地图。
7. 按需要逐层读：
   [`opencode-v2-input-execution.md`](./opencode-v2-input-execution.md)
   [`opencode-v2-events-projection.md`](./opencode-v2-events-projection.md)
   [`opencode-v2-runner-context-tools.md`](./opencode-v2-runner-context-tools.md)
8. 最后读 [`opencode-prompt-system.md`](./opencode-prompt-system.md)，理解提示词、工具、MCP、skills、权限和记忆边界如何合成 coding agent。
9. 后续若继续扩展，可补充：
   `opencode-message-storage.md`
   `opencode-v2-location-runtime.md`
   `opencode-v2-permission-policy.md`
