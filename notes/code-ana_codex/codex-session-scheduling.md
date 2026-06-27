# Codex Session 执行调度设计分析

本文分析 `openai/codex` 当前源码中 session 执行调度的设计：一次用户请求如何进入 session、如何变成 turn/task、运行中的 turn 如何吸收追加输入、如何中断和收尾，以及多 session / subagent 场景如何控制并发。分析以本 topic submodule 的源码为准，官方 Codex manual 的 thread/session 语境用于校准概念。

## 核心结论

Codex 的 session 调度不是一个集中式全局 scheduler，而是几层局部调度组合起来的：

1. **产品语义层**：官方语境中，一个 thread 就是一个 session；thread running 表示 Codex 正在处理该 thread。多个 thread 可以并行运行，但同一工作区同时改同一批文件需要用户自己避免。
2. **core session 层**：每个 [`Codex`](./codex/codex-rs/core/src/session/mod.rs) 是一对 queue：bounded submission queue 接收 `Op`，unbounded event queue 输出 `Event`。后台只有一个 `submission_loop` 顺序分发提交。
3. **task 层**：每个 [`Session`](./codex/codex-rs/core/src/session/session.rs) 同时最多一个 `RunningTask`。`Regular`、`Review`、`Compact` 都实现 [`SessionTask`](./codex/codex-rs/core/src/tasks/mod.rs)，`spawn_task` 会先 abort 当前任务再启动新任务。
4. **turn 内层**：一次 regular task 不是一次模型请求，而是 [`run_turn`](./codex/codex-rs/core/src/session/turn.rs) 的循环：sampling -> stream -> tool futures -> tool output 写回 history -> follow-up sampling。模型要求继续、工具调用、用户 steer、mailbox 消息、stop hook、自动 compact 都会让同一个 turn 继续跑。
5. **追加输入层**：`turn/steer` 不新建并发 turn，而是把用户输入放进 active regular turn 的 [`InputQueue`](./codex/codex-rs/core/src/session/input_queue.rs)，下一次采样边界再被模型看到。
6. **跨 session 层**：root thread 可以派生 subagent threads。单个 session 仍然一次只跑一个 task；跨 subagent 的数量和并发执行由 [`AgentControl`](./codex/codex-rs/core/src/agent/control.rs)、[`AgentRegistry`](./codex/codex-rs/core/src/agent/registry.rs) 和 execution guard 限制。
7. **app-server 层**：app-server 负责把 `turn/start`、`turn/steer`、`turn/interrupt` 翻译成 core `Op`，并用 listener task 从 core event queue 读取事件、翻译成 v2 通知/请求。它还会在 thread 无订阅且 idle 一段时间后卸载 session。

一句话概括：Codex 的调度原则是 **thread 间可以并行，thread 内 task 串行，turn 内通过边界点续跑，跨 subagent 用共享控制面限流**。

## 概念边界

### Thread、Session、Turn、Task

源码里几个词的关系如下：

| 概念 | 用户语义 | 源码对象 | 调度含义 |
| --- | --- | --- | --- |
| Thread | 一条可继续/恢复的对话或工作流 | [`CodexThread`](./codex/codex-rs/core/src/codex_thread.rs) | app-server 持有的 thread facade，代理 submit、steer、next_event。 |
| Session | 一个初始化后的 agent runtime | [`Session`](./codex/codex-rs/core/src/session/session.rs) | 保存状态、history、active turn、input queue、services；最多一个 running task。 |
| Turn | 一次用户触发或自动触发的工作段 | [`TurnContext`](./codex/codex-rs/core/src/session/turn_context.rs) | 携带 sub_id、配置快照、环境、telemetry、扩展数据。 |
| Task | 实际运行的异步工作 | [`SessionTask`](./codex/codex-rs/core/src/tasks/mod.rs) | `Regular` / `Review` / `Compact` 等任务类型；由 Tokio task 执行。 |
| Sampling step | 一次模型请求 | [`run_sampling_request`](./codex/codex-rs/core/src/session/turn.rs) | 一个 turn 内可多次发生，通常由工具输出或 pending input 触发 follow-up。 |

这里最容易混淆的是 thread/session/turn：官方说 thread 是 single session，源码为了实现会拆成 `CodexThread -> Codex -> Session -> TurnContext/RunningTask`。调度逻辑主要落在 `Session` 和 `tasks` 模块里。

## 总体调度图

```text
TUI / app / IDE / subagent
  -> app-server JSON-RPC: thread/start, turn/start, turn/steer, turn/interrupt
  -> CodexThread facade
  -> Codex::tx_sub bounded queue
  -> submission_loop 顺序分发 Op
  -> Session::spawn_task / steer_input / interrupt_task / notify approval
  -> RunningTask(Tokio)
  -> RegularTask::run
  -> run_turn 内循环
       -> model stream
       -> tool futures
       -> pending input / mailbox / compact / hooks
  -> Session::send_event
  -> Codex::rx_event
  -> app-server listener
  -> v2 turn/item/approval/thread notifications
```

这个图里有两个异步边界：

- `Codex::submit_with_id()` 只把 `Submission` 放进 queue，`turn/start` 因此能很快返回 turn id。
- `Session::start_task()` 再 `tokio::spawn` 真正的 task，后续靠 event queue 推进 UI。

## 第一层：SQ/EQ 队列模型

[`Codex`](./codex/codex-rs/core/src/session/mod.rs) 的注释直接说明它是 queue pair：发送 submissions，接收 events。`spawn_internal` 创建：

- `tx_sub/rx_sub = async_channel::bounded(SUBMISSION_CHANNEL_CAPACITY)`，容量是 512。
- `tx_event/rx_event = async_channel::unbounded()`。
- 后台 `tokio::spawn(submission_loop(...))`。

`Codex::submit()` / `submit_user_input_with_client_user_message_id()` 生成 submission id，然后调用 `submit_with_id()` 将 `Submission { id, op, trace, client_user_message_id }` 送入 `tx_sub`。`Codex::next_event()` 从 `rx_event` 读取 core 事件。

这一层的设计价值：

- 外部调用者不直接操作 session 内部状态，只提交 typed `Op`。
- `submission_loop` 顺序消费 `Op`，避免 approval、interrupt、settings update、user input 同时改 session 状态。
- 真正的模型和工具执行不堵住 submission loop；loop 只负责启动/通知/中断 task。

### FAQ：session_loop 跑在哪里？有专门执行器吗？

`session_loop` 实际是 [`Codex::spawn`](./codex/codex-rs/core/src/session/mod.rs) 中创建的 per-session Tokio task：`spawn_internal` 初始化 `Session` 后调用 `tokio::spawn(async move { submission_loop(session_for_loop, config, rx_sub).instrument(info_span!("session_loop", ...)).await })`，并把 `JoinHandle` 包装成 `session_loop_termination`，供 `shutdown_and_wait()` 或 `CodexThread::wait_until_terminated()` 等待。

所以它没有专门的自定义 executor，也不是单独 OS 线程或进程。它运行在当前进程的 Tokio runtime 上；常规 CLI/TUI/app-server 入口由 [`arg0_dispatch_or_else`](./codex/codex-rs/arg0/src/lib.rs) 包起来，先启动 `codex-main` OS 线程，再构建 `tokio::runtime::Builder::new_multi_thread().enable_all()` 的多线程 runtime，随后各层 `tokio::spawn` 都挂在这个 runtime 上。

需要区分两个“执行”层次：

- `session_loop` 是控制面 actor：从 `rx_sub` 串行消费 `Submission`，分发 `Op::UserInput`、`Op::Interrupt`、approval、compact、shutdown 等。
- 具体耗时 turn 是 task 执行面：例如 `UserInput` 在没有 active turn 时会调用 `Session::spawn_task(..., RegularTask::new())`；`Session::start_task()` 再 `tokio::spawn` 执行 `SessionTask::run()`。模型采样、工具调用、auto compact 等主要发生在这个 spawned task 里。

## 第二层：submission_loop 是控制面分发器

[`submission_loop`](./codex/codex-rs/core/src/session/handlers.rs) 按 `Op` 类型分发：

| Op | 调度行为 |
| --- | --- |
| `UserInput` | 进入 `user_input_or_turn_inner`，可能 steer 当前 turn，也可能启动 `RegularTask`。 |
| `Interrupt` | 调 `Session::interrupt_task()`，取消当前 task。 |
| `ExecApproval` / `PatchApproval` | 找到当前 turn 的 pending approval oneshot 并回复；`Abort` 会中断 task。 |
| `RequestPermissionsResponse` / `UserInputAnswer` / `DynamicToolResponse` / MCP elicitation | 回复 active turn 内的 pending waiter。 |
| `Compact` | 启动 `CompactTask`，会替换当前 task。 |
| `Review` | 启动 review 路径。 |
| `InterAgentCommunication` | 写入 mailbox；若 `trigger_turn` 且 session idle，则可能启动 regular turn。 |
| `RunUserShellCommand` | 有 active turn 时作为辅助命令跑；否则独立起一个 user-shell task。 |
| `ThreadRollback` | 要求没有 active turn；直接重建/persist history。 |
| `Shutdown` | 关闭 startup prewarm、active task、realtime、unified exec、MCP、guardian review session。 |

注意这里的 submission loop 不是耗时任务执行器。它更像 session 的控制面 actor：所有外部指令先经过它，再落到 `Session` 的状态机。

## 第三层：Session 同时最多一个 RunningTask

[`Session`](./codex/codex-rs/core/src/session/session.rs) 注释写得很直白：一个 session at most 1 running task at a time, and can be interrupted by user input。具体状态在 [`ActiveTurn`](./codex/codex-rs/core/src/state/turn.rs)：

```text
Session
  active_turn: Mutex<Option<ActiveTurn>>

ActiveTurn
  task: Option<RunningTask>
  turn_state: Arc<Mutex<TurnState>>

RunningTask
  kind: Regular | Review | Compact
  cancellation_token
  tokio handle
  done Notify
  turn_context
  agent_execution_guard
```

`Session::spawn_task()` 的策略很硬：

1. `abort_all_tasks(TurnAbortReason::Replaced)`，先取消当前 active task。
2. 清空 connector selection。
3. `start_task()` 创建新 task。

`start_task()` 则负责：

- 标记 turn start 时间和 turn metadata。
- 创建 cancellation token 和 `done` notify。
- 从 `InputQueue` 取出上一阶段 pending input，并绑定到当前 `TurnState`。
- 触发 turn start lifecycle。
- 创建 `SessionTaskContext`。
- 启动 Tokio task 执行 `SessionTask::run()`。
- 将 `RunningTask` 填入 `active_turn.task`。

这意味着 session 内不存在两个 regular turns 并发采样的情况。用户后续输入要么 steer 当前 regular turn，要么等当前 task 结束后再成为下一 turn。

## 第四层：RegularTask 是 turn 内循环壳

[`RegularTask`](./codex/codex-rs/core/src/tasks/regular.rs) 很薄，但它体现了 regular turn 的关键调度语义：

1. 先 inline 发 `TurnStarted`，避免首次 turn 被 startup prewarm 阻塞。
2. 消费 startup prewarm 出来的 `ModelClientSession`，若可用就复用。
3. 调 [`run_turn`](./codex/codex-rs/core/src/session/turn.rs)。
4. 如果 `InputQueue` 还有 pending input，就继续 loop，再调一次 `run_turn`；否则返回最后 agent message。

换句话说，一个 `RegularTask` 可覆盖“用户的一次 turn + 运行中追加输入 + 子 agent mailbox 后续输入”的连续处理，直到 input queue 真的清空。

更精确地说，`RegularTask` 外层多次 `run_turn` 的触发条件只有一个：某次 `run_turn` 返回后，`sess.input_queue.has_pending_input(&sess.active_turn)` 仍为 true。常见来源包括用户在 active regular turn 中 steer 进来的新输入、扩展/内部路径注入的 `ResponseItem`、以及当前 turn 仍接受 delivery 的 subagent mailbox 消息。大多数工具调用、`end_turn=false`、pending input follow-up、auto compact 和 stop hook continuation 都已经在同一个 `run_turn` 的内部 sampling loop 里处理；外层再次 `run_turn` 更像是处理“run_turn 即将结束或刚结束时又进入队列”的兜底机制。

### Startup prewarm：首 turn 的连接与上下文预热

[`Session::new`](./codex/codex-rs/core/src/session/session.rs) 完成 MCP manager 初始化后会调用 `schedule_startup_prewarm(base_instructions)`。它不是提前执行用户请求，也不会生成 assistant 答案；核心目标是降低首个 regular turn 的建连和请求准备延迟。

预热分两条路径：

- 如果 Responses WebSocket 未启用，只异步调用 `model_client.prewarm_auth()`，提前解析当前 auth/provider 设置，让 Agent Identity 的 session fallback 有机会在首个用户请求前就绪。
- 如果 Responses WebSocket 启用，则另起 Tokio task 构造一个 startup prewarm turn：创建 `TurnContext`、捕获 step context、构建 tool router、用空输入和 `base_instructions` 构造 prompt，生成 `CodexResponsesRequestKind::Prewarm` metadata，然后创建 `ModelClientSession` 并调用 `prewarm_websocket(...)`。

WebSocket 预热请求走正常 Responses-over-WebSocket 通道，但 `warmup=true` 会把 payload 设成 `generate=false`。这表示它是连接/setup 请求，不作为一次真正推理写入 trace；代码会等待 warmup stream 收到 `Completed`，这样首个真实 `run_turn` 可以复用同一个 `ModelClientSession`，包括 WebSocket 连接、sticky routing 状态和 warmup 后可复用的 `previous_response_id`。

首个 [`RegularTask`](./codex/codex-rs/core/src/tasks/regular.rs) 会先 inline 发 `TurnStarted`，避免 UI 被预热等待卡住，然后 `consume_startup_prewarm_for_regular_turn()` 取走这个 handle：成功就把预热好的 `ModelClientSession` 传给 `run_turn`，失败、未调度或超时就降级为普通 `new_session()`。如果 turn 在等待预热时被取消，预热 task 会 abort；session shutdown 也会 abort 未消费的 startup prewarm。

## 第五层：run_turn 是模型/工具内层 scheduler

`run_turn` 的开头注释说明：每次 sampling request，模型要么回 function/tool call，要么回 assistant message；有 tool call 时执行工具并把输出送回下一次 sampling request。

这里的 **sampling** 指一次“让模型基于当前 prompt 采样/生成下一批输出”的请求边界，源码入口是 `run_sampling_request()`。它不是整个 turn，也不是 tool 执行本身：一次 `run_turn` 可能包含多次 sampling；一次 sampling 内会构建 tools 和 prompt、打开/复用 `ModelClientSession` stream、消费 `ResponseEvent`，并在模型产出 tool call 时启动 tool future。tool future 完成后，tool output 会写回 history，随后进入下一次 sampling。

简化状态机：

```text
run_turn
  -> pre-sampling compact
  -> capture step context
  -> record context updates / skills / plugins / hooks / user input
  -> loop:
       drain pending input? 取决于 can_drain_pending_input
       capture step context
       build tool router
       build prompt
       stream model response
       handle OutputItemDone:
          non-tool item -> 记录 history / 发 item lifecycle
          tool call -> 启动 tool future / needs_follow_up = true
       Completed:
          记录 token usage / end_turn=false 也 needs_follow_up
       post sampling:
          needs_follow_up = model_needs_follow_up || has_pending_input
          if context limit reached -> inline auto compact -> continue
          if no follow-up -> run stop hooks / after-agent hook -> break
          else continue
```

这里有几个细节值得注意：

- `ModelClientSession` 是 turn-scoped，会跨同一 turn 的重试和 follow-up 复用，维持 WebSocket/sticky routing 状态。
- pending input 默认在下一轮 sampling 前写入 history，但 turn 一开始和 auto compact 后会延迟 drain，避免新用户输入打断工具/模型必须继续的上下文。
- `try_run_sampling_request` 用 `FuturesOrdered` 收集 in-flight tool futures；stream 结束后工具输出进入 history，再触发下一次模型请求。
- `ResponseEvent::Completed { end_turn: Some(false) }` 也会继续 follow-up，即使没有显式工具调用。
- stop hook 可以通过写入 continuation prompt 阻止 turn 结束，让同一 turn 继续采样。

这层调度让 Codex 不需要把“工具调用后的下一轮模型请求”建模为新的用户 turn；它仍属于同一 task/turn 的内部循环。

## Steer：运行中输入不抢占，只入队

app-server 的 `turn/steer` 通过 [`TurnRequestProcessor::turn_steer_inner`](./codex/codex-rs/app-server/src/request_processors/turn_processor.rs) 调 `CodexThread::steer_input()`，再到 [`Session::steer_input`](./codex/codex-rs/core/src/session/mod.rs)。

`steer_input` 的规则：

- 必须有 active turn。
- 必须有 active task。
- 如果传入 `expected_turn_id`，必须等于当前 active turn id。
- 只允许 steer `TaskKind::Regular`；`Review` 和 `Compact` 会返回 non-steerable。
- 输入不能为空。
- additional context 先 merge 到 session state。
- 最终写入 `InputQueue::extend_pending_input_and_accept_mailbox_delivery_for_turn_state()`。

所以 steer 是“向当前 regular turn 追加模型可见输入”，不是“打断当前采样并并发开新任务”。模型真正看到 steer 输入，要等 `run_turn` 到达 pending input drain 边界。

## Interrupt：取消 token + 短 grace + 终止事件

`turn/interrupt` 在 app-server 侧会校验 active turn id，然后提交 `Op::Interrupt`。core 进入 `Session::interrupt_task()` 和 `abort_all_tasks(TurnAbortReason::Interrupted)`。

中断路径的关键行为：

- 从 `active_turn` 取出 `RunningTask`。
- cancel task 的 `CancellationToken`。
- 最多等 100ms grace，让 task 自行观察 cancellation 并结束。
- 超时后 abort Tokio handle。
- 调 task 的 `abort()` hook。
- 如果配置要求，向 history 写入 model-visible interrupted marker。
- 发送 `TurnAborted`，flush rollout。
- 清理 pending waiters/pending input，避免 approval waiter 后续把拒绝误当成模型可见结果。
- 如果中断后 mailbox 里有 `trigger_turn` 消息，会尝试启动下一 turn。

这个设计把“用户中断”作为正常生命周期处理，而不是仅仅杀掉 future：history、事件、rollout 和 parent/subagent 通知都要保持一致。

## InputQueue 与 mailbox：边界点调度

[`InputQueue`](./codex/codex-rs/core/src/session/input_queue.rs) 管两类输入：

- turn-local `pending_input`：steer、injected response item、additional context 等。
- session-scoped `mailbox_pending_mails`：subagent 发来的 `InterAgentCommunication`。

它还维护一个 mailbox delivery phase：

| Phase | 含义 |
| --- | --- |
| `CurrentTurn` | mailbox 消息可并入当前 turn 的下一次模型请求。 |
| `NextTurn` | 当前 turn 已经输出用户可见答案，晚到 mailbox 不再延长当前答案，留到下一 turn。 |

这解决了 multi-agent 下一个细微问题：子 agent 晚到的结果不能随便把已经展示完的答案“续命”。源码测试覆盖了这些边界：

- late mailbox 在 answer boundary 后不延长当前 turn。
- `trigger_turn` mailbox 在 answer boundary 后等待下一 turn。
- 用户 steer 会重新打开当前 turn 的 mailbox delivery，让 steer 和排队的 child update 一起进入下一次采样。

## Idle 自动工作：低优先级、需让位

扩展可以在 thread idle 时触发自动工作，但必须走 [`try_start_turn_if_idle`](./codex/codex-rs/core/src/session/inject.rs) 这个门。

它拒绝以下情况：

- input 为空则直接 no-op。
- 已有 `trigger_turn` mailbox。
- 当前是 Plan mode。
- 有 active turn/task。
- 创建 turn context 过程中状态又变化。

如果通过，它会先占用一个空 `ActiveTurn` 作为 reservation，再写入 pending input，最后 `start_task(RegularTask)`。这个 reservation 防止“判断 idle”和“启动 task”之间被用户输入或 mailbox 抢占。

对应地，[`emit_thread_idle_lifecycle_if_idle`](./codex/codex-rs/core/src/tasks/lifecycle.rs) 只有在没有 active turn 且没有 trigger-turn mailbox 时才调用 thread idle contributors。这保证自动扩展任务不会压过用户或 subagent 显式触发的工作。

当前源码中，真正利用 thread idle 自动启动模型可见工作的内置扩展主要是 [`GoalExtension`](./codex/codex-rs/ext/goal/src/extension.rs)：`on_thread_idle` 调 `GoalRuntimeHandle::continue_if_idle()`，如果 thread 有 active goal，就构造 continuation steering item 并通过 `CodexThread::try_start_turn_if_idle()` 启动一个 automatic regular turn。app-server 在恢复 thread goal snapshot 后也会显式调用 `emit_thread_idle_lifecycle_if_idle()`，让 goal 可以在恢复后继续。

另一个相邻但优先级更高的机制是 subagent mailbox 的 `trigger_turn`：[`inter_agent_communication`](./codex/codex-rs/core/src/session/handlers.rs) 会把消息入队，若 `trigger_turn=true`，再由 `maybe_start_turn_for_pending_work_with_sub_id()` 在 session idle 时启动 regular turn。这不是 extension idle contributor，而是 core multi-agent pending work；它会阻止 `on_thread_idle` 被调用。

不要把所有“idle”都归为这类后台工作：app-server 的 idle thread unload 是资源回收，不会启动模型 turn；skills、web-search、image-generation、memories、guardian 等扩展虽然注册了 thread lifecycle contributor，但当前主要在 `on_thread_start` 初始化 thread-scoped 配置或状态，默认没有 idle 自动 turn。

## app-server listener：事件翻译与卸载

app-server 不是模型执行器，它维护 client-facing 的 thread lifecycle：

- `turn/start` 构造 core `Op::UserInput`，提交后立即返回 `TurnStartResponse { turn.id = submission id, status = InProgress }`。
- `turn/steer` 调 core `steer_input`，返回当前 active turn id。
- `turn/interrupt` 提交 `Op::Interrupt`；普通 turn 的响应要等 core 发出 `TurnAborted` 后再完成。
- `ensure_conversation_listener` 给 thread attach listener task。

[`thread_lifecycle.rs`](./codex/codex-rs/app-server/src/request_processors/thread_lifecycle.rs) 的 listener task 同时监听三类事情：

1. 取消信号：listener 被替换或 thread teardown。
2. listener command：例如 running thread resume response、thread goal notification、server request resolve。
3. `conversation.next_event()`：读取 core event queue，更新 `ThreadState`，再调用 bespoke event handling 翻译成 v2 notification/request。

它还有资源回收逻辑：如果 thread 没有 subscribers 且不是 active，等待 `THREAD_UNLOADING_DELAY`，当前是 30 分钟，然后 shutdown thread 并从 manager 移除。若 agent 仍是 running，则刷新 activity 时间，不卸载。

## 多 session / subagent 调度

单个 session 的 invariant 是“最多一个 running task”。但 Codex 支持 root thread 派生 subagent threads，所以还需要跨 session 控制面。

[`AgentControl`](./codex/codex-rs/core/src/agent/control.rs) 的注释说明它在一个 root session tree 内共享：

- root thread 和所有 subagent 持有同一个 `AgentControl`。
- `AgentRegistry` 记录 agent tree、agent path、nickname、last task message。
- `SpawnReservation` 在创建 subagent 前保留 slot，失败时 drop 释放。
- `agent_max_depth` 限制 thread-spawn 深度。
- `agent_max_threads` 限制 subagent 数量或 resident v2 threads。

执行并发还有一层 [`AgentExecutionLimiter`](./codex/codex-rs/core/src/agent/control/execution.rs)：

- 只限制 `MultiAgentVersion::V2` 的 subagent session。
- root session 和 V1 subagent 不计入这个 execution guard。
- `ensure_execution_capacity_for_op` 只对会开始 turn 的 op 生效：`UserInput` 或 `trigger_turn` 的 `InterAgentCommunication`。
- `Session::start_task` 创建 `agent_execution_guard`，task drop 后 guard 释放 active count。

这让 Codex 可以做到：

- root thread 可以继续控制和收集结果。
- 多个 subagent 可并行，但受 max threads / execution guard 约束。
- V2 resident subagent 还有 LRU 卸载：[`V2Residency`](./codex/codex-rs/core/src/agent/control/residency.rs) 在容量不足时尝试卸载可卸载 resident thread。

## 生命周期与错误恢复

task 结束统一由 `Session::on_task_finished()` 收口：

1. 取消 git enrichment。
2. 从 active turn 中取出 task handle 并 detach。
3. 把剩余 pending input 按 user-visible lifecycle 写入 history，避免 late steer 丢失。
4. 统计 memory/tool/token/turn profile metrics。
5. 根据结果发送 `TurnComplete` 或 `TurnAborted`。
6. 清理 active turn。
7. 若已 idle，触发 thread idle lifecycle。
8. flush rollout。

一个重要边界是错误也要释放 session。`core/tests/suite/stream_error_allows_next_turn.rs` 覆盖了 stream error 后仍应发 `Error` 和 `TurnComplete`，随后第二个用户 turn 可以正常提交。如果 active task 没被清掉，后续 turn 会被卡住或无限排队。

## 设计取舍

### 优点

- **单 session 串行化简单可靠**：`active_turn` + `RunningTask` 让同一 session 内不会出现两个模型循环同时改 history。
- **外部协议和执行解耦**：`turn/start` 返回快，后续全靠 event stream 推动 UI。
- **运行中交互自然**：steer、approval、permission response、MCP elicitation 都是同一 submission loop 的 op，不需要 UI 直接碰 internal future。
- **turn 内 continuation 统一**：工具、pending input、auto compact、stop hook 都复用 `run_turn` loop。
- **subagent 可并行但受控**：单 session 内串行，session tree 间通过 `AgentControl` 共享限额。
- **可恢复/可观察**：terminal event、rollout flush、token/turn metrics、thread state watch 都在任务收尾统一处理。

### 成本与复杂点

- **`active_turn` + `InputQueue` 的竞态边界细**：idle reservation、mailbox delivery phase、pending waiters 清理都需要严格测试。
- **turn 与 task 概念不完全一一对应**：一个 `RegularTask` 可能多次 `run_turn`，一个 `run_turn` 又可能多次 sampling。读代码时不能把 turn/start 等同于一次模型请求。
- **submission queue 顺序不等于模型执行顺序**：queue 只是控制面顺序；实际耗时在 spawned task 和 tool futures。
- **错误路径也必须产生 terminal event**：否则 app-server 和 UI 的 active turn 状态会悬挂。
- **跨 session 并发需要两套限制**：spawn/residency 限制“有多少 agent/thread”，execution guard 限制“有多少 v2 subagent turn 正在跑”。

## 关键源码入口

core session/control plane：

- [`core/src/session/mod.rs`](./codex/codex-rs/core/src/session/mod.rs)：`Codex` queue pair、spawn、submit、steer、interrupt、event 发送。
- [`core/src/session/handlers.rs`](./codex/codex-rs/core/src/session/handlers.rs)：`submission_loop` 与 `Op` 分发。
- [`core/src/session/session.rs`](./codex/codex-rs/core/src/session/session.rs)：`Session` 状态和配置。
- [`core/src/codex_thread.rs`](./codex/codex-rs/core/src/codex_thread.rs)：app-server 持有的 thread facade。
- [`core/src/thread_manager.rs`](./codex/codex-rs/core/src/thread_manager.rs)：thread 创建、恢复、fork、全局 thread registry。

task / turn：

- [`core/src/tasks/mod.rs`](./codex/codex-rs/core/src/tasks/mod.rs)：`SessionTask`、`spawn_task`、`start_task`、abort、finish lifecycle。
- [`core/src/tasks/regular.rs`](./codex/codex-rs/core/src/tasks/regular.rs)：regular turn task 壳。
- [`core/src/tasks/compact.rs`](./codex/codex-rs/core/src/tasks/compact.rs)：manual compact task。
- [`core/src/tasks/review.rs`](./codex/codex-rs/core/src/tasks/review.rs)：review task。
- [`core/src/session/turn.rs`](./codex/codex-rs/core/src/session/turn.rs)：model/tool/pending-input/compact 内层循环。
- [`core/src/session/input_queue.rs`](./codex/codex-rs/core/src/session/input_queue.rs)：turn-local pending input 和 inter-agent mailbox。
- [`core/src/state/turn.rs`](./codex/codex-rs/core/src/state/turn.rs)：`ActiveTurn`、`RunningTask`、`TurnState`。
- [`core/src/session/inject.rs`](./codex/codex-rs/core/src/session/inject.rs)：idle 自动 turn 和 active-turn 注入。
- [`core/src/tasks/lifecycle.rs`](./codex/codex-rs/core/src/tasks/lifecycle.rs)：turn/thread lifecycle contributor 调用。

app-server：

- [`app-server/src/request_processors/turn_processor.rs`](./codex/codex-rs/app-server/src/request_processors/turn_processor.rs)：`turn/start`、`turn/steer`、`turn/interrupt` 到 core op 的转换。
- [`app-server/src/request_processors/thread_lifecycle.rs`](./codex/codex-rs/app-server/src/request_processors/thread_lifecycle.rs)：listener task、event 翻译、idle unload、running resume。
- [`app-server/src/bespoke_event_handling.rs`](./codex/codex-rs/app-server/src/bespoke_event_handling.rs)：core event 到 v2 client protocol 的定制映射。

multi-agent / concurrency：

- [`core/src/agent/control.rs`](./codex/codex-rs/core/src/agent/control.rs)：root session tree 共享的 agent control plane。
- [`core/src/agent/control/execution.rs`](./codex/codex-rs/core/src/agent/control/execution.rs)：v2 subagent execution guard。
- [`core/src/agent/control/spawn.rs`](./codex/codex-rs/core/src/agent/control/spawn.rs)：spawn subagent、继承环境/exec policy、slot reservation。
- [`core/src/agent/control/residency.rs`](./codex/codex-rs/core/src/agent/control/residency.rs)：V2 resident thread LRU 卸载。
- [`core/src/agent/registry.rs`](./codex/codex-rs/core/src/agent/registry.rs)：agent tree、spawn reservation、depth/path/nickname 管理。

测试证据：

- [`core/tests/suite/stream_error_allows_next_turn.rs`](./codex/codex-rs/core/tests/suite/stream_error_allows_next_turn.rs)：stream error 后 session 释放，下一 turn 可继续。
- [`core/src/session/tests.rs`](./codex/codex-rs/core/src/session/tests.rs)：steer、pending input、thread idle、mailbox phase、idle auto turn 等边界。
- [`core/src/agent/control/execution_tests.rs`](./codex/codex-rs/core/src/agent/control/execution_tests.rs)：V2 subagent execution guard 限制与释放。
