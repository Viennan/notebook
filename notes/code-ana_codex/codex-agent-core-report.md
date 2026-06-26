# Codex Agent Core 主链路分析

本文分析 `openai/codex` 当前源码中，交互式 Codex CLI 从用户输入到模型请求、工具调用、审批/沙箱、事件回传的主链路。分析基于本 topic 内 submodule：

- 源码路径：[`./codex`](./codex)
- 上游 commit：`6d2168f06ae275d5e1f73cabf935d2bcc8549998`
- 重点范围：Rust workspace `codex-rs`，尤其是 CLI/TUI、app-server、core、protocol 与 tool runtime。

本文不是 crate-by-crate 的目录说明，而是围绕一次普通交互 turn 的运行链路展开。

## 核心结论

1. 当前交互式 `codex` 默认路径不是 TUI 直接调用 `codex-core`，而是 `CLI/TUI -> app-server -> core`。app-server 可以是嵌入式进程内 server、local daemon 或 remote endpoint。
2. core 的稳定边界是 [`CodexThread`](./codex/codex-rs/core/src/codex_thread.rs) 包装的 [`session::Codex`](./codex/codex-rs/core/src/session/mod.rs)。协议模型是 SQ/EQ：`Submission Queue` 接收 `Op`，`Event Queue` 输出 `Event`。
3. 一次普通 agent turn 是一个 `SessionTask`。[`RegularTask`](./codex/codex-rs/core/src/tasks/regular.rs) 很薄，真正的模型和工具循环在 [`run_turn`](./codex/codex-rs/core/src/session/turn.rs)。
4. `run_turn` 的核心循环是：准备上下文和工具 -> 构造 `Prompt` -> streaming 调模型 -> 持久化 response item -> 执行工具 -> 把工具输出写回历史 -> 必要时继续 follow-up sampling。
5. 工具执行的安全边界不在 UI 层，而在 core tool runtime：`ExecPolicyManager` 决定跳过/审批/拒绝，`ToolOrchestrator` 编排审批、sandbox 选择、首次执行和 sandbox denied 后的升级重试。
6. app-server 负责把 core 的低层事件翻译成 v2 客户端协议，例如 `TurnStarted`、`ItemStarted`、`CommandExecutionRequestApproval`、`TokenCount`、`TurnDiff`。TUI 主要消费这些 request/notification 并回传用户决定。

## 源码地图

| 层级 | 关键文件 | 主要职责 |
| --- | --- | --- |
| CLI | [`cli/src/main.rs`](./codex/codex-rs/cli/src/main.rs) | 解析命令；默认进入 TUI；`exec`/`review` 是旁路子命令。 |
| TUI | [`tui/src/lib.rs`](./codex/codex-rs/tui/src/lib.rs) | 启动 ratatui app；选择 embedded/local/remote app-server。 |
| TUI 输入 | [`tui/src/chatwidget/input_submission.rs`](./codex/codex-rs/tui/src/chatwidget/input_submission.rs)、[`tui/src/app/thread_routing.rs`](./codex/codex-rs/tui/src/app/thread_routing.rs) | 将用户输入、图片、skill/plugin/app mention 等整理为 `AppCommand::UserTurn`，并决定 `turn/start` 或 `turn/steer`。 |
| TUI 到 app-server | [`tui/src/app_server_session.rs`](./codex/codex-rs/tui/src/app_server_session.rs) | 将 TUI 操作编码成 app-server JSON-RPC 请求。 |
| app-server 请求 | [`app-server/src/message_processor.rs`](./codex/codex-rs/app-server/src/message_processor.rs)、[`request_processors/turn_processor.rs`](./codex/codex-rs/app-server/src/request_processors/turn_processor.rs)、[`request_processors/thread_processor.rs`](./codex/codex-rs/app-server/src/request_processors/thread_processor.rs) | 分发 `thread/start`、`turn/start`、`turn/steer`，映射 v2 输入到 core `Op`。 |
| core thread/session | [`core/src/thread_manager.rs`](./codex/codex-rs/core/src/thread_manager.rs)、[`core/src/codex_thread.rs`](./codex/codex-rs/core/src/codex_thread.rs)、[`core/src/session/mod.rs`](./codex/codex-rs/core/src/session/mod.rs) | 创建 thread/session，维护 `Submission` 与 `Event` 队列。 |
| core op 分发 | [`core/src/session/handlers.rs`](./codex/codex-rs/core/src/session/handlers.rs) | `submission_loop` 分发 `Op::UserInput`、approval、interrupt、compact 等。 |
| turn task | [`core/src/tasks/mod.rs`](./codex/codex-rs/core/src/tasks/mod.rs)、[`core/src/tasks/regular.rs`](./codex/codex-rs/core/src/tasks/regular.rs) | turn task 生命周期、取消、active turn 状态；regular turn 进入 `run_turn`。 |
| 模型循环 | [`core/src/session/turn.rs`](./codex/codex-rs/core/src/session/turn.rs)、[`core/src/client.rs`](./codex/codex-rs/core/src/client.rs)、[`core/src/stream_events_utils.rs`](./codex/codex-rs/core/src/stream_events_utils.rs) | 构造 prompt、调用 Responses stream、处理 streamed response、记录历史和工具输出。 |
| 工具系统 | [`core/src/tools/router.rs`](./codex/codex-rs/core/src/tools/router.rs)、[`core/src/tools/registry.rs`](./codex/codex-rs/core/src/tools/registry.rs)、[`core/src/tools/parallel.rs`](./codex/codex-rs/core/src/tools/parallel.rs)、[`core/src/tools/spec_plan.rs`](./codex/codex-rs/core/src/tools/spec_plan.rs) | 注册工具、生成 model-visible tool specs、调度 tool call、处理并发和 hooks。 |
| 权限与 sandbox | [`core/src/exec_policy.rs`](./codex/codex-rs/core/src/exec_policy.rs)、[`core/src/tools/orchestrator.rs`](./codex/codex-rs/core/src/tools/orchestrator.rs)、[`core/src/tools/sandboxing.rs`](./codex/codex-rs/core/src/tools/sandboxing.rs)、[`core/src/tools/runtimes/shell.rs`](./codex/codex-rs/core/src/tools/runtimes/shell.rs)、[`core/src/tools/runtimes/unified_exec.rs`](./codex/codex-rs/core/src/tools/runtimes/unified_exec.rs) | 命令审批策略、审批缓存、sandbox attempt、网络审批、执行重试。 |
| 事件回传 | [`app-server/src/request_processors/thread_lifecycle.rs`](./codex/codex-rs/app-server/src/request_processors/thread_lifecycle.rs)、[`app-server/src/bespoke_event_handling.rs`](./codex/codex-rs/app-server/src/bespoke_event_handling.rs)、[`tui/src/chatwidget/protocol_requests.rs`](./codex/codex-rs/tui/src/chatwidget/protocol_requests.rs) | 从 core event queue 拉事件，转成 app-server notification/request，再交给 TUI。 |

## 主链路总览

```text
codex binary
  -> cli::main / cli_main
  -> codex_tui::run_main
  -> App::run
  -> ChatWidget input submission
  -> AppCommand::UserTurn
  -> AppServerSession::turn_start or turn_steer
  -> app-server MessageProcessor
  -> TurnRequestProcessor
  -> CodexThread::submit_user_input_with_client_user_message_id / steer_input
  -> Codex::tx_sub <- Submission { id, op, trace }
  -> submission_loop
  -> user_input_or_turn_inner
  -> Session::spawn_task(RegularTask)
  -> RegularTask::run
  -> run_turn
  -> ModelClientSession::stream
  -> ToolRouter / ToolCallRuntime
  -> Session::send_event -> Codex::rx_event
  -> app-server thread_lifecycle
  -> bespoke_event_handling
  -> TUI ChatWidget
```

这条链路说明了一个关键分层：TUI 处理交互体验，app-server 处理 v2 API、线程状态和请求翻译，core 处理 agent session、模型 stream、工具和安全策略。

## 入口：CLI 到 TUI/app-server

[`cli/src/main.rs`](./codex/codex-rs/cli/src/main.rs) 的默认子命令进入交互式 TUI；`exec` 和 `review` 子命令调用 `codex_exec::run_main`，属于并列入口，不是本文主线。

交互式路径进入 [`codex_tui::run_main`](./codex/codex-rs/tui/src/lib.rs)。TUI 启动时会把 CLI flags、sandbox、approval、web-search、config override 等合并到运行配置，并选择 app-server 目标：

- explicit remote endpoint
- implicit local daemon
- embedded in-process app-server

因此，即使用户看到的是本地终端 UI，真正的 thread/turn 操作也是通过 app-server 协议提交的。

## Thread 创建：app-server 到 core session

TUI 首次进入工作流时会调用 app-server `thread/start`。app-server 的 [`ThreadRequestProcessor`](./codex/codex-rs/app-server/src/request_processors/thread_processor.rs) 会：

1. 加载配置与 overrides。
2. 解析信任、cwd、环境选择、dynamic tools、capability roots 等。
3. 调用 core [`ThreadManager::start_thread_with_options`](./codex/codex-rs/core/src/thread_manager.rs)。
4. 自动 attach thread listener，用于后续从 core event queue 拉事件。
5. 构造 `ThreadStartResponse` 返回 model、cwd、approval policy、sandbox、instruction sources 等 UI 需要的信息。

core [`ThreadManager`](./codex/codex-rs/core/src/thread_manager.rs) 再进入 [`Codex::spawn`](./codex/codex-rs/core/src/session/mod.rs)。`spawn_internal` 是 session 初始化的关键点：

- 创建 bounded `tx_sub/rx_sub` 作为 submission queue。
- 创建 unbounded `tx_event/rx_event` 作为 event queue。
- 加载或继承 exec policy。
- 解析 model、model info、base instructions、service tier、collaboration mode。
- 构造 [`Session`](./codex/codex-rs/core/src/session/session.rs)。
- 启动 `submission_loop(session_for_loop, config, rx_sub)` Tokio task。
- 返回 `Codex { tx_sub, rx_event, session, ... }`。

[`CodexThread`](./codex/codex-rs/core/src/codex_thread.rs) 是 app-server 持有的 thread facade。它把 `submit`、`submit_user_input_with_client_user_message_id`、`steer_input`、`next_event` 等操作代理到内部 `Codex`。

## 用户输入：TUI 到 Op::UserInput

TUI 的 [`input_submission.rs`](./codex/codex-rs/tui/src/chatwidget/input_submission.rs) 会把一次输入整理成结构化 `UserInput`：

- 文本输入
- remote/local image
- skill mention
- plugin mention
- app mention
- IDE context
- 当前 cwd、approval policy、permission profile、model、reasoning effort、service tier、collaboration mode、personality 等 turn settings

随后生成 `AppCommand::UserTurn`。[`thread_routing.rs`](./codex/codex-rs/tui/src/app/thread_routing.rs) 决定是：

- 当前有活跃 turn 且可 steer：调用 app-server `turn/steer`
- 当前没有活跃 turn：调用 app-server `turn/start`

app-server 的 [`TurnRequestProcessor::turn_start_inner`](./codex/codex-rs/app-server/src/request_processors/turn_processor.rs) 做核心转换：

1. 载入 thread 并确认允许 direct input。
2. 校验输入大小和图片 URL。
3. 将 app-server v2 `UserInput` 映射成 core input item。
4. 解析 cwd、environment selections、runtime workspace roots。
5. 构造 thread settings override，包括 approval、sandbox/permissions、model、service tier、reasoning、collaboration mode 等。
6. 构造 core `Op::UserInput { items, final_output_json_schema, responsesapi_client_metadata, additional_context, thread_settings }`。
7. 调用 `CodexThread::submit_user_input_with_client_user_message_id`，返回 submission id 作为 turn id。

`turn/steer` 的路径相似，但不会新建 turn，而是调用 `CodexThread::steer_input`，并要求 expected turn id 匹配当前活跃 turn。

## core 的 SQ/EQ 与 submission_loop

[`protocol/src/protocol.rs`](./codex/codex-rs/protocol/src/protocol.rs) 将 core 协议描述为 SQ/EQ：提交端发送 `Submission { id, op, client_user_message_id, trace }`，事件端输出 `Event { id, msg }`。

[`Codex::submit_with_id`](./codex/codex-rs/core/src/session/mod.rs) 只做一件核心事情：把 `Submission` send 到 `tx_sub`。后台 [`submission_loop`](./codex/codex-rs/core/src/session/handlers.rs) 从 `rx_sub` 顺序消费：

- `Op::UserInput` -> `user_input_or_turn`
- `Op::ExecApproval` -> 唤醒 pending exec approval
- `Op::PatchApproval` -> 唤醒 patch approval
- `Op::Interrupt` -> 中断任务
- `Op::ThreadSettings`、compact、rollback、realtime、shutdown 等走同一队列

这个设计让 UI/app-server 不需要直接管理 core 内部并发。它们只提交 typed op，core 自己维护 turn、task、approval 和 cancellation 状态。

## UserInput 如何变成 RegularTask

[`user_input_or_turn_inner`](./codex/codex-rs/core/src/session/handlers.rs) 是 `Op::UserInput` 进入 agent turn 的关键函数：

1. 从 `thread_settings` 生成 `SessionSettingsUpdate`，并绑定 `final_output_json_schema`。
2. 调用 `sess.new_turn_with_sub_id(sub_id, updates)` 创建新的 turn context。
3. 尝试 `sess.steer_input(...)`。
4. 如果已经有活跃 turn，输入被放入 active turn 的 pending input。
5. 如果返回 `SteerInputError::NoActiveTurn`，说明需要启动新任务：
   - 合并 additional context。
   - 构造 `TurnInput::ResponseItem` 和 `TurnInput::UserInput`。
   - 调用 `sess.spawn_task(..., RegularTask::new())`。

[`Session::spawn_task`](./codex/codex-rs/core/src/tasks/mod.rs) 会先 abort 当前任务，然后 `start_task`：

- 标记 turn start 时间。
- 创建 cancellation token。
- 更新 active turn 状态。
- 生成 task tracing span。
- Tokio spawn 具体 `SessionTask::run`。

[`RegularTask`](./codex/codex-rs/core/src/tasks/regular.rs) 只做少量 orchestration：

- inline 发出 `TurnStarted`，避免首 turn lifecycle 等待 prewarm。
- 消费可能存在的 startup prewarm model session。
- 调用 `run_turn(...)`。
- 如果 `run_turn` 结束后 input queue 仍有 pending input，则继续循环开启下一段处理。

## run_turn：模型与工具主循环

[`run_turn`](./codex/codex-rs/core/src/session/turn.rs) 是 agent core 的中心。它不是一次单纯模型调用，而是一个可多轮 sampling 的 turn loop。

每轮大致分为以下阶段：

1. 预处理：必要时做 pre-sampling compaction，生成 `StepContext`，记录 world state/context update。
2. 输入处理：运行 pending input hooks，把用户输入、additional context、inter-agent communication 写入历史。
3. 能力注入：`build_skills_and_plugins` 处理 mention 到的 skills/plugins/apps，并注入相应指导或 extension turn input。
4. 工具构建：`built_tools` 从 MCP、deferred MCP、dynamic tools、extension tools、connector snapshot 等生成 [`ToolRouter`](./codex/codex-rs/core/src/tools/router.rs)。
5. Prompt 构建：`build_prompt` 组合：
   - history input
   - model-visible tool specs
   - parallel tool call capability
   - base instructions
   - output schema
6. 调模型：`run_sampling_request` 调用 `ModelClientSession::stream`。
7. 流式处理：`try_run_sampling_request` 消费 `ResponseEvent`，处理 text/reasoning delta、tool argument delta、output item done、completed 等事件。
8. 工具 follow-up：如果模型发出 tool call，执行工具并将 tool output 写回历史；如果 response `end_turn == false` 或有 pending input，则继续下一轮 sampling。
9. 收尾：没有 follow-up 时运行 stop hooks、legacy after-agent hook，并返回最后 agent message。

[`ModelClientSession::stream`](./codex/codex-rs/core/src/client.rs) 是 core 到模型服务的边界。它在 Responses API wire path 下优先使用 WebSocket stream，失败后切到 HTTP Responses API，并通过 `x-codex-turn-state` 在同一 turn 内维持 sticky routing。

## Response stream 如何驱动工具

[`try_run_sampling_request`](./codex/codex-rs/core/src/session/turn.rs) 处理 Responses stream 时，有几个关键动作：

- `OutputItemAdded`：如果是 assistant message/reasoning 等非工具 item，先生成 UI 可见的 started item，并转发 text/reasoning delta。
- `ToolCallInputDelta`：把流式 tool argument delta 交给对应 tool 的 diff consumer，必要时发协议事件。
- `OutputItemDone`：调用 [`handle_output_item_done`](./codex/codex-rs/core/src/stream_events_utils.rs)。
- `Completed`：记录 token usage；如果 `end_turn == false`，标记需要 follow-up。
- stream 结束后 drain 所有 in-flight tool futures，再发 token count 和 turn diff。

[`handle_output_item_done`](./codex/codex-rs/core/src/stream_events_utils.rs) 对 tool call 和非 tool response item 分流：

- 如果 [`ToolRouter::build_tool_call`](./codex/codex-rs/core/src/tools/router.rs) 能从 `ResponseItem` 识别出 tool call：
  - 先把模型发出的 tool call 记录进历史。
  - 创建 `ToolCallRuntime::handle_tool_call` future。
  - 设置 `needs_follow_up = true`。
- 如果不是 tool call：
  - 转成 `TurnItem`。
  - 发 `ItemStarted`/`ItemCompleted`。
  - 写入历史，更新最后 agent message。
- 如果 tool call 不受支持但可回复模型：
  - 生成 `FunctionCallOutput` 错误响应写回历史。
  - 继续 follow-up。

[`drain_in_flight`](./codex/codex-rs/core/src/session/turn.rs) 会等待工具执行完成，并把每个工具输出转为 `ResponseItem` 写回 conversation history。下一轮 prompt 会从 history 中取到这些工具输出，模型因此可以继续推理。

## ToolRouter、ToolRegistry 与并发执行

工具系统有三层：

1. [`ToolRouter`](./codex/codex-rs/core/src/tools/router.rs)：持有 `ToolRegistry` 和 model-visible `ToolSpec`，负责从 model response item 构造 `ToolCall`。
2. [`ToolRegistry`](./codex/codex-rs/core/src/tools/registry.rs)：保存 tool name 到 runtime handler 的映射，处理 pre/post tool use hooks、telemetry、tool lifecycle start/finish。
3. [`ToolCallRuntime`](./codex/codex-rs/core/src/tools/parallel.rs)：负责并发控制、取消处理、把工具结果转成 `ResponseInputItem`。

并发控制在 `ToolCallRuntime` 中完成：如果 tool 声明支持 parallel，就使用 read lock；否则使用 write lock 串行化。取消时，如果 tool runtime 需要自己完成 teardown，则等待 runtime 退出；否则 abort task 并给模型返回 aborted tool output。

工具 specs 的生成集中在 [`tools/spec_plan.rs`](./codex/codex-rs/core/src/tools/spec_plan.rs)。这里会决定哪些工具对模型可见，哪些是 deferred 或 hidden，并把 MCP、dynamic tools、extensions、多 agent、shell、patch 等能力组合成最终 tool plan。

## 命令工具、审批与 sandbox

shell/unified exec 是最能体现 Codex agent 安全边界的工具族。

### exec policy 先判断是否需要审批

[`run_exec_like`](./codex/codex-rs/core/src/tools/handlers/shell.rs) 和 [`ExecCommandHandler`](./codex/codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs) 会先解析命令、cwd、权限请求、额外权限，并处理已授予的 turn/session permissions。

随后调用 [`ExecPolicyManager::create_exec_approval_requirement_for_command`](./codex/codex-rs/core/src/exec_policy.rs)。它会：

- 把 shell 命令解析为一个或多个 policy command。
- 用 exec-policy rules 和 fallback heuristics 评估。
- 返回 [`ExecApprovalRequirement`](./codex/codex-rs/core/src/tools/sandboxing.rs)：
  - `Skip { bypass_sandbox, proposed_execpolicy_amendment }`
  - `NeedsApproval { reason, proposed_execpolicy_amendment }`
  - `Forbidden { reason }`

这意味着“是否让用户审批”不是 UI 临时判断，而是 core policy 对每次命令执行的判定结果。

### ToolOrchestrator 编排审批、sandbox 与重试

[`ToolOrchestrator`](./codex/codex-rs/core/src/tools/orchestrator.rs) 是通用执行编排器，适用于 shell、unified exec、apply_patch 等 runtime。它的顺序是：

1. 根据 `ExecApprovalRequirement` 和 `AskForApproval` 决定是否请求 approval。
2. 若有 PermissionRequest hooks，先让 hooks 决定 allow/deny。
3. 若启用 guardian/strict auto-review，走 guardian review；否则走用户 approval。
4. 根据 permission profile、filesystem sandbox policy、network sandbox policy、tool sandbox preference 选择首次 `SandboxAttempt`。
5. 执行 tool runtime。
6. 如果 sandbox denied：
   - 检查是否允许升级重试。
   - 如果涉及 managed network denied，构造 network approval context。
   - 必要时再次请求 approval。
   - 构造 retry `SandboxAttempt`，可能不再使用 filesystem sandbox。

一个重要保护是 denied-read restrictions：如果 active filesystem policy 含有 denied-read path，`unsandboxed_execution_allowed` 会禁止脱 sandbox，因为那会丢失 denied-read 的唯一 enforcement 机制。

### approval 事件如何回到 core

当需要用户审批时，core [`Session::request_command_approval`](./codex/codex-rs/core/src/session/mod.rs) 会：

1. 在 active turn state 中插入 pending approval oneshot。
2. 构造 `EventMsg::ExecApprovalRequest`，包含 command、cwd、reason、network context、proposed amendment、available decisions 等。
3. 通过 `send_event` 发到 event queue。
4. 等待 oneshot 返回 `ReviewDecision`。

app-server 在 [`bespoke_event_handling.rs`](./codex/codex-rs/app-server/src/bespoke_event_handling.rs) 收到 `ExecApprovalRequest` 后，将它转换成 `CommandExecutionRequestApproval` server request。TUI 显示审批 UI，用户选择后，app-server 最终提交：

```text
Op::ExecApproval { id, turn_id, decision }
```

[`submission_loop`](./codex/codex-rs/core/src/session/handlers.rs) 收到该 op 后调用 `exec_approval`，再通过 `sess.notify_approval` 唤醒之前的 pending approval。

## 事件回传：core Event 到 TUI

core 事件统一由 [`Session::send_event`](./codex/codex-rs/core/src/session/mod.rs) 发出。它同时服务于：

- event queue：供 app-server/TUI 消费。
- rollout：持久化 thread 历史和状态。
- lifecycle/telemetry：部分事件会触发 turn lifecycle 或 realtime mirror。

app-server 的 [`thread_lifecycle`](./codex/codex-rs/app-server/src/request_processors/thread_lifecycle.rs) 持续调用 `conversation.next_event()`。每个 event 会先更新 thread-local state，然后交给 [`apply_bespoke_event_handling`](./codex/codex-rs/app-server/src/bespoke_event_handling.rs)。

这个 bespoke layer 会把 core events 转成客户端协议：

- `TurnStarted` -> `ServerNotification::TurnStarted`
- `TurnComplete` / `TurnAborted` -> turn 状态更新，并 abort pending server requests
- `ItemStarted` / `ItemCompleted` / `PatchApplyUpdated` / `TerminalInteraction` -> item notification
- `ExecCommandBegin` / `ExecCommandOutputDelta` / `ExecCommandEnd` -> command execution item 更新
- `ExecApprovalRequest` -> `CommandExecutionRequestApproval`
- `ApplyPatchApprovalRequest` -> `FileChangeRequestApproval`
- `RequestUserInput`、`RequestPermissions`、MCP elicitation、dynamic tool request -> 对应 server request
- `TokenCount`、`TurnDiff`、`PlanUpdate`、stream delta -> UI 进度和增量展示

TUI 的 [`chatwidget/protocol_requests.rs`](./codex/codex-rs/tui/src/chatwidget/protocol_requests.rs) 再把 server request 分发给审批弹窗、permission request、request user input、MCP elicitation 等 UI。

## 交互式路径与 exec/review 子命令

需要避免一个容易误读的点：`codex` 默认交互式路径走 TUI + app-server；但 `codex exec` 和 `codex review` 在 [`cli/src/main.rs`](./codex/codex-rs/cli/src/main.rs) 里进入 `codex_exec::run_main`。这些路径可能复用 core 能力，但入口和 UI/event adapter 不是本文这条 TUI 主链路。

后续若分析非交互式自动执行，应单独追踪 `codex-rs/exec`。

## 设计观察

### 1. core 是队列驱动的 agent runtime

app-server/TUI 不直接操作内部 turn 状态，而是提交 `Op`，再消费 `Event`。这种 SQ/EQ 边界让 interactive UI、remote app、embedded server、future clients 都可以复用同一个 core session。

### 2. app-server 是产品协议适配层

app-server 不只是薄代理。它维护 thread state、turn snapshots、pending requests、watchers、listener lifecycle，并把 core events 转成更稳定的 v2 app-server protocol。这解释了为什么 TUI 大量逻辑围绕 app-server request/notification，而不是直接解析 core event。

### 3. run_turn 是模型 loop，不是单次调用

工具调用、pending input、auto compact、stop hooks 都会让同一个 turn 继续 sampling。理解 Codex 行为时，应把 turn 看成“直到无需 follow-up 的 agent loop”，而不是“一次用户输入等于一次模型请求”。

### 4. 工具安全在 runtime 中闭环

审批 UI 只是人机交互入口，真正的判定链路在 core：

```text
exec policy -> approval requirement -> permission hooks/guardian/user approval
  -> sandbox attempt -> network approval -> retry policy -> tool output
```

这让 shell/unified exec/apply_patch 等工具可以共享审批缓存、policy amendment、network approval 和 sandbox denied 重试语义。

### 5. 历史是 prompt 与 UI 的共同事实源

模型 response item、tool call、tool output、turn item、token count、diff 等都会被记录或转译。下一轮 prompt 从 conversation history 构造，UI 从 event/rollout 构造可见状态。理解 bug 时，需要同时看 history 写入点和 event 发出点。

## 后续专题建议

1. `codex-tool-runtime.md`：细拆 `spec_plan` 中各类工具如何注册、曝光、隐藏、deferred、search。
2. `codex-sandbox-approvals.md`：专门分析 permission profile、sandbox policy、exec policy、network approval 与 guardian。
3. `codex-app-server-protocol.md`：分析 app-server v2 request/notification schema 与 thread state。
4. `codex-customization-surface.md`：分析 `AGENTS.md`、skills、plugins、MCP、hooks、dynamic tools 如何注入 `run_turn`。
