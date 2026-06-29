# Codex Goal 模式设计分析

本文分析 `openai/codex` 中的 Goal 模式：它相比普通非 goal 运行有什么不同，Codex 通过哪些机制提高“目标被完成”的概率，以及它如何处理多步骤、长时间运行任务。分析以本 topic submodule 的当前源码为准，官方 Codex manual 用于校准用户可见语义。

## 核心结论

Goal 模式不是一个更长的 prompt，也不是 `Plan` / `Default` 这种 collaboration mode。它是一套挂在线程生命周期上的持久目标扩展：

1. **目标被持久化为 thread metadata**：源码用独立的 `goals_1.sqlite` 和 `thread_goals` 表保存目标、状态、token budget、已用 token 和已用时间。
2. **active goal 会自动续跑**：当线程 idle 时，goal runtime 读取 active goal，构造隐藏的 continuation steering item，并通过 `try_start_turn_if_idle` 启动下一轮 regular turn。
3. **完成不是靠“最后一句话说完成”**：模型只能通过 `update_goal` 把目标标成 `complete` 或 `blocked`；pause/resume/budget/usage-limit 由用户或系统控制。
4. **“保证完成”是工程约束，不是数学保证**：Codex 不能证明模型一定正确完成任务，但它用持久目标、自动续跑、完成审计 prompt、状态机、预算/错误保护、恢复机制和 UI 可见性，减少长任务半途漂移或草率结束。
5. **Goal 的核心动机是突破单 turn 长闭环的可靠性边界**：普通 `run_turn` 可以在一个 turn 内反复执行“模型采样 -> 工具调用 -> 工具结果回灌 -> 再采样”，但当 step 数非常大时，继续把所有决策压在同一个 turn 的自我循环上，会放大上下文污染、目标漂移、过早总结、恢复困难和不可观察的问题。Goal 模式把长任务改造成跨 turn 的 durable workflow。
6. **长任务优化集中在四处**：跨 turn 续跑、进度记账、状态恢复、目标更新注入。它们都围绕“目标还没达成就继续，真的阻塞或耗尽预算就停住并交代清楚”设计。

一句话概括：普通模式是“用户发一轮，Codex 做到本轮结束”；Goal 模式是“线程有一个可恢复、可记账、可自动续跑的完成条件，Codex 在每轮结束后继续追它，直到状态进入停止态”。

## 为什么需要 Goal 模式

Goal 模式要解决的不是“普通模式完全不能做多步骤任务”。普通 turn 本身已经支持多轮采样和工具调用：模型发出 function/tool call 后，Codex 执行工具并把结果写回 history；只要模型继续请求 follow-up，`run_turn` 就会继续采样。这条普通 turn 机制在 [`core/src/session/turn.rs`](./codex/codex-rs/core/src/session/turn.rs) 里体现为：tool call 会触发后续采样，只有 assistant message 且不再需要 follow-up 时才进入 turn stop。也就是说，非 Goal 模式已经可以完成“工具调用 -> 观察结果 -> 再调用工具”的局部闭环。

真正的问题出现在任务 horizon 很长时：如果一个目标需要几十甚至上百个有效 step，单个 turn 内的闭环会逐渐从“高效自治”变成“脆弱的长跑”。

### 单 turn 长闭环的瓶颈

1. **继续/结束完全贴着当前上下文判断**
   普通 turn 的结束边界主要由模型本次输出决定：继续发工具调用或 `end_turn=false` 就继续；只输出最终 assistant message 就进入停止流程。短任务里这很自然，长任务里则容易出现“阶段性完成感”被误判成“任务完成”。

2. **目标容易被执行细节淹没**
   长任务会产生大量工具结果、错误重试、局部决策、测试输出和用户 steer。即使有 compaction，模型每次判断时看到的是被历史和摘要加工后的上下文；原始目标如果只存在于早期对话，就更容易被稀释或被当前阶段的局部目标替代。

3. **模型的自我循环质量随 step 数下降**
   LLM 可以做多步推理，但持续几十轮“计划、执行、评估、修正、再计划”时，常见风险是重复检查、漏掉已知约束、过早写总结、把临时方案当完成、或在失败后继续消耗资源。这里的限制不只是 token 上限，也是长 horizon agent loop 的稳定性问题。

4. **单 turn 缺少持久、可观察的完成状态**
   普通模式里 assistant 说“完成了”只是对话内容，不是线程级状态。系统无法在后续 idle 边界上知道“这个目标还欠什么”，也无法把 active / paused / blocked / budget_limited / complete 这些状态作为调度条件。

5. **恢复与用户控制边界不够清晰**
   超长任务需要能暂停、恢复、编辑目标、统计预算、在错误或用量限制时停止到明确状态。把这些都塞进一个连续 turn，会让调度、UI、记账和错误恢复都依赖模型在当前上下文里“记得自己该怎么做”。

### Goal 的设计回答

Goal 模式把“继续完成目标”从单个 turn 的模型自律，提升为 thread runtime 的责任：

| 单 turn 长闭环问题 | Goal 模式的回答 |
| --- | --- |
| 模型在某个阶段性节点可能把局部完成误判为全局完成。 | completion 不只是最终话术；必须通过 `update_goal(status="complete")` 改变持久 goal 状态，并受 completion audit prompt 约束。 |
| 原始目标在长 history/compaction 后被稀释。 | objective 持久化为 `ThreadGoal.objective`，每次 continuation 都重新注入完整目标，并要求不要缩小成功条件。 |
| 超多 step 的持续自我循环质量下降。 | 每个 turn 只需要朝真实目标做具体进展；turn 结束后如果 goal 仍 active，runtime 在 idle 边界重新点火。 |
| 系统不知道“还要不要继续”。 | `active` 是明确调度状态；只要目标 active 且线程 idle，goal runtime 就尝试启动 continuation turn。 |
| 错误、预算、用户暂停等边界会混在普通对话里。 | blocked / usage_limited / budget_limited / paused / complete 都是状态机的一部分，调度和 UI 可以直接理解。 |

因此，Goal 模式存在的最重要意义可以概括为：

> 当任务复杂到不能稳定地压在一个 turn 的连续工具闭环里完成时，Goal 模式用持久目标、跨 turn continuation、完成审计和状态机，把“长 horizon agent loop”工程化。

这里的跨 turn 不是为了让 LLM “忘掉前文重新开始”，也不是多 agent 协作；它仍沿用同一个 thread/session lineage。关键是每个 turn 结束后，系统不会把任务自然停止当作完成，而是依据持久 goal 状态决定是否继续，并在下一 turn 用 [`continuation.md`](./codex/codex-rs/ext/goal/templates/goals/continuation.md) 隐藏 steering 重新对齐完整目标、当前预算和完成标准。

## 用户语义

官方手册把 Goal mode 描述为给 Codex 一个跨长任务持续追踪的 persistent objective；goal 文本既是 starting prompt，也是 completion criteria。用户通过 `/goal` 设置目标，app/IDE/CLI 都有入口；目标为空或超过 4000 字符会被拒绝，较长说明建议放到文件里再让 goal 引用它。

从源码看，这个语义落在这些位置：

| 语义 | 源码入口 |
| --- | --- |
| feature gate：`features.goals` | [`features/src/lib.rs`](./codex/codex-rs/features/src/lib.rs) |
| `/goal`、`pause`、`resume`、`clear`、`edit` | [`tui/src/chatwidget/slash_dispatch.rs`](./codex/codex-rs/tui/src/chatwidget/slash_dispatch.rs)、[`tui/src/app/thread_goal_actions.rs`](./codex/codex-rs/tui/src/app/thread_goal_actions.rs) |
| 目标协议对象和 4000 字符限制 | [`protocol/src/protocol.rs`](./codex/codex-rs/protocol/src/protocol.rs) |
| 长目标/粘贴/图片 materialize 成附件文件 | [`tui/src/goal_files.rs`](./codex/codex-rs/tui/src/goal_files.rs) |
| UI 状态行、用量、菜单 | [`tui/src/chatwidget/goal_status.rs`](./codex/codex-rs/tui/src/chatwidget/goal_status.rs)、[`tui/src/chatwidget/goal_menu.rs`](./codex/codex-rs/tui/src/chatwidget/goal_menu.rs) |

需要注意：Goal 模式依赖持久线程状态。临时 ephemeral thread 会拒绝 goal；review subagent 也会禁用/隐藏 goal 工具，避免子任务误接管主线程目标。

## 与非 Goal 模式的对比

| 维度 | 非 Goal 模式 | Goal 模式 |
| --- | --- | --- |
| 触发方式 | 用户消息、slash command、app-server `turn/start` 触发一轮工作。 | 用户显式 `/goal` 或模型在被允许时 `create_goal` 创建目标；目标激活后可自动触发后续 turn。 |
| 状态保存 | 主要保存对话 history、thread metadata、token usage。 | 额外保存 `ThreadGoal`：objective、status、budget、tokens/time used、created/updated time。 |
| 完成条件 | 本轮 assistant 结束输出即 turn complete；是否“任务完成”主要靠用户判断或模型自述。 | 必须满足 objective；模型需要调用 `update_goal(status="complete")` 才进入完成态。 |
| 后续调度 | turn 结束后线程 idle，除 compaction/queued work 等机制外不会继续做同一任务。 | active goal 在 idle 生命周期里调用 `try_start_turn_if_idle` 自动启动 continuation turn。 |
| 模型可见工具 | 普通工具集。 | 额外有 `get_goal`、`create_goal`、`update_goal`。 |
| 用户控制 | 用户继续发消息、steer、interrupt。 | 用户还可 pause/resume/edit/clear goal；follow-up 仍可 steer 当前目标。 |
| 风险保护 | 常规 interrupt、approval、sandbox、usage-limit、context compaction。 | 额外有 goal budget、budget-limited wrap-up steering、turn error -> blocked、usage-limit -> usage_limited。 |
| 长任务记账 | 线程/turn token usage。 | goal 维度累计 token 和 elapsed time，并在完成、预算耗尽等节点上报。 |
| 恢复行为 | resume 后恢复 thread/history，等待用户或已有 queued work。 | resume 后先发 goal snapshot，再让 active goal 在 idle 时继续。 |

非 Goal 模式依然能做复杂任务，尤其配合计划、索引和 compaction；Goal 模式的关键增量是：把“最终目标”从一次对话内容提升成线程级状态，并允许系统在用户不重复提示的情况下持续推进。

## 核心对象与状态机

协议层的 `ThreadGoalStatus` 有六种状态：

| 状态 | 含义 | 谁能进入 |
| --- | --- | --- |
| `active` | 正在追目标；线程 idle 时可自动续跑。 | 用户创建/恢复，模型创建，外部 set。 |
| `paused` | 用户暂停；不自动续跑。 | 用户或外部 set。 |
| `blocked` | 目标确实卡住，需要用户输入或外部状态变化。 | 模型 `update_goal`，或 turn error 后系统置为 blocked。 |
| `usage_limited` | 遇到账户/服务用量限制。 | 系统错误处理。 |
| `budget_limited` | goal token budget 已耗尽或降低预算后已超限。 | 持久化/记账层自动判定。 |
| `complete` | 目标已达成。 | 模型 `update_goal` 或外部 set。 |

状态存在两层限制：

- **模型工具限制**：`update_goal` 的 schema 只允许 `complete` / `blocked`，并在描述中要求 complete 前做证据审计，blocked 前至少连续三轮遇到同一阻塞条件。
- **系统/用户控制限制**：pause/resume/usage-limit/budget-limit 不由模型随意设置；TUI/app-server 通过 `thread/goal/set` 管理，budget-limit 由 usage accounting 自动触发。

`ThreadGoal` 在 state 层还有一个 `goal_id`。更新时可带 `expected_goal_id`，防止旧 turn 或并发外部 mutation 把已经替换的新 goal 覆盖掉。

## 数据持久化

Goal 早期在主 state DB 里有迁移，后续主迁移里删除了旧 `thread_goals` 表；当前运行时用独立 goals DB：

```text
$CODEX_HOME/goals_1.sqlite
  thread_goals(
    thread_id primary key,
    goal_id,
    objective,
    status,
    token_budget,
    tokens_used,
    time_used_seconds,
    created_at_ms,
    updated_at_ms
  )
```

关键入口：

- [`state/src/runtime.rs`](./codex/codex-rs/state/src/runtime.rs)：初始化 state/logs/goals/memories 四个 SQLite DB，`GoalStore` 使用 goals pool。
- [`state/goals_migrations/0001_thread_goals.sql`](./codex/codex-rs/state/goals_migrations/0001_thread_goals.sql)：当前 goals DB 表结构。
- [`state/src/runtime/goals.rs`](./codex/codex-rs/state/src/runtime/goals.rs)：插入、替换、更新、删除、usage accounting、预算超限判定。

这个拆分对长时间运行有现实意义：Goal 的 token/time 记账可能在 tool finish、turn stop、错误处理、外部 set/clear 前频繁写入，独立 DB 能减少与主 thread metadata / logs / memories 的锁竞争。

## 运行链路

### 设置 Goal

```text
用户输入 /goal <objective>
  -> TUI 解析 GoalDraft
  -> 过长目标/粘贴/图片可 materialize 到 $CODEX_HOME/attachments
  -> app-server ThreadGoalSet
  -> GoalService::set_thread_goal
  -> GoalStore 写入 goals_1.sqlite
  -> 发 ThreadGoalUpdated notification
  -> GoalRuntimeHandle::apply_external_goal_set
  -> status=active 时 continue_if_idle
```

这里的“goal 文本作为 starting prompt”不是把 `/goal ...` 当普通 user message 直接送模型，而是 goal 激活后由 runtime 构造 hidden continuation context，让下一轮模型围绕该 objective 开始工作。

### 自动续跑

```text
regular turn 结束
  -> Session 清 active_turn
  -> emit_thread_idle_lifecycle_if_idle
  -> GoalExtension::on_thread_idle
  -> GoalRuntimeHandle::continue_if_idle
  -> 读取 active ThreadGoal
  -> continuation_steering_item(goal)
  -> CodexThread::try_start_turn_if_idle
  -> Session::try_start_turn_if_idle
  -> 启动新的 RegularTask
```

自动续跑不是无条件后台循环。核心 gate 在 [`core/src/session/inject.rs`](./codex/codex-rs/core/src/session/inject.rs)：

- 有用户/client 触发的 pending turn 时拒绝，让用户输入优先。
- 当前有 active turn/task 时拒绝。
- 当前处于 Plan mode 时拒绝。
- 创建 turn context 后会再次检查 Plan mode 和 pending trigger-turn，避免 race。

所以 Goal 模式复用了 session scheduling 的“thread 内串行、idle 才自动工作”原则。它不是开另一个并发 agent 偷偷跑，而是在同一个 thread 的 idle 边界上启动下一轮 regular turn。

### 当前 turn 中目标变化

Goal 支持运行中被用户编辑。`GoalService` 在外部 mutation 前会调用 `prepare_external_goal_mutation` 先结算当前 active goal progress，并用 `goal_state_lock` 包住读写窗口，避免“刚读到旧 goal 准备续跑，用户同时改了目标”的竞态。

如果 active goal 的 objective 被编辑，runtime 会通过 `inject_active_turn_steering` 把 objective-updated steering item 注入正在运行的 turn，提示模型立刻转向新目标，而不是继续做只服务旧目标的工作。

### 完成 Goal

```text
模型判断目标已达成
  -> 按 continuation prompt 做 requirement-by-requirement completion audit
  -> 调 update_goal(status="complete")
  -> update_goal 先结算本 turn 的最终 token/time
  -> GoalStore status=complete
  -> 事件通知 UI
  -> accounting 清理 active goal
  -> 后续 idle 不再自动续跑
```

如果目标带 token budget，`update_goal` 的返回还会给出 structured usage；工具描述要求模型在最终回复里报告预算消耗。

### 阻塞与错误

Goal 模式把几类“无法继续”显式状态化：

- 普通 turn error：系统把 active goal 标成 `blocked`，避免自动续跑在不可恢复错误上循环烧 token。
- usage-limit error：系统把 active/budget-limited goal 标成 `usage_limited`。
- token budget 达到上限：usage accounting 把 goal 标成 `budget_limited`，并向当前 turn 注入 budget-limit steering，要求尽快收尾、总结进展和下一步，不再开始实质新工作。
- 模型主动 blocked：只有在同一阻塞条件连续至少三轮出现且确实无法推进时，才应调用 `update_goal(status="blocked")`。

这也是 Goal 模式“保证完成”的另一面：它不仅推动继续，还要在不能继续时停在可解释状态，而不是无限重试。

## 模型看到什么

Goal 扩展暴露三个 Responses API function tool，定义在 [`ext/goal/src/spec.rs`](./codex/codex-rs/ext/goal/src/spec.rs)，执行逻辑在 [`ext/goal/src/tool.rs`](./codex/codex-rs/ext/goal/src/tool.rs)：

| 工具 | 作用 | 关键约束 |
| --- | --- | --- |
| `get_goal` | 读取当前目标、状态、预算和用量。 | 空参数。 |
| `create_goal` | 显式请求时创建新 active goal。 | 目标不能为空；budget 必须是正数；已有未完成 goal 时失败，除非旧 goal 已 complete。 |
| `update_goal` | 标记已有 goal complete 或 blocked。 | 不能 pause/resume/budget-limit/usage-limit；complete 必须真的达成；blocked 必须满足严格阻塞审计。 |

自动续跑、预算耗尽、目标编辑通过隐藏 steering item 实现，模板在 [`ext/goal/templates/goals/`](./codex/codex-rs/ext/goal/templates/goals)：

- `continuation.md`：保留完整 objective，不把成功条件缩小；从当前 worktree 和外部状态取证；多步骤时维护 plan；complete 前逐项审计 evidence。
- `budget_limit.md`：预算耗尽后不要开始新实质工作，尽快总结进展、剩余工作和下一步。
- `objective_updated.md`：新 objective 替代旧 objective，当前 turn 应转向新目标。

这些模板是 Goal 模式最强的“行为约束”来源：它们让模型每次续跑都重新对齐完整目标，并把“当前证据是否足够证明完成”作为标记 complete 前的必要步骤。

### Goal 特有的 `user` role hidden steering

这里的范围只包括 Goal extension 自己合成、且服务于 Goal runtime 的 `user` role item。普通 session initial context、hook prompt、compaction summary、compaction 保留的用户消息副本也可能以 `role: "user"` 出现在 goal turn 附近，但它们不是 Goal 模式特有机制，不在本节构造示例。

Goal 特有的三类 item 都由 [`ext/goal/src/steering.rs`](./codex/codex-rs/ext/goal/src/steering.rs) 构造，经 `goal_context_input_item` 包装成 `InternalModelContextFragment(source="goal")`。它们的共同点是：raw item 的 role 是 `user`，但内容带 `<codex_internal_context source="goal">`，属于 hidden internal context，不应被理解成真实用户又输入了一句话。

#### 1. Active goal continuation

active goal 在 thread idle 后自动续跑时，runtime 会把持久化的 objective、预算和完成审计要求重新注入下一轮 regular turn。结构大致是：

```json
{
  "type": "message",
  "role": "user",
  "content": [
    {
      "type": "input_text",
      "text": "<codex_internal_context source=\"goal\">\nContinue working toward the active thread goal.\n\n<objective>\n...\n</objective>\n\nBudget:\n- Tokens used: ...\n- Token budget: ...\n- Tokens remaining: ...\n\nCompletion audit:\n...\n</codex_internal_context>"
    }
  ]
}
```

它对应 [`templates/goals/continuation.md`](./codex/codex-rs/ext/goal/templates/goals/continuation.md)：要求不要缩小 objective、从当前状态取证、必要时维护 plan，并在标记 complete 前做 requirement-by-requirement completion audit。

#### 2. Objective updated steering

用户在 active goal 运行中编辑 objective 时，runtime 会向当前 turn 注入新目标提示，让模型停止继续只服务旧 objective 的工作。结构大致是：

```json
{
  "type": "message",
  "role": "user",
  "content": [
    {
      "type": "input_text",
      "text": "<codex_internal_context source=\"goal\">\nThe active thread goal objective was edited by the user.\n\n<untrusted_objective>\n...\n</untrusted_objective>\n\nAdjust the current turn to pursue the updated objective. Avoid continuing work that only served the previous objective unless it also helps the updated objective.\n\nDo not call update_goal unless the updated goal is actually complete.\n</codex_internal_context>"
    }
  ]
}
```

它对应 [`templates/goals/objective_updated.md`](./codex/codex-rs/ext/goal/templates/goals/objective_updated.md)：新 objective 是用户提供的数据，覆盖旧 objective；模型应调整当前 turn，而不是继续惯性执行旧目标。

#### 3. Budget-limit steering

goal token budget 达到上限后，系统会把目标标为 `budget_limited`，并向当前 turn 注入收尾提示。结构大致是：

```json
{
  "type": "message",
  "role": "user",
  "content": [
    {
      "type": "input_text",
      "text": "<codex_internal_context source=\"goal\">\nThe active thread goal has reached its token budget.\n\n<objective>\n...\n</objective>\n\nBudget:\n- Tokens used: ...\n- Token budget: ...\n- Tokens remaining: 0\n\nThe system has marked the goal as budget_limited, so do not start new substantive work for this goal. Wrap up this turn soon: summarize useful progress, identify remaining work or blockers, and leave the user with a clear next step.\n\nDo not call update_goal unless the goal is actually complete.\n</codex_internal_context>"
    }
  ]
}
```

它对应 [`templates/goals/budget_limit.md`](./codex/codex-rs/ext/goal/templates/goals/budget_limit.md)：重点不是继续推进，而是停止新工作、总结进展和剩余问题。若目标其实已经完成，仍允许调用 `update_goal(status="complete")`，否则保持 budget-limited 收尾。

旧历史里还可能看到兼容格式 `<goal_context>...</goal_context>`；当前生成路径使用的是 `<codex_internal_context source="goal">...</codex_internal_context>`。

## 进度记账与预算控制

进度记账由 [`ext/goal/src/accounting.rs`](./codex/codex-rs/ext/goal/src/accounting.rs)、[`ext/goal/src/extension.rs`](./codex/codex-rs/ext/goal/src/extension.rs)、[`state/src/runtime/goals.rs`](./codex/codex-rs/state/src/runtime/goals.rs) 配合完成：

1. turn start 时记录 token usage baseline；Plan mode 的 turn 不参与 goal token accounting。
2. token usage event 更新当前 turn 的累计 usage。
3. tool finish、turn stop、turn abort、turn error、外部 goal mutation 前都会尝试结算 delta。
4. token delta 计算为非 cached input tokens + output tokens，不把 cached input 重复计入。
5. time delta 用 wall-clock baseline 累积；线程 resume 后如果有 active goal，会重新标记 idle active goal 并继续计时。
6. `progress_accounting_lock` 保证并发 tool finish hook 不会重复扣同一段 token/time。
7. usage 写入 state 后，如果 `tokens_used + delta >= token_budget`，自动转为 `budget_limited`。

测试里覆盖了几个关键边界：

- 创建 goal 会重置 baseline，避免把创建前的 turn token 算进 goal。
- 并发 tool finish 只记一次账。
- budget-limited 后，同一 in-flight turn 的后续 usage 仍会继续补记，直到 turn stop 或主动清理 active goal。
- Plan mode turn 的 usage-limit 不会错误停止 active goal。
- stale turn 的 usage-limit 不会停止当前新 turn 的 active goal。

## 长任务专项优化

Goal 模式针对“多步骤、长时间运行”做了几类显式调整。

### 1. 把超长 step loop 拆到多个 turn

非 Goal 模式的 `run_turn` 可以在同一 turn 内反复工具调用，但这种闭环越长，越依赖模型在当前上下文里持续保持目标、判断下一步、识别完成与否。Goal 模式允许一个 turn 在做出实质进展后结束，同时让 active goal 在 idle 边界自动触发下一轮 continuation。

这带来两个效果：

- 单个 turn 不必承载整个长 horizon 的所有计划、执行、评估和收尾压力。
- 每次 continuation 都能重新注入完整 objective、预算和完成审计要求，把下一段工作重新锚回最终目标。

### 2. 跨 turn 目标保持

普通对话越长越容易把“最初到底要完成什么”稀释在历史里。Goal 模式把 objective 放到持久字段里，每次 continuation 都重新注入，并明确要求不缩小成功条件。这让目标不依赖模型记忆或 compact summary 的质量。

### 3. 自动 idle continuation

长任务不需要用户每次说“继续”。只要 goal 仍是 active、线程 idle、没有用户/client pending work、不是 Plan mode，runtime 就会启动下一轮 turn。这个续跑仍受 session 串行调度约束，所以不会和用户 steer 或另一个 turn 并发修改同一 thread state。

### 4. 可暂停、可恢复、可编辑

用户可以 pause/resume/edit/clear。暂停后不会自动续跑；resume 把状态改回 active；编辑目标时 runtime 会在当前 turn 注入 updated-objective steering。TUI 在恢复 paused/blocked/usage-limited goal 后还会提示用户是否 resume。

### 5. 预算和用量透明

Goal 有独立 token budget、tokens used、time used。UI 状态行会显示 active goal 的时间或 token 预算进度；complete/budget-limited 等状态会显示收尾用量。metrics 和 analytics 也记录 created/resumed/completed/budget_limited/usage_limited/blocked、token count 和 duration。

### 6. 失败防循环

普通错误如果直接进入 idle continuation，可能形成“失败 -> 自动继续 -> 再失败”的循环。Goal runtime 在 non-retryable turn error 后把目标置为 blocked，在 usage-limit 后置为 usage_limited，budget 耗尽后置为 budget_limited。这些状态都会停止自动续跑。

### 7. Resume 顺序优化

app-server 在 resume 线程时先发送 token usage / goal snapshot，再触发 thread idle lifecycle。这样客户端先看到恢复后的 goal 状态，之后 active goal 才可能自动继续，避免 UI 和实际运行状态错位。

### 8. 长目标附件化

目标字段上限 4000 字符，但 TUI 可以把过长 objective、pending paste、图片写成 `$CODEX_HOME/attachments/<uuid>/...`，再把 goal objective 替换为短引用。这让 Goal 模式能承载复杂规格，同时保持协议字段短小可验证。

## 如何“保证完成”

源码体现的是一组 layered safeguards：

```text
持久 objective
  + 自动 continuation
  + 模型可见 completion audit
  + update_goal 完成/阻塞工具
  + 状态机停止条件
  + token/time accounting
  + pause/resume/edit/clear 控制面
  + error/budget/usage-limit 防循环
  + resume rehydration
  + UI/metrics 可见性
```

这些机制共同保证的是：

- Codex 不会因为一个 turn 结束就默认放弃 active goal。
- Codex 每轮都会看到同一个 objective 和预算/用量信息。
- Codex 只有在声称 evidence 足够时才应标 complete。
- Codex 遇到真正阻塞、预算耗尽、用量限制或错误时会停到明确状态。
- 用户随时能暂停、恢复、编辑或清理目标。

但它不保证：

- 模型一定正确理解 objective。
- 模型的 completion audit 一定完备。
- 外部环境一定可用，测试一定覆盖全部要求。
- 用户给出的 goal 一定可验证或可完成。

所以更准确的表述是：Goal 模式把“完成复杂目标”从一次回答的自律问题，改造成有持久状态、自动调度、审计提示和停止条件的工程流程。

## 与 session 调度的关系

Goal 模式站在既有 session scheduling 上方。前一份 [`codex-session-scheduling.md`](./codex-session-scheduling.md) 里已经说明：同一 session 同时最多一个 running task，turn 内通过 sampling/tool/pending-input 边界续跑，thread idle 后可触发 extension-initiated idle work。

Goal 利用的正是这个扩展点：

- [`GoalExtension`](./codex/codex-rs/ext/goal/src/extension.rs) 注册为 thread lifecycle、turn lifecycle、tool lifecycle、token usage 和 tool contributor。
- `on_thread_idle` 调 runtime 的 `continue_if_idle`。
- `continue_if_idle` 调 core 的 `try_start_turn_if_idle`。
- core 负责拒绝 busy、Plan mode、pending trigger-turn。

因此 Goal 模式没有引入新的全局 scheduler；它是在每个 thread 的 idle 边界上添加一个“如果 active goal 还没结束，就发起下一轮”的局部续跑策略。

## FAQ：Goal 与 Compaction

### Q1：Compaction 后，Goal objective 是否会作为特殊 developer message 自动重新注入？

不会。Goal extension 安装时注册的是 thread/config/turn/token/tool lifecycle contributor 和 tool contributor，没有注册 `context_contributor` 或 `turn_input_contributor`。因此它不会参与 `build_initial_context_with_world_state(...)` 的 canonical context 重建链路，也不会在 compaction 后额外生成一条“当前处于 Goal 模式 / active goal 是什么”的 developer message。

这意味着 [`InitialContextInjection::BeforeLastUserMessage`](./codex/codex-rs/core/src/compact.rs) 保护的是普通 prompt 系统里的 canonical context：权限、环境、AGENTS.md、skills/plugins、world state、协作模式等。它本身不等价于“重新注入 active goal objective”。

### Q2：Mid-turn compaction 会保留 Goal 的 `InternalModelContextFragment` 吗？

通常不会把原始 Goal internal fragment 当作真实 user message 保留。Goal continuation、objective-updated、budget-limit 都会被包装成 `InternalModelContextFragment(source="goal")`，wire 层 role 是 `user`，但它是 hidden internal context，不是用户自然语言输入。

compaction 安装 replacement history 时会过滤上下文型 user message。local compaction 的 `collect_user_messages(...)` 只收能解析成 `TurnItem::UserMessage` 的真实用户消息；remote compaction 的 `should_keep_compacted_history_item(...)` 也会丢弃 non-user-content 的 user wrapper，只保留真实 user message、hook prompt、assistant/compaction item 等。这个设计避免旧权限、旧环境、旧 hidden context 被重复或过期地带入新窗口。

### Q3：压缩后如果没有“真实用户消息”，第一条 `user` role message 是什么？

不一定是用户手写消息。Codex 的 Responses history 里 `role: "user"` 会承载多类内容：真实用户输入、contextual user fragment、hook prompt、compaction summary。mid-turn compaction 会把 canonical initial context 插到最后真实 user message 之前；如果没有真实 user message，则插到 compaction summary 或 compaction item 之前，让 summary/compaction item 保持模型期望的位置。

所以“压缩后第一条 user role message”可能是重新注入的 contextual user context，也可能是 compaction summary，而不是一个真实用户消息。这是 prompt 系统的 typed-history 设计，不表示用户真的又输入了一轮。

### Q4：如果 summary 没有写入 goal，LLM 是否会在 mid-turn compaction 后丢失目标？

要区分两种“丢失”。

系统状态不会丢。Goal objective/status/budget/usage 存在独立的 `thread_goals` 持久状态里，不依赖 transcript，也不会因为 `replace_compacted_history(...)` 被删除。`GoalRuntime`、`GoalService` 和 `get_goal` 工具都能重新读取它。

模型当前上下文可能短暂丢。排除下一轮 continuation steering 后，mid-turn compaction 并没有看到一条专门的 active-goal developer message 会自动补回 objective。模型能否在同一 turn 的 compact 后 follow-up 中继续知道目标，主要依赖 compaction summary 是否把目标语义写进去，或者依赖后续工具/事件重新暴露目标。

因此，Goal 的强保证在 runtime state 层；prompt 层对 mid-turn compaction 的目标保持是“summary 应该保留重要任务语义”的软约束，而不是独立的 hard reinjection。

### Q5：是否有 prompt 提醒 LLM“当前处于 Goal 模式”，从而引导它调用 `get_goal`？

没有看到通用 developer prompt 明确写“当前处于 Goal 模式”。模型能看到的间接提示主要来自工具集合：当 goals feature 启用、线程有持久状态且不是 review subagent 时，Goal extension 会暴露 `get_goal`、`create_goal`、`update_goal` 三个工具；`get_goal` 的 tool description 明确说明可以读取当前 thread goal、状态、预算和用量。

这条路径是恢复能力，不是自动提示。也就是说，compaction 后模型如果意识到需要恢复目标，可以调用 `get_goal`；但如果当前 prompt 没有让它意识到“我正在追一个 active goal”，工具描述本身不一定足以让它主动恢复 objective。

### Q6：除 continuation 外，还有哪些路径会重新把 goal objective 暴露给模型？

有几条有限路径：

- 用户运行中编辑 objective 时，runtime 会注入 objective-updated steering，内容包含新 objective。
- token budget 达到上限后，tool lifecycle 会注入 budget-limit steering，内容包含 objective 和预算状态，要求收尾。
- 模型主动调用 `get_goal` 时，工具输出会返回结构化 `goal`。
- `create_goal` / `update_goal` 的工具输出也会返回当前 goal 状态；如果这些输出发生在 compaction 前，summary 也可能把它们吸收进任务摘要。

这些路径都不是“每次 compaction 后无条件补回 objective”的机制。更准确地说，Goal 模式通过持久状态保证系统不忘目标，通过 continuation/事件 steering 和 `get_goal` 工具让模型在需要时重新获得目标，通过 compaction summary 在同一窗口内尽量延续目标语义。

## 设计取舍

### 优点

- **目标不丢**：objective 不依赖上下文剩余量或对话摘要。
- **自动推进**：适合迁移、修复、批量分析、长测试闭环等多 turn 任务。
- **可控停止**：blocked / usage_limited / budget_limited / paused / complete 都有明确含义。
- **可观察**：UI、notifications、metrics、analytics 都能看到 goal 状态和用量。
- **竞态处理细**：`goal_state_lock`、`progress_accounting_lock`、`expected_goal_id` 都在避免长任务常见 race。

### 代价与限制

- **实现复杂度高**：Goal 横跨 TUI、app-server、core session、extension API、state DB、prompt templates、analytics。
- **完成判定仍依赖模型**：工具和 prompt 只能约束，不是形式化 verifier。
- **只适合持久线程**：ephemeral thread 不支持。
- **可能消耗更多资源**：active goal 会在 idle 时继续跑，需要 budget 和 pause 控制。
- **目标质量很关键**：模糊 goal 会让 completion audit 难以判断；明确 deliverable、验证命令和停止条件更适合 Goal 模式。

## 源码阅读地图

| 模块 | 作用 |
| --- | --- |
| [`ext/goal/src/lib.rs`](./codex/codex-rs/ext/goal/src/lib.rs) | Goal extension crate 出口。 |
| [`ext/goal/src/extension.rs`](./codex/codex-rs/ext/goal/src/extension.rs) | 注册生命周期 contributor、goal tools、token/tool/turn hooks。 |
| [`ext/goal/src/runtime.rs`](./codex/codex-rs/ext/goal/src/runtime.rs) | active goal runtime、idle continuation、resume restore、外部 mutation effect。 |
| [`ext/goal/src/tool.rs`](./codex/codex-rs/ext/goal/src/tool.rs) | `get_goal` / `create_goal` / `update_goal` 执行逻辑。 |
| [`ext/goal/src/spec.rs`](./codex/codex-rs/ext/goal/src/spec.rs) | 模型可见工具 schema 与行为说明。 |
| [`ext/goal/src/steering.rs`](./codex/codex-rs/ext/goal/src/steering.rs) | continuation / budget-limit / objective-updated hidden steering item。 |
| [`ext/goal/src/accounting.rs`](./codex/codex-rs/ext/goal/src/accounting.rs) | token/time delta 记账、并发防重。 |
| [`ext/goal/src/api.rs`](./codex/codex-rs/ext/goal/src/api.rs) | app-server/UI 外部控制面 `GoalService`。 |
| [`state/src/runtime/goals.rs`](./codex/codex-rs/state/src/runtime/goals.rs) | goal persistence 与 budget 状态转换。 |
| [`app-server/src/request_processors/thread_goal_processor.rs`](./codex/codex-rs/app-server/src/request_processors/thread_goal_processor.rs) | JSON-RPC `thread/goal/*` 请求、resume snapshot、notification ordering。 |
| [`core/src/session/inject.rs`](./codex/codex-rs/core/src/session/inject.rs) | `try_start_turn_if_idle` 自动 idle work gate。 |
| [`core/src/tasks/lifecycle.rs`](./codex/codex-rs/core/src/tasks/lifecycle.rs) | turn stop 后触发 thread idle lifecycle。 |
| [`tui/src/app/thread_goal_actions.rs`](./codex/codex-rs/tui/src/app/thread_goal_actions.rs) | TUI goal 设置、替换确认、状态更新、错误提示。 |
| [`tui/src/goal_files.rs`](./codex/codex-rs/tui/src/goal_files.rs) | 长 objective / paste / image 附件化。 |
| [`ext/goal/tests/goal_extension_backend.rs`](./codex/codex-rs/ext/goal/tests/goal_extension_backend.rs) | goal 工具、记账、预算、错误、resume、service 行为测试。 |

## 结论

Goal 模式本质上是 Codex 为“明确但漫长的任务”加的一层 durable objective runtime。它不改变模型本身，也不绕开普通 session 串行调度；它把目标、进度、预算、停止状态和自动续跑都变成可持久化、可观察、可恢复的系统对象。

这也是它和普通模式最大的区别：普通模式关注“下一轮怎么回答”，Goal 模式关注“这个线程最终要证明什么已经完成，并且在证明前持续推进”。
