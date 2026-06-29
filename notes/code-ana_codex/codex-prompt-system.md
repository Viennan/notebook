# Codex Prompt 系统与 Compaction 策略分析

本文分析 `openai/codex` 当前源码中的 prompt 系统：prompt 有哪些种类、分别承担什么用途、如何组织成一次模型请求，以及 compaction 如何触发、如何改写 history、相关 prompt 模板如何组织。分析以本 topic submodule 的当前源码为准；官方 Codex manual 仅用于校准用户可见的 thread/session 语境。

## 核心结论

Codex 的 prompt 系统不是一个大号 markdown prompt，而是一套围绕 Responses API 组织的结构化输入系统：

1. **基础指令、上下文、用户输入、工具 spec 是分层装配的**：base/model instructions 进入 Responses `instructions` 字段；权限、协作模式、环境、AGENTS.md、skills/plugins 等作为 `developer` 或 `user` role 的 `ResponseItem::Message` 写入 history；工具能力作为顶层 `tools` 或 Responses Lite 下的 `AdditionalTools`。
2. **prompt 模板分散在多个 crate 中**：[`prompts`](./codex/codex-rs/prompts) 是共享模板库，提供 compaction、permissions、goals、review、realtime、apply_patch 等模板；[`protocol`](./codex/codex-rs/protocol) 定义 base instructions 和 Responses item；[`core`](./codex/codex-rs/core) 负责按 session/turn 状态装配；`ext`、`memories`、`collaboration-mode-templates` 继续提供扩展 prompt。
3. **普通 turn 的 prompt 是 history 的当前视图**：[`run_turn`](./codex/codex-rs/core/src/session/turn.rs) 每次 sampling 前克隆 [`ContextManager`](./codex/codex-rs/core/src/context_manager/history.rs)，调用 `for_prompt(...)` 标准化 call/output、图片能力和历史项，再由 `build_prompt(...)` 加上 tool router 与 base instructions。
4. **上下文注入采用“完整注入 + 后续 diff”的策略**：第一次 turn 或 reference context 丢失时，Codex 注入完整权限、协作模式、环境、AGENTS.md、skills/plugins 等上下文；后续 turn 只追加 model switch、permissions、collaboration mode、realtime、personality、world state 等变化。
5. **ContextualUserFragment 是 prompt 片段的统一抽象**：每个片段声明 role、markers 和 body，渲染后被合并成 developer/user message。XML-like marker 让系统能识别哪些 message 是上下文注入项，从而支持 rollback、diff、compaction 后重建。
6. **Compaction 不是简单截断，而是替换 history 窗口**：local compaction 用客户端模板让模型生成 handoff summary；remote compaction 让服务端返回 compacted history；remote v2 在普通 Responses stream 中追加 `compaction_trigger` 并期望一个 `compaction` item。三者最终都会调用 `replace_compacted_history(...)` 安装新窗口。
7. **Compaction 会重新注入 canonical context，而不是完全相信摘要**：pre-turn/manual compaction 通常清空 reference context，让下一轮重新注入；mid-turn compaction 会把当前初始上下文插回 replacement history 的模型期望位置，保证继续采样时仍看到当前权限、环境和指令。
8. **Prompt cache 设计与 prompt 组织强相关**：Codex 尽量让同一 thread 的稳定前缀不被重写；上下文变化作为追加 diff；compaction 则显式开启新 window。Responses API 与 prompt cache 的细节见 [`codex-responses-api.md`](./codex-responses-api.md)。

一句话概括：Codex 把 prompt 设计成 **typed history + stable instructions + contextual fragments + tool specs + compaction checkpoint**，核心目标是在长会话里保持模型可见状态准确、可恢复、可缓存。

## 源码地图

| 主题 | 关键文件 | 作用 |
| --- | --- | --- |
| 默认 base instructions | [`protocol/src/prompts/base_instructions/default.md`](./codex/codex-rs/protocol/src/prompts/base_instructions/default.md) | Codex CLI 默认基础行为说明。 |
| BaseInstructions 类型 | [`protocol/src/models.rs`](./codex/codex-rs/protocol/src/models.rs) | `BaseInstructions`、`ResponseItem`、`CompactionTrigger`、`Compaction` 等 wire item。 |
| 模型指令与 personality | [`protocol/src/openai_models.rs`](./codex/codex-rs/protocol/src/openai_models.rs)、[`core/templates/model_instructions`](./codex/codex-rs/core/templates/model_instructions)、[`core/templates/personalities`](./codex/codex-rs/core/templates/personalities) | `ModelInfo::get_model_instructions(...)` 按 model messages/personality 生成 base instructions。 |
| Prompt 抽象 | [`core/src/client_common.rs`](./codex/codex-rs/core/src/client_common.rs) | `Prompt { input, tools, parallel_tool_calls, base_instructions, output_schema }`。 |
| 普通 turn 组装 | [`core/src/session/turn.rs`](./codex/codex-rs/core/src/session/turn.rs) | `run_turn`、`build_skills_and_plugins`、`build_prompt`、pre/mid-turn auto compact。 |
| 初始上下文 | [`core/src/session/mod.rs`](./codex/codex-rs/core/src/session/mod.rs) | `build_initial_context_with_world_state_and_mcp`、`record_context_updates_and_set_reference_context_item`、`replace_compacted_history`。 |
| Context fragments | [`context-fragments/src/fragment.rs`](./codex/codex-rs/context-fragments/src/fragment.rs)、[`core/src/context`](./codex/codex-rs/core/src/context) | developer/user prompt 片段抽象与具体实现。 |
| Context updates | [`core/src/context_manager/updates.rs`](./codex/codex-rs/core/src/context_manager/updates.rs) | 将 fragment 合并成 message，生成 settings diff。 |
| History 管理 | [`core/src/context_manager/history.rs`](./codex/codex-rs/core/src/context_manager/history.rs) | 记录、规范化、估算 token、剥离图片、rollback 与 compaction 后替换。 |
| Responses request | [`core/src/client.rs`](./codex/codex-rs/core/src/client.rs) | 将 `Prompt` 映射为 Responses request；处理 Responses Lite、prompt cache key、compact endpoint。 |
| 共享 prompt crate | [`prompts/src/lib.rs`](./codex/codex-rs/prompts/src/lib.rs) | 导出 compact、permissions、goals、review、realtime、apply_patch 模板。 |
| Compaction 本地模板 | [`prompts/templates/compact`](./codex/codex-rs/prompts/templates/compact) | `prompt.md` 与 `summary_prefix.md`。 |
| Local compaction | [`core/src/compact.rs`](./codex/codex-rs/core/src/compact.rs) | 用 summarization prompt 运行一次 Responses 请求，生成 summary 并替换 history。 |
| Remote compaction | [`core/src/compact_remote.rs`](./codex/codex-rs/core/src/compact_remote.rs) | 调 `/responses/compact`，处理服务端返回的 compacted history。 |
| Remote compaction v2 | [`core/src/compact_remote_v2.rs`](./codex/codex-rs/core/src/compact_remote_v2.rs) | 在普通 Responses stream 中追加 `CompactionTrigger`，接收 `Compaction` item。 |
| Token budget compaction | [`core/src/compact_token_budget.rs`](./codex/codex-rs/core/src/compact_token_budget.rs) | 不做摘要，直接开启新 context window。 |
| Manual compact task | [`core/src/tasks/compact.rs`](./codex/codex-rs/core/src/tasks/compact.rs) | `/compact` 或 `Op::Compact` 的任务入口，选择 local/remote/v2/token-budget 实现。 |

## Prompt 分类

### 1. Base / Model Instructions

Base instructions 是最高稳定性的模型行为层。协议层默认值来自 [`BASE_INSTRUCTIONS_DEFAULT`](./codex/codex-rs/protocol/src/models.rs)，文件是 [`default.md`](./codex/codex-rs/protocol/src/prompts/base_instructions/default.md)。它描述 Codex CLI 的身份、工作方式、AGENTS.md 规则、计划工具、编辑约束、验证策略和最终回答格式。

实际运行时的 base instructions 可以被多处覆盖或重写：

| 来源 | 作用 |
| --- | --- |
| model catalog 的 `base_instructions` | 每个模型的默认基础指令。 |
| `ModelMessages.instructions_template` | 新模型可用模板化 instructions 替代旧 `base_instructions`。 |
| personality 模板 | `{{ personality }}` 占位符被 friendly/pragmatic/none 对应文本替换。 |
| config `instructions` / `model_instructions_file` | 用户或环境可覆盖 base instructions。 |
| model switch update | 若 turn 切到不同模型，Codex 会追加 `<model_switch>` developer message，避免新模型看不到自己的模型指令。 |

[`ModelInfo::get_model_instructions`](./codex/codex-rs/protocol/src/openai_models.rs) 是核心函数：如果存在 `instructions_template`，就用 personality message 替换 `{{ personality }}`；否则回退到 `base_instructions`。这使 Codex 能把“模型专属行为”和“session 中动态上下文”分开。

### 2. Developer Context Fragments

Developer fragments 是 Codex 用来告诉模型“当前运行边界和工具/模式状态”的主力层。它们通常 role 为 `developer`，由 [`ContextualUserFragment`](./codex/codex-rs/context-fragments/src/fragment.rs) 渲染。

常见 developer prompt 包括：

| Prompt | 来源 | 用途 |
| --- | --- | --- |
| permissions instructions | [`prompts/src/permissions_instructions.rs`](./codex/codex-rs/prompts/src/permissions_instructions.rs) | 描述 sandbox mode、network access、approval policy、可写根、denied reads、已批准命令前缀。 |
| collaboration mode | [`core/src/context/collaboration_mode_instructions.rs`](./codex/codex-rs/core/src/context/collaboration_mode_instructions.rs)、[`collaboration-mode-templates`](./codex/codex-rs/collaboration-mode-templates/templates) | 注入 Default/Plan/Execute/Pair Programming 等协作模式的 developer instructions。 |
| realtime start/end | [`core/src/context/realtime_*`](./codex/codex-rs/core/src/context)、[`prompts/templates/realtime`](./codex/codex-rs/prompts/templates/realtime) | 告诉模型 realtime 状态变化。 |
| personality fallback | [`core/src/context/personality_spec_instructions.rs`](./codex/codex-rs/core/src/context/personality_spec_instructions.rs) | 当 personality 没 baked 进 base instructions 时追加 developer message。 |
| apps/connectors | [`core/src/context/apps_instructions.rs`](./codex/codex-rs/core/src/context/apps_instructions.rs) | 描述可用 app connector 能力。 |
| skills/plugins | [`core/src/context/available_skills_instructions.rs`](./codex/codex-rs/core/src/context/available_skills_instructions.rs)、[`available_plugins_instructions.rs`](./codex/codex-rs/core/src/context/available_plugins_instructions.rs)、[`plugin_instructions.rs`](./codex/codex-rs/core/src/context/plugin_instructions.rs) | thread-level 可用 skill/plugin 与 turn-level 显式注入。 |
| token budget | [`core/src/context/token_budget_context.rs`](./codex/codex-rs/core/src/context/token_budget_context.rs) | 记录 window id、budget guidance、remaining reminders。 |
| multi-agent mode | [`core/src/context/multi_agent_mode_instructions.rs`](./codex/codex-rs/core/src/context/multi_agent_mode_instructions.rs) | 告诉模型是否可主动/需显式请求再 spawn subagent。 |
| guardian/review reminders | [`core/src/context/guardian_followup_review_reminder.rs`](./codex/codex-rs/core/src/context/guardian_followup_review_reminder.rs) | 审查场景的 follow-up 指令。 |
| hooks/additional context | [`core/src/context/hook_additional_context.rs`](./codex/codex-rs/core/src/context/hook_additional_context.rs) | lifecycle hooks 注入额外上下文。 |

permissions prompt 值得单独看：它不把 permission profile 原样 dump 给模型，而是按 `FileSystemSandboxPolicy` 推导 `danger-full-access` / `workspace-write` / `read-only`，再按 `AskForApproval` 渲染 approval 模板。这让模型可见的是“该如何请求权限/哪些动作可直接做”的行为规则，而不是底层 policy 对象。

#### 追加型 developer message

`developer` role message 不只用于 thread 开头的完整上下文，也经常作为运行时“高优先级上下文补丁”追加到 history。它的目的通常是更新模型的操作边界或补充系统事实，而不是表达用户的新需求；也不是修改 request 顶层的 base instructions。

常见追加场景包括：

| 场景 | 触发 | 追加内容 |
| --- | --- | --- |
| 权限/沙箱状态变化 | sandbox mode、approval policy、permission profile 变化 | 新的 `<permissions instructions>`，描述当前文件系统/网络边界、approval 规则、可写根和 denied reads。 |
| 协作模式变化 | Default/Plan/Execute/Pair Programming 等 mode 切换 | 新的 `<collaboration_mode>...</collaboration_mode>` developer instructions。 |
| 模型切换 | 上一 turn 与当前 turn 使用不同模型 | `<model_switch>`，提示模型按新模型指令继续历史对话。 |
| multi-agent mode 变化 | explicit-request-only/proactive/none 状态变化 | 新的 multi-agent mode developer message，告诉模型是否可主动使用 subagent。 |
| realtime 开关变化 | realtime 从 active 到 inactive，或反向变化 | realtime start/end developer message。 |
| personality 变化 | personality feature 启用且当前模型没有把 personality baked 进 base instructions | personality spec developer message。 |
| 命令前缀被批准 | 用户批准并保存 command prefix | `Approved command prefix saved: ...`，供后续命令审批判断使用。 |
| 网络规则被保存 | network approval 保存 allow/deny rule | `Allowed/Denied network rule saved in execpolicy ...`。 |
| 当前时间提醒 | 采样前需要刷新时间事实 | `It is ... UTC.` 这类无 marker 的 developer message。 |
| context window / token budget | token-budget feature、window guidance、remaining reminder | `<context_window>`、guidance 或 remaining budget developer message。 |
| compaction 后续跑 | mid-turn compaction 使用 `InitialContextInjection::BeforeLastUserMessage` | 将当前 canonical initial context 插回 replacement history，其中可能包含 permissions、collaboration、token budget、apps/skills 等 developer messages。 |

这些追加信息通常通过 [`build_settings_update_items`](./codex/codex-rs/core/src/context_manager/updates.rs)、`ContextualUserFragment::into(...)` 或相关 runtime helper 形成新的 `ResponseItem::Message { role: "developer" }`。设计重点是 append-only：Codex 很少回头改旧 prompt，而是追加一条新的 developer message 来覆盖或补充模型应遵循的当前状态。

### 3. User Context Fragments

User context fragments role 为 `user`，但它们不是用户手写的普通消息，而是模型应该当作上下文读取的信息。典型例子：

| Prompt | 来源 | 用途 |
| --- | --- | --- |
| AGENTS.md instructions | [`core/src/context/user_instructions.rs`](./codex/codex-rs/core/src/context/user_instructions.rs)、[`core/src/context/world_state/agents_md.rs`](./codex/codex-rs/core/src/context/world_state/agents_md.rs) | 将适用目录的 AGENTS.md 指令作为 `# AGENTS.md instructions ... <INSTRUCTIONS>` 注入。 |
| environment context | [`core/src/context/world_state/environment.rs`](./codex/codex-rs/core/src/context/world_state/environment.rs)、[`environment_context.rs`](./codex/codex-rs/core/src/context/environment_context.rs) | cwd、shell、current date、timezone、network、filesystem、subagent 环境等。 |
| recommended plugins | [`core/src/context/recommended_plugins_instructions.rs`](./codex/codex-rs/core/src/context/recommended_plugins_instructions.rs) | 根据用户提及和可用插件给出推荐。 |
| user shell command | [`core/src/context/user_shell_command.rs`](./codex/codex-rs/core/src/context/user_shell_command.rs) | 用户直接执行 shell command 后，把结果作为上下文传给模型。 |
| turn aborted / inter-agent completion | [`turn_aborted.rs`](./codex/codex-rs/core/src/context/turn_aborted.rs)、[`inter_agent_completion_message.rs`](./codex/codex-rs/core/src/context/inter_agent_completion_message.rs) | 描述 turn 被中断或 subagent 完成的事实。 |

这些 user context 的共同特点是：它们改变模型对当前任务环境的理解，但不应该被等同于用户最新自然语言需求。Codex 通过 marker 和 event mapping 区分它们，以便 rollback、compaction、diff 时能识别“这是一段系统注入的上下文”。

### 4. User Turn Input 与 History

用户真实输入会被转成 `ResponseInputItem` / `ResponseItem::Message { role: "user" }` 记录到 history。后续 assistant message、reasoning、tool call、tool output、web/image generation call 等也都是 [`ResponseItem`](./codex/codex-rs/protocol/src/models.rs)。

这意味着 Codex 的 prompt history 不是“messages + 字符串工具结果”，而是类型化的事件日志。`ContextManager::record_items` 只保留 API message；`for_prompt` 发送前会：

- 补齐缺失的 call output。
- 删除孤儿 output。
- 对不支持图片的模型剥离图片。
- 保留 reasoning encrypted content、tool calls、compaction items 等 Responses-native item。

### 5. Tool Prompt / Tool Specs

工具并不主要靠 prompt 文本暴露，而是通过 Responses API 的 tool specs：

```text
ToolRouter
  -> model_visible_specs()
  -> Prompt.tools
  -> create_tools_json_for_responses_api(...)
  -> Responses request tools
```

普通 Responses 模式下，工具放在顶层 `tools` 字段，`parallel_tool_calls` 由模型能力控制。Responses Lite 下，Codex 把工具包装成 `ResponseItem::AdditionalTools { role: "developer" }` 插入 `input` 最前面，顶层 `tools` 为空，并禁用 parallel tool calls。这样 Lite 后端仍能在 input 中看到可用工具。

另有少量工具 prompt 以模板存在：

- [`prompts/templates/apply_patch_tool_instructions.md`](./codex/codex-rs/prompts/templates/apply_patch_tool_instructions.md)：给不支持原生 apply_patch tool 的模型补详细使用说明。
- [`core/templates/search_tool`](./codex/codex-rs/core/templates/search_tool)：tool search 与插件安装请求说明。
- [`ext/web-search/web_run_description.md`](./codex/codex-rs/ext/web-search/web_run_description.md)、[`ext/image-generation/imagegen_description.md`](./codex/codex-rs/ext/image-generation/imagegen_description.md)：扩展工具描述。

### 6. Special Task / Mode Prompts

除普通 turn 外，Codex 还有一组面向特殊任务的 prompt：

| 类型 | 文件 | 用途 |
| --- | --- | --- |
| Goal | [`prompts/templates/goals`](./codex/codex-rs/prompts/templates/goals)、[`ext/goal/templates/goals`](./codex/codex-rs/ext/goal/templates/goals) | active goal 续跑、预算耗尽收尾、objective 更新后的 hidden steering。 |
| Review | [`prompts/templates/review/rubric.md`](./codex/codex-rs/prompts/templates/review/rubric.md)、[`prompts/src/review_request.rs`](./codex/codex-rs/prompts/src/review_request.rs) | code review rubric 与 review target prompt。 |
| Realtime | [`prompts/templates/realtime`](./codex/codex-rs/prompts/templates/realtime) | realtime backend prompt、start/end instructions。 |
| Orchestrator/subagent | [`core/templates/agents/orchestrator.md`](./codex/codex-rs/core/templates/agents/orchestrator.md) | 多 agent/orchestrator 场景指令。 |
| Collaboration templates | [`collaboration-mode-templates/templates`](./codex/codex-rs/collaboration-mode-templates/templates) | Default/Plan/Execute/Pair Programming 模式的 developer instructions。 |
| Memories | [`ext/memories/src/prompts.rs`](./codex/codex-rs/ext/memories/src/prompts.rs)、[`memories/write/src/prompts.rs`](./codex/codex-rs/memories/write/src/prompts.rs)、[`memories/write/templates/memories`](./codex/codex-rs/memories/write/templates/memories) | memory read/write/consolidation。 |
| Guardian | [`core/src/guardian/prompt.rs`](./codex/codex-rs/core/src/guardian/prompt.rs) | guardian review session 的策略 prompt。 |

这些 prompt 通常不直接进入每次普通 turn，而是由对应 task、extension、tool 或 session source 在特定场景下注入。

## 普通 Turn 的 Prompt 组装链路

### 1. Session 初始化解析 base instructions

session configuration 中保存 `base_instructions` 与 `developer_instructions`。base instructions 可以来自模型信息、配置文件、命令行覆盖或 model instructions file。`Session::get_base_instructions()` 返回 [`BaseInstructions`](./codex/codex-rs/protocol/src/models.rs)，后续每次 sampling 都复用。

这里有一个重要设计：base instructions 不是被写入 history 的普通 message。普通 Responses 模式下它作为 request 的 top-level `instructions` 发送；只有 Responses Lite 会把它前插为 developer message。

### 2. 初始上下文构造

[`build_initial_context_with_world_state_and_mcp`](./codex/codex-rs/core/src/session/mod.rs) 是上下文 prompt 的中心汇聚点。它先准备三类容器：

- `developer_sections`：大部分 developer fragment 聚合到一个 developer message。
- `contextual_user_sections`：环境、AGENTS.md、推荐插件等 user fragment 聚合到一个 user message。
- `separate_developer_sections`：需要隔离为独立 developer message 的片段。

它按顺序加入：

1. model switch message。
2. permissions instructions。
3. session developer instructions，guardian reviewer 时可隔离为单独 developer message。
4. collaboration mode instructions。
5. realtime 状态。
6. personality spec fallback。
7. apps/connectors instructions。
8. available skills。
9. recommended plugins、available plugins。
10. extension thread context 与 turn context。
11. token budget context。
12. world state full render，包括 environment 与 AGENTS.md。
13. multi-agent usage hint 与 mode instructions。

最后用 [`build_developer_update_item`](./codex/codex-rs/core/src/context_manager/updates.rs) 和 `build_contextual_user_message` 把文本片段变成 `ResponseItem::Message`。每个 message 的 `content` 是多个 `ContentItem::InputText`，而不是预先拼成一个大字符串。

### 3. ContextualUserFragment 的结构

所有上下文片段共享同一接口：

```text
ContextualUserFragment
  role() -> "developer" | "user"
  markers() -> start/end marker
  body() -> fragment body
  render() -> start + body + end
```

例如 permissions fragment 的 marker 是 `<permissions instructions>` / `</permissions instructions>`；AGENTS.md fragment 的 marker 是 `# AGENTS.md instructions` / `</INSTRUCTIONS>`；environment context 有自己的 XML-like marker。marker 有两个作用：

- 让模型看到清晰边界，降低上下文混淆。
- 让 runtime 后续能识别这些注入项，支持 diff、rollback、history trim 和 compaction filter。

### 4. 首轮完整注入，后续差异注入

[`record_context_updates_and_set_reference_context_item`](./codex/codex-rs/core/src/session/mod.rs) 是普通 turn 进入模型前的上下文落盘点：

```text
reference_context_item 不存在
  -> build_initial_context_with_world_state_and_mcp(...)
  -> set_world_state_baseline(full snapshot)
  -> 写入完整 context items

reference_context_item 存在
  -> build_settings_update_items(...)
  -> history.update_world_state(...)
  -> 写入 settings/world-state diff
```

[`build_settings_update_items`](./codex/codex-rs/core/src/context_manager/updates.rs) 只覆盖一部分高价值变化：model switch、permissions、collaboration mode、multi-agent mode、realtime、personality。world state 则通过 `WorldState::render_history_diff` 做 section-level diff。

这个设计同时服务两件事：

- 避免每轮重复塞完整环境/AGENTS.md/skills 导致上下文膨胀。
- 尽量让 prompt 前缀稳定，提升 Responses prompt cache 命中。

### 5. Turn-level 注入

`run_turn` 记录初始上下文后，还会调用 [`build_skills_and_plugins`](./codex/codex-rs/core/src/session/turn.rs) 根据当前用户输入里的显式 skill/plugin/app mention 注入 turn-scoped guidance。这些注入只针对本 turn，不一定成为 thread 初始 context 的一部分。

随后 hooks 检查输入；用户输入、hook additional contexts、skill/plugin injection items 都写入 history。真正 sampling 前，`run_turn` 才克隆 history 并调用 `for_prompt(...)`。

### 6. Prompt 到 Responses request

[`build_prompt`](./codex/codex-rs/core/src/session/turn.rs) 得到的结构是：

```text
Prompt {
  input: Vec<ResponseItem>,
  tools: Vec<ToolSpec>,
  parallel_tool_calls,
  base_instructions,
  output_schema,
  output_schema_strict,
}
```

[`ModelClient::build_responses_request`](./codex/codex-rs/core/src/client.rs) 再把它映射到 Responses API：

| 模式 | `instructions` | `input` | `tools` | `parallel_tool_calls` |
| --- | --- | --- | --- | --- |
| 普通 Responses | `base_instructions.text` | history items | tool specs | model 支持则开启 |
| Responses Lite | 空字符串 | `AdditionalTools` + base instructions developer message + history items | `None` | false |

这就是 Codex prompt 的最终结构：base instructions、history input、tool specs 是 Responses request 的不同字段；只有在 Lite 模式下，base/tool specs 被折叠进 `input` 前缀。

## Prompt 组织的设计要点

### 1. Prompt 是 append-only history，而不是每次重拼全文

大多数上下文变化都通过追加 message 表达。即使权限或协作模式变化，Codex 也会写入新的 developer update，而不是回头改旧消息。这对流式 resume、rollback、prompt cache、WebSocket incremental request 都更友好。

### 2. Stable prefix 与 dynamic tail 分离

相对稳定的内容：

- base instructions。
- thread 起始 context。
- AGENTS.md / environment full snapshot。
- tool specs 中不变的部分。

动态内容：

- 用户 turn 输入。
- 工具 call/output。
- permission/mode/time/world-state diff。
- current time reminder、token budget reminder。
- skill/plugin 显式注入。

这种分离让同一 thread 在多轮请求中保持较长相同前缀，后续变化主要发生在尾部。

### 3. Runtime 不把所有安全边界交给 prompt

permissions instructions 会告诉模型 sandbox/approval 规则，但真实约束仍在 exec policy、filesystem sandbox、approval reviewer、tool handler 中执行。prompt 的作用是让模型“知道如何行动”，不是作为唯一安全机制。这一点与 sandbox 专题报告中的结论一致，见 [`codex-sandbox-approvals.md`](./codex-sandbox-approvals.md)。

### 4. Model-visible 与 non-model-visible 分离

`client_metadata`、headers、turn-state sticky routing、window id 等不是 prompt 内容；它们走 Responses 控制面。相反，permissions/environment/AGENTS.md 是明确 model-visible 的 `ResponseItem`。这种分离避免模型读取不该作为任务事实的传输元数据。

## Compaction 策略

Codex 的 compaction 目标不是让 history “变短一点”，而是在 context window 压力下创建一个新的可继续窗口。它需要同时保留：

- 用户真实需求和最近用户消息。
- 任务进度 summary。
- 当前 canonical context：权限、环境、AGENTS.md、协作模式等。
- Responses-native 状态：compaction item、window id、token usage、rollout trace。

### 触发条件

Compaction 入口有两类：manual compact task 与 auto compact。

| 触发 | 入口 | 行为 |
| --- | --- | --- |
| 用户手动 compact | [`core/src/tasks/compact.rs`](./codex/codex-rs/core/src/tasks/compact.rs) | 启动 `CompactTask`，按 provider/feature 选择 token-budget、remote v2、remote legacy 或 local。 |
| pre-turn token limit | [`run_pre_sampling_compact`](./codex/codex-rs/core/src/session/turn.rs) | 普通 turn 开始前发现 token limit reached，先 compact。 |
| comp_hash 变化 | [`maybe_run_previous_model_inline_compact`](./codex/codex-rs/core/src/session/turn.rs) | 上一模型与当前模型的 compaction compatibility hash 不同，使用上一模型 context compact。 |
| model downshift | 同上 | 切到更小 context window 的模型且当前 active tokens 已超过新限制时 compact。 |
| mid-turn context limit | [`run_turn`](./codex/codex-rs/core/src/session/turn.rs) | sampling 后仍需 follow-up，且 token limit reached 或收到 new context window request，则 mid-turn compact 后继续。 |
| token-budget feature | [`compact_token_budget.rs`](./codex/codex-rs/core/src/compact_token_budget.rs) | 跳过摘要，直接开启新 context window。 |

`ModelInfo::auto_compact_token_limit()` 默认使用 context window 的 90%，也允许模型配置显式 limit，但会被 context window 上限 clamp。context status 还可以按 `Total` 或 `BodyAfterPrefix` scope 计算。

### InitialContextInjection

[`InitialContextInjection`](./codex/codex-rs/core/src/compact.rs) 是 compaction 后上下文摆放的关键枚举：

| 变体 | 使用场景 | 语义 |
| --- | --- | --- |
| `DoNotInject` | manual / pre-turn compact | replacement history 不插初始上下文；reference context 置空，下一次普通 turn 会完整重新注入。 |
| `BeforeLastUserMessage(world_state)` | mid-turn compact | 把当前 canonical initial context 插入 replacement history 中最后真实用户消息之前，让 compact 后同一 turn 的 follow-up 能立即继续。 |

为什么 mid-turn 要特殊处理？因为模型已经处在一个 turn 的连续采样中，compact 后马上可能继续请求工具或回答用户。如果只等下一轮普通 turn 再注入上下文，中间这次 follow-up 就会缺少当前权限/环境/AGENTS.md。Codex 因此把 context 插回 replacement history。

### Local Compaction：客户端 prompt 生成摘要

local compaction 的实现是 [`core/src/compact.rs`](./codex/codex-rs/core/src/compact.rs)：

```text
run_inline_auto_compact_task / run_compact_task
  -> 选择 config.compact_prompt 或 SUMMARIZATION_PROMPT
  -> 把 compaction prompt 当作合成 user input 写入临时 history
  -> 用 base_instructions 发起一次 Responses stream
  -> 读取最后 assistant message 作为 summary
  -> 加 SUMMARY_PREFIX
  -> collect_user_messages(...)
  -> build_compacted_history(...)
  -> 必要时插入 initial context
  -> replace_compacted_history(...)
```

本地 compaction 模板在 [`prompts/templates/compact/prompt.md`](./codex/codex-rs/prompts/templates/compact/prompt.md)。它明确告诉模型正在做 context checkpoint compaction，并要求生成给“另一个将继续任务的 LLM”的 handoff summary。固定前缀来自 [`summary_prefix.md`](./codex/codex-rs/prompts/templates/compact/summary_prefix.md)，用于把摘要包装成可识别的 summary user message。

local compaction replacement history 的结构大致是：

```text
[可选 initial context]
[最近真实 user messages，最多约 20k token]
[user message: SUMMARY_PREFIX + summary]
```

用户消息保留逻辑有两个细节：

- `collect_user_messages` 会跳过已有 summary message，避免 summary 套 summary。
- `build_compacted_history_with_limit` 从最近用户消息往前取，最多 `COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000`，必要时截断最早被选中的那条。

如果 compact 请求本身遇到 context window exceeded，local compaction 会从历史开头删除最旧 item 后重试，尽量保留最近上下文。

### Remote Compaction Legacy：`/responses/compact`

legacy remote compaction 在 [`core/src/compact_remote.rs`](./codex/codex-rs/core/src/compact_remote.rs)。它仍然由客户端构造一个完整 `Prompt`，包括：

- compact 前经过 `for_prompt(...)` 的 history。
- 当前 tool specs。
- base instructions。
- reasoning / service tier / prompt cache key 等 Responses request 字段。

然后 [`ModelClient::compact_conversation_history`](./codex/codex-rs/core/src/client.rs) 把它发送到 `/responses/compact`。这个 endpoint 返回 `Vec<ResponseItem>` 形式的 compacted history。

客户端收到后不会直接信任全部输出，而是调用 `process_compacted_history(...)`：

- 删除 remote 输出里的 `developer` message，避免 stale/duplicated instructions。
- user message 只保留能解析成真实 user message 或 hook prompt 的内容。
- 保留 assistant / agent message / compaction item。
- 删除工具 call/output、web/image call、AdditionalTools、CompactionTrigger 等不该进入 replacement history 的 item。
- 按 `InitialContextInjection` 插入当前 canonical initial context。

legacy remote compaction 还会在发送前尝试 `trim_function_call_history_to_fit_context_window(...)`，把过大的 function output 改写成截断提示，保证 compact endpoint 本身能装下输入。

### Remote Compaction v2：`CompactionTrigger`

remote v2 在 [`core/src/compact_remote_v2.rs`](./codex/codex-rs/core/src/compact_remote_v2.rs)，它不调用 `/responses/compact`，而是在普通 Responses stream 输入末尾追加：

```text
ResponseItem::CompactionTrigger {}
```

协议层将它序列化为：

```json
{ "type": "compaction_trigger" }
```

服务端应在 stream 中返回且只返回一个 `ResponseItem::Compaction`。客户端 `collect_compaction_output(...)` 会强校验：

- 必须看到 `response.completed`。
- output item 中必须正好有一个 `Compaction`。
- 否则返回 fatal/stream error。

v2 的 replacement history 不是完全由服务端给出，而是由客户端构造：

```text
prompt_input
  -> 只保留 role 为 user/developer/system 的 message
  -> 再经过 should_keep_compacted_history_item 过滤
  -> 按 64k token retained-message budget 截断
  -> 追加服务端返回的 Compaction item
  -> process_compacted_history 插入 canonical context
```

这条路径的含义是：summary/compaction 内容由服务端模型能力生成，但“哪些历史项继续保留、如何插回当前上下文、如何安装新窗口”仍由客户端控制。相比 legacy remote，它更接近 Responses-native 的 compaction control item，也能把 compaction token usage、cached tokens、retained image count 纳入 analytics。

### Token Budget Compaction：无摘要的新窗口

当 `Feature::TokenBudget` 启用时，auto/manual compact 会走 [`compact_token_budget.rs`](./codex/codex-rs/core/src/compact_token_budget.rs)。这条路径不调用模型，也不生成 summary，而是：

```text
run_pre_compact_hooks
  -> emit ContextCompaction turn item
  -> start_new_context_window(...)
  -> emit completed
  -> run_post_compact_hooks
```

`start_new_context_window(...)` 会用当前 world state 构造 initial context，并安装一个空 message 的 `CompactedItem`。这更像“显式开启新 window”，适合 token budget 机制接管上下文重置的场景。

### Replacement History 安装

所有 compaction 路径最终都会调用 [`replace_compacted_history`](./codex/codex-rs/core/src/session/mod.rs)。它做几件关键事情：

1. 如果启用 item ids，为 replacement history 补缺失 id。
2. 用 replacement history 替换 `ContextManager` 当前 history。
3. 设置或清空 `reference_context_item`。
4. 如果传入 world state baseline，则以 full snapshot 形式持久化。
5. 持久化 `RolloutItem::Compacted`，必要时再持久化 `WorldState` 和 `TurnContext`。
6. queue pending session-start source 为 `Compact`，让后续 hooks/生命周期知道是 compact 后的新窗口。
7. 重新计算 token usage。

因此 compaction 是一个显式历史窗口边界，而不是在原 history 里追加一条摘要就结束。

## Compaction Prompt 组织结构

compaction prompt 的组织分为客户端模板与 Responses-native item 两代：

### 本地模板

[`prompts/src/compact.rs`](./codex/codex-rs/prompts/src/compact.rs) 只导出两个常量：

```text
SUMMARIZATION_PROMPT = templates/compact/prompt.md
SUMMARY_PREFIX = templates/compact/summary_prefix.md
```

`prompt.md` 的语义是“正在进行 context checkpoint compaction，请生成 handoff summary”；它要求摘要包含当前进展、关键决定、约束/偏好、剩余步骤、继续所需的关键数据/例子/引用。`summary_prefix.md` 则把 summary 包装成“另一个模型已经开始解决并留下摘要”的格式。

配置上，local compact 可以用 `compact_prompt` 或 `experimental_compact_prompt_file` 覆盖模板。配置解析会 trim 空白，空字符串视为未设置。

### Remote legacy

legacy remote compact 不使用客户端 `SUMMARIZATION_PROMPT`。客户端把完整 prompt/request 送到 `/responses/compact`，服务端负责内部 compaction 策略。客户端关注的是输入裁剪、输出过滤、canonical context reinjection 和 replacement history 安装。

### Remote v2

remote v2 也不发送自然语言 summarization prompt。客户端只在 Responses input 末尾追加 `compaction_trigger`，相当于把“请 compact”变成结构化控制 item。服务端返回 `compaction` item，客户端再把它和 retained messages 合成新 history。

这说明 Codex 的 compaction prompt 正在从“客户端自然语言模板”迁移到“Responses API 结构化 compaction primitive”。

## 与 Prompt Cache 的关系

Compaction 会破坏旧 history prefix，但它也创建新的稳定前缀。设计上有几个 cache 友好的选择：

1. **普通 turn 不反复重写 prefix**：上下文变化追加 diff。
2. **pre-turn compact 后下一轮完整重新注入**：新 window 的开头是 canonical context，而不是不可信摘要里的权限/环境描述。
3. **mid-turn compact 精确插入 context**：同一 turn 继续采样时保持 prompt 结构可预测。
4. **remote v2 retained messages 从尾部预算截断**：优先保留最近消息和图片，减少无效旧前缀。
5. **window id 与 compaction metadata 单独走控制面**：模型可见历史和运行时追踪信息分离。

更完整的 prompt cache 机制、`prompt_cache_key` 和 WebSocket `previous_response_id` 关系，见 [`codex-responses-api.md`](./codex-responses-api.md)。

## 测试与验证线索

可优先阅读这些测试来验证上面的判断：

| 测试 | 关注点 |
| --- | --- |
| [`core/tests/suite/compact.rs`](./codex/codex-rs/core/tests/suite/compact.rs) | local compaction、custom compact prompt、mid-turn compact、history replacement。 |
| [`core/tests/suite/compact_remote.rs`](./codex/codex-rs/core/tests/suite/compact_remote.rs) | remote/remote v2 compaction、filter、retained history、error path。 |
| [`core/tests/suite/compact_remote_parity.rs`](./codex/codex-rs/core/tests/suite/compact_remote_parity.rs) | local 与 remote compact 行为对齐。 |
| [`core/tests/suite/prompt_caching.rs`](./codex/codex-rs/core/tests/suite/prompt_caching.rs) | prompt prefix 稳定性与 `prompt_cache_key`。 |
| [`core/tests/suite/responses_lite.rs`](./codex/codex-rs/core/tests/suite/responses_lite.rs) | Responses Lite 下 instructions/tools 前插到 input。 |
| [`prompts/src/permissions_instructions_tests.rs`](./codex/codex-rs/prompts/src/permissions_instructions_tests.rs) | sandbox/approval prompt 渲染。 |
| [`prompts/src/goals_tests.rs`](./codex/codex-rs/prompts/src/goals_tests.rs) | goal prompt XML escaping 与模板渲染。 |
| [`core/src/session/tests.rs`](./codex/codex-rs/core/src/session/tests.rs) | initial context、context updates、extension fragments、world-state reinjection。 |

## 设计评价

Codex prompt 系统的核心优点是把 prompt 当作可维护的运行时状态，而不是一次性字符串：

- **清晰分层**：基础行为、权限/环境、用户需求、工具能力、特殊任务 prompt 分开维护。
- **可恢复**：history 是 typed item，compaction 可重写窗口，rollout 可持久化 replacement history。
- **可缓存**：稳定前缀和 append-only diff 降低重复输入成本。
- **可审计**：permissions、guardian、review、goal 等关键指令有独立模板和 tests。
- **可演进**：local natural-language compaction 与 remote structured compaction 可以并存，按 provider/feature 切换。

风险和复杂度也很明显：

- prompt 来源分散，理解一次请求需要跨 `protocol`、`core`、`prompts`、`ext`、`memories` 多个 crate。
- initial context 与 steady-state diff 并不完全等价，源码里也保留了 TODO，说明仍有一些 model-visible item 的 diff/replay 不完全确定。
- remote compaction 输出必须经过严格过滤，否则 stale developer instructions 或旧环境可能污染新窗口。
- Responses Lite 改变了 instructions/tools 的布局，所有 prompt cache 和测试都要同时覆盖普通模式与 Lite 模式。

总体看，Codex 的 prompt 系统已经从“prompt engineering 文件集合”演化成“agent runtime 的上下文操作系统”：模板只是入口，真正的设计重心在 typed history、上下文 diff、工具控制面和 compaction window 管理。
