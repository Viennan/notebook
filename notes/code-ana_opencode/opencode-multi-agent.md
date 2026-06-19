# opencode Multi-Agent 能力与设计研究

本文研究 opencode 在 multi-agent 方面的能力与实现方式，重点关注 coding agent 主干中真实参与工作的系统：agent profile、`task` 工具、child session、权限隔离、前后台任务和父子 agent 通信。不展开 TUI 细节、企业版和配置文件加载细节；涉及 V2 时只讨论和 multi-agent 抽象直接相关的部分。

核心结论先放在前面：

> opencode 的 multi-agent 不是“多个 agent 组成一个自治群聊/黑板系统”，而是“主 agent 通过 `task` 工具把一个封闭任务委派给 subagent child session”。它更像可审计、可权限约束、可恢复的任务委派树，而不是 peer-to-peer agent swarm。

这一区分很重要。opencode 的主执行单位仍是 session loop。subagent 并没有获得一个和 parent 对等的共享对话空间；它获得的是一个新的 session、一个 agent profile、一组权限、一个明确 prompt，然后在完成后把最终文本结果返回给 parent。

## 关键源码地图

- 旧版当前主路径：
  - [`packages/opencode/src/agent/agent.ts`](./opencode/packages/opencode/src/agent/agent.ts)
  - [`packages/opencode/src/tool/task.ts`](./opencode/packages/opencode/src/tool/task.ts)
  - [`packages/opencode/src/tool/task.txt`](./opencode/packages/opencode/src/tool/task.txt)
  - [`packages/opencode/src/tool/registry.ts`](./opencode/packages/opencode/src/tool/registry.ts)
  - [`packages/opencode/src/agent/subagent-permissions.ts`](./opencode/packages/opencode/src/agent/subagent-permissions.ts)
  - [`packages/opencode/src/session/prompt.ts`](./opencode/packages/opencode/src/session/prompt.ts)
  - [`packages/opencode/src/session/session.ts`](./opencode/packages/opencode/src/session/session.ts)
  - [`packages/core/src/background-job.ts`](./opencode/packages/core/src/background-job.ts)
- V2/core 侧抽象：
  - [`packages/core/src/agent.ts`](./opencode/packages/core/src/agent.ts)
  - [`packages/core/src/config/agent.ts`](./opencode/packages/core/src/config/agent.ts)
  - [`packages/core/src/plugin/agent.ts`](./opencode/packages/core/src/plugin/agent.ts)
- 用户可见文档：
  - [`packages/web/src/content/docs/agents.mdx`](./opencode/packages/web/src/content/docs/agents.mdx)

## 功能特点

opencode 当前 multi-agent 能力主要表现为以下几类。

### Primary agent 与 subagent

agent 被分为 `primary`、`subagent`、`all` 三种 mode。

- `primary`：用户主会话中直接交互的 agent，例如 `build`、`plan`。
- `subagent`：被主 agent 通过 `task` 调用，或被用户 `@` 提及触发的子代理，例如 `general`、`explore`。
- `all`：既可以作为 primary，也可以作为 subagent 的自定义 agent 默认模式。

源码中的旧版内置 agent 主要是：

- `build`：默认主 agent，执行工具并完成 coding task。
- `plan`：主 agent，禁止普通 edit/write，允许生成计划并通过 `plan_exit` 转入执行。
- `general`：通用 subagent，用于复杂研究、多步骤任务和并行工作，默认禁止 `todowrite`。
- `explore`：代码库探索 subagent，偏 read/search，默认不允许修改文件。
- `compaction`、`title`、`summary`：隐藏内部 agent，不属于用户主要 multi-agent 协作面。

V2 core 的 [`plugin/agent.ts`](./opencode/packages/core/src/plugin/agent.ts) 也初始化了 `build`、`plan`、`general`、`explore` 和隐藏内部 agent。也就是说，agent profile 已经被抽象到 core 层，但当前 `task`/subagent 运行链路仍主要在 `packages/opencode/src` 旧路径里。

文档 [`agents.mdx`](./opencode/packages/web/src/content/docs/agents.mdx) 提到 `Scout`，但当前本地源码的旧版 agent 表和 V2 agent plugin 默认表里都没有看到 `scout` 的默认注册。本文以源码默认行为为准：当前核心默认 subagent 是 `general` 与 `explore`。

### 自动委派与手动委派

opencode 支持两种委派入口：

1. **模型自动调用 `task` 工具**：主 agent 在普通 session loop 中看到 `task` 工具，自己决定是否调用某个 `subagent_type`。
2. **用户手动 `@subagent` 或 command 指定 subagent**：`SessionPrompt.command` 会把目标 subagent 转成 `SubtaskPart`，随后 `handleSubtask` 直接执行 `task` 工具。

前者是 agent 自主调度，后者是用户指定角色。两者最终都落到同一个边界：`TaskTool.execute`。

### 并行与后台 subagent

[`task.txt`](./opencode/packages/opencode/src/tool/task.txt) 明确提示模型可以在一个响应里发起多个 task tool calls，从而并行运行多个 subagent。并行性不是来自一个专门调度器，而是来自 provider/tool-call 层允许一个 assistant message 包含多个 tool call。

此外，`OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true` 时，`task` 参数会暴露 `background`。后台 task 会立即返回 running 结果，真正的 child session 在后台 job 中继续运行，完成后再把结果注入 parent session。

### 上下文隔离与可恢复

每次 fresh task 会创建一个 child session。subagent 默认不会看到 parent 的完整对话历史，parent 必须在 `prompt` 参数里写清楚背景、目标、边界和返回格式。若传入 `task_id`，则复用已有 child session，subagent 会继续自己原来的消息和工具历史。

这使 opencode 的 multi-agent 更接近“上下文隔离的分工委派”，而不是“所有 agent 共享同一上下文”。

## 整体架构

可以用下面的结构理解一次普通自动委派：

```text
parent SessionPrompt loop
  ├─ resolve current agent
  ├─ ToolRegistry.tools(...)
  │    └─ task tool description 动态列出可调用 subagents
  ├─ provider turn
  │    └─ model emits task({ subagent_type, prompt, description })
  └─ TaskTool.execute
       ├─ check task permission
       ├─ resolve subagent profile
       ├─ create/reuse child session(parentID = parent session)
       ├─ derive child session permissions
       ├─ ops.prompt(child session, subagent agent, prompt parts)
       │    └─ child SessionPrompt loop
       └─ wrap child final text as <task> result for parent
```

这里没有独立的 “multi-agent runtime”。真正复用的是同一个 `SessionPrompt` loop，只是换了 session、agent profile、model/variant 和 permission ruleset。

## Agent Profile：角色不是执行器，而是运行剖面

旧版 [`Agent.Info`](./opencode/packages/opencode/src/agent/agent.ts) 的关键字段包括：

- `name`
- `description`
- `mode`
- `model`
- `variant`
- `prompt`
- `temperature` / `topP`
- `permission`
- `steps`
- `hidden`

这说明 opencode 中“角色”的本质不是一个新的 actor class，而是一组影响同一个 agent loop 的运行剖面：

```text
agent role = system prompt + model selection + tool permissions + visibility + step policy
```

`description` 在 multi-agent 里尤其关键。它不是单纯给人看的文案，而会被拼进 `task` 工具描述中，成为模型选择 subagent 的路由提示。

## Task 工具：multi-agent 的调用边界

[`task` 工具](./opencode/packages/opencode/src/tool/task.ts) 是 multi-agent 的核心调用边界。它的参数包括：

- `description`：短描述，用于 tool title / child session title。
- `prompt`：真正交给 subagent 的任务说明。
- `subagent_type`：目标 agent 名称。
- `task_id`：可选，表示继续已有 subagent session。
- `command`：可选，记录由哪个 command 触发。
- `background`：可选 experimental 字段，表示后台运行。

模型看到的 [`task.txt`](./opencode/packages/opencode/src/tool/task.txt) 明确规定了几个行为：

- 只把复杂、多步骤任务交给 Task。
- 简单读文件、找 class、在少数文件里搜索时，不要用 Task。
- 可以并发启动多个 agents。
- 委派出去后不要重复做同一件事。
- subagent 返回结果不会直接展示给用户，parent 需要自己总结给用户。
- fresh invocation 没有 parent context，除非复用 `task_id`。
- prompt 必须包含详细任务说明和期望返回格式。
- 要告诉 subagent 是写代码还是只做研究。

这段提示词构成了 opencode multi-agent 的“软调度协议”：运行时提供 subagent 能力，模型依据描述决定何时调用、调用谁、如何写任务书。

## 任务分配机制：没有专门的意图识别 agent

opencode 没有看到一个专门的“任务分配 agent”或“意图识别 agent”。任务分配主要由三层机制共同完成：

1. **工具可见性**：`ToolRegistry.describeTask` 把当前 agent 可调用的 subagent 列到 `task` 工具描述里。
2. **subagent description**：每个 agent 的 `description` 描述适用场景。
3. **运行时权限**：`permission.task` 可以把某些 subagent 从当前 agent 的 task 工具描述中移除，或者在调用时要求审批/拒绝。

[`ToolRegistry.describeTask`](./opencode/packages/opencode/src/tool/registry.ts) 的核心逻辑是：

```text
agents.list()
  -> filter(mode !== "primary")
  -> filter(current agent 的 task permission 没有 deny 该 agent)
  -> 按 name 排序
  -> 拼成 "- name: description"
```

所以 parent model 不是在一个隐藏 router 的结果上行动，而是在 provider turn 中直接看到一个包含可用 subagent 清单的 `task` 工具，然后自己选择是否调用。

### plan mode 的强提示式分配

plan mode 里有更强的提示词约束。[`plan-mode.txt`](./opencode/packages/opencode/src/session/prompt/plan-mode.txt) 会要求：

- 初始理解阶段只使用 `explore` subagent。
- 范围不明确时最多并行启动 3 个 explore agents。
- 设计阶段可启动 `general` agent 协助形成计划。

这属于 prompt-level orchestration：不是 runtime 自动拆任务，而是系统提醒让 primary model 采用某种分工策略。

### output truncation 的委派提示

[`truncate.ts`](./opencode/packages/opencode/src/tool/truncate.ts) 还有一个很有意思的细节：当工具输出过长且当前 agent 有 task 能力时，截断提示会建议使用 Task tool 让 `explore` agent 去处理完整输出文件，而不是 parent 自己读取整份大文件。

这体现出 opencode 对 subagent 的一个重要定位：subagent 也是一种 context economy 工具，用来把大范围搜索、长输出消化、代码库扫描隔离到子上下文里。

## 角色指定：`subagent_type` 与 `@subagent`

角色指定有两条路径。

### 模型通过 `task.subagent_type` 指定

普通自动委派中，模型调用：

```json
{
  "description": "inspect cache bug",
  "prompt": "Investigate ... Return concise findings with file paths.",
  "subagent_type": "explore"
}
```

`TaskTool.execute` 根据 `subagent_type` 调用 `agent.get(...)` 找到目标 profile。找不到则报错。

### 用户通过 `@subagent` 或 command 指定

[`SessionPrompt.command`](./opencode/packages/opencode/src/session/prompt.ts) 中，如果 command 指向的 agent 是 `mode === "subagent"`，或 command 明确设置 `subtask === true`，就会创建 `SubtaskPart`：

```text
SubtaskPart
  type: "subtask"
  prompt
  description
  agent
  model?
  command?
```

主循环在处理 task queue 时看到 `SubtaskPart`，会走 `handleSubtask`，由它构造一条 assistant tool part，然后以 `bypassAgentCheck: true` 调用 `TaskTool`。

这个 bypass 很关键：用户显式 `@subagent` 本质上是用户直接指定子代理，因此不会再受 parent agent 的 `permission.task` 路由限制。文档也明确说，用户可通过 `@` 直接调用 subagent，即使当前 agent 的 task 权限会拒绝自动调用该 subagent。

但这不意味着 subagent 可以无权限乱跑。child session 里真正能调用哪些工具，仍由 subagent 自己的 permission 和 session permission 决定。

## Child Session：subagent 的执行容器

`TaskTool.execute` 创建或复用 child session：

```text
sessions.create({
  parentID: ctx.sessionID,
  title: description + " (@agent subagent)",
  agent: next.name,
  permission: derived child permission
})
```

`parentID` 是 session tree 的核心字段。[`Session.children(parentID)`](./opencode/packages/opencode/src/session/session.ts) 能查到某个 parent 下的 child sessions。删除 parent session 时也会递归删除 children，并取消相关 background jobs。

child session 的运行不是特殊逻辑。`TaskTool` 调用：

```text
ops.prompt({
  sessionID: childSession.id,
  agent: subagent.name,
  model,
  variant,
  parts
})
```

这会重新进入同一套 `SessionPrompt` loop。child session 有自己的 user/assistant/tool history、自己的 compaction、自己的 title/summary、自己的工具调用过程。parent session 只看到 `task` 这个 tool call 的输出。

## 权限隔离：parent 限制不是简单复制

subagent 权限是 opencode multi-agent 设计里最容易误解的一点。

[`deriveSubagentSessionPermission`](./opencode/packages/opencode/src/agent/subagent-permissions.ts) 的注释已经把原则讲得很清楚：

- parent agent restrictions 只治理 parent agent。
- subagent 自己的 permissions 决定 subagent 能力。
- child session 会继承 parent session 的 deny 规则和 `external_directory` 规则。
- 若 subagent 自己没有显式配置 `todowrite` 或 `task`，则默认在 child session 层 deny 这两类权限。

换句话说：

```text
effective child permission
  = subagent.permission
    + parent session deny rules
    + parent session external_directory rules
    + default child denies(todowrite/task if not explicitly permitted)
    + experimental primary_tools denies
```

这不是“parent 不能 edit，所以 child 也不能 edit”的继承模型。测试 [`plan-mode-subagent-bypass.test.ts`](./opencode/packages/opencode/test/agent/plan-mode-subagent-bypass.test.ts) 明确覆盖了这个设计：plan agent 自己禁止 edit，但一个拥有 edit 权限的 subagent 可以在自己的 session 中编辑，除非 parent session 层存在硬 deny。

设计含义是：

- `plan` 的只读性约束 parent planning loop。
- subagent 是否能执行写操作，由 subagent profile 决定。
- 用户或配置可以构造“plan agent 负责思考，executor subagent 负责执行”的模式。
- 真正的硬上限应放在 session permission 或全局/user permission，而不是只依赖 parent agent permission。

这也是为什么 `permission.task` 很重要：它控制 parent model 能不能自动调用某类 subagent。默认 plan agent 会 deny `task.general`，但允许 `task.explore`；用户可以覆盖这一点。

## 通信模型：非对称、一次性、以结果回流为主

opencode 的 agent 间通信是非对称的。

### Parent 到 child：完整任务 prompt

parent 通过 `task.prompt` 给 child 发送任务。fresh child session 不自动继承 parent 的完整对话历史，因此 task prompt 必须包含：

- 背景。
- 目标。
- 相关文件/路径/命令。
- 是否允许修改代码。
- 期望返回格式。
- 验证方式。

`TaskTool` 会调用 `ops.resolvePromptParts(params.prompt)`，所以 task prompt 可以被解析成 session prompt parts；但本质上仍是 parent 写给 child 的任务书，不是共享 transcript。

### Child 到 parent：最终文本结果

foreground task 完成后，`runTask` 只取 child 结果中最后一个 text part，并包装为：

```xml
<task id="..." state="completed">
<task_result>
...
</task_result>
</task>
```

这个结果是 parent assistant 的 tool output。`task.txt` 特别强调：subagent 结果不会直接对用户可见；parent 需要自己读结果、整合并回复用户。

### Background child 到 parent：synthetic user prompt 注入

background 模式下，`task` 会立即返回 running：

```xml
<task id="..." state="running">
<summary>Background task started</summary>
<task_result>
The task is working in the background...
</task_result>
</task>
```

后台 job 完成或失败后，`TaskTool.injectBackgroundResult` 会向 parent session 调用 `ops.prompt`，注入一条 synthetic text part，内容仍是 `<task>` wrapper。也就是说，后台结果不是通过某个 side-channel 给模型，而是作为后续 parent session 的合成用户输入进入主循环。

### 继续同一 child：`task_id`

如果 parent 传入已有 `task_id`，`TaskTool` 会尝试获取对应 session 并继续使用它。对于后台 running job，`BackgroundJob.extend` 会把新的 run 串到已有 job 的 tail 之后执行，而不是并发写同一个 child session。

这使 child agent 可以拥有自己的长期局部上下文，但这种上下文仍在 child session 内部，不会自动并入 parent。

### 没有 peer-to-peer channel

没有看到以下机制：

- subagent 之间直接发消息。
- 多个 agent 共享黑板状态。
- parent 实时读取 child 每一步思考并参与。
- 一个调度器根据 agent 输出自动拆下一级任务。

它们唯一稳定的共享对象是 workspace 文件系统、session/event 存储和 runtime 服务；语言层通信主要通过 `task` prompt/result 完成。

## 前台、后台与取消语义

foreground task 默认会阻塞 parent 当前 tool call，直到 child session 完成。若 parent run 被 abort，`TaskTool` 会取消 child session。

background task 是 experimental：

- 需要 `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true`。
- `background=true` 时立即返回 running。
- job 完成后自动向 parent 注入 synthetic task result。
- foreground task 可以被 promote 成 background，而不是重启。
- 继续同一个 running background task 时，`extend` 会排队追加上下文。
- 删除 parent session、删除 child session、取消 parent run，都会取消相应 background jobs。

[`BackgroundJob`](./opencode/packages/core/src/background-job.ts) 本身是 process-local registry。源码注释说明它不是 durable 的：进程重启会丢失 live job 状态。这一点限制了 background subagent 成为真正可靠的远程 agent worker 系统。

## 不同 activity/session 之间隔离了什么

可以把 opencode 的 subagent 隔离分成四层。

### Transcript 隔离

child session 有自己的 messages 和 parts。fresh task 不 replay parent history。parent 只收到 child 的最终文本结果。

### Tool state 隔离

child session 的 tool calls、tool outputs、attachments、compaction 都属于 child session。parent 只看到 `task` tool part。

### Permission 隔离

child 的能力由 subagent permission 与 child session permission 决定。parent agent 的普通工具限制不会自动降临到 child；parent session 的 deny/external_directory 会作为硬上限继承。

### UI/API 观察隔离

session tree 通过 `parentID` 暴露。UI 可以展示 subagent tabs，API 可以查询 children，但这属于 projection/观察层，不改变 agent 间通信模型。

共享的部分也要明确：

- 同一个 workspace 文件系统。
- 同一个 opencode 数据库和事件系统。
- 同一个 provider/model 访问环境。
- 可能继承 parent assistant 的 model/provider/variant，除非 subagent 自己指定 model。

所以 opencode 的隔离不是 OS 进程级沙箱，而是 session/context/tool-permission 层面的 agent 隔离。

## V2 方向：agent 抽象已 core 化，subagent 运行仍在迁移边界

V2 core 中 [`AgentV2.Info`](./opencode/packages/core/src/agent.ts) 把 agent 抽象成：

- `id`
- `model`
- `request`
- `system`
- `description`
- `mode`
- `hidden`
- `color`
- `steps`
- `permissions`

这比旧版 `Agent.Info` 更 provider-neutral，也更贴近 V2 durable runner 的 typed runtime。V2 的 [`plugin/agent.ts`](./opencode/packages/core/src/plugin/agent.ts) 初始化默认 agents，说明默认角色正在从旧 runtime 下沉到 core plugin。

不过，从当前源码看，完整 task/subagent 执行链路仍主要由旧版 [`TaskTool`](./opencode/packages/opencode/src/tool/task.ts) 与 [`SessionPrompt`](./opencode/packages/opencode/src/session/prompt.ts) 承担。V2 已经有 agent profile、typed permissions、runner/context/tool settlement 等基础设施，但 multi-agent task delegation 尚未完全表现为 V2-native 的 durable subagent runner。

这意味着阅读 opencode multi-agent 时要避免两个误区：

- 不要把 V2 agent schema 误认为当前所有 subagent 执行都已经 V2-native。
- 也不要把旧版 `TaskTool` 当成临时小功能；它实际上承载了当前 multi-agent 的主能力。

## 设计哲学

### 1. 委派而非群体自治

opencode 把多 agent 设计成父子任务委派，而不是 agent swarm。parent model 始终是用户任务的主叙事者和结果整合者；subagent 是被委派的工作单元。

好处是：

- 用户更容易理解当前谁在负责。
- parent 可以控制任务说明和结果整合。
- child 上下文不会污染 parent。
- 权限、审计、取消都能绑定到 session/tool call。

代价是：

- 没有复杂 agent 间协商。
- 没有自动共识/交叉验证。
- parent prompt 写得不好，child 就会缺上下文。
- 多个 subagent 写同一文件可能冲突，运行时没有高级冲突协调。

### 2. Prompt-mediated routing + runtime-enforced boundary

任务选择是 prompt-mediated 的：模型读 subagent description 后选择 `subagent_type`。但边界执行是 runtime-enforced 的：找不到 agent、权限 deny、child session 创建、权限合并、background cancellation 都在运行时完成。

这种组合比纯 prompt router 更可靠，也比纯静态调度器更灵活。

### 3. Agent 是能力包，而不是人格

opencode 的 agent profile 由 prompt、model、permissions、steps 组成。`explore` 不是“另一个人格”，而是“读/搜优先、禁止修改、适合代码库探索”的能力包。`general` 不是万能助手人格，而是“可承担多步骤任务的隔离上下文”。

这使自定义 agent 的重点也落在：

- 什么时候使用。
- 能用哪些工具。
- 是否能修改文件。
- 用什么模型。
- 输出什么格式。

### 4. 上下文经济优先

subagent 的 fresh context、`task_id` resume、truncation hint、plan mode 并行 explore，都说明 opencode 把 multi-agent 作为节省主上下文和并行搜索的手段。它不是为了模拟团队协作，而是为了让主 agent 少背大段上下文、少串行等待搜索。

### 5. 信任 subagent，但保持 parent 作为整合层

`task.txt` 写到 subagent outputs should generally be trusted。这是一个很强的设计选择：parent 不需要默认重复验证所有 child 发现，否则 multi-agent 的性能收益会消失。

但 parent 仍负责：

- 判断是否需要验证。
- 把结果转述给用户。
- 决定下一步是否继续委派或自己执行。
- 避免重复做已委派工作。

因此信任不是完全放权，而是把 subagent 当作可信的 worker result，同时保留主 agent 的协调职责。

## 值得注意的 magic

### `task` 描述会动态注入 subagent 清单

模型不是凭空知道可用 subagent。`ToolRegistry.describeTask` 每轮根据当前 agent 和权限动态拼接清单。被 `permission.task` deny 的 subagent 不会出现在 task tool description 里，模型通常也就不会尝试调用。

### 用户 `@subagent` 会 bypass parent task permission

自动委派受 `permission.task` 限制；用户直接指定 subagent 则通过 `SubtaskPart` 和 `bypassAgentCheck` 跳过这一层。这体现了“用户显式意图高于 parent agent 自动调度限制”的设计。

### parent agent 的 edit deny 不等于 child edit deny

这是最容易踩的点。plan agent 禁止 edit，不代表它启动的每个 child 都禁止 edit。child 是否能 edit 看 subagent 自己的 permission，以及 parent session/global hard deny。

### background result 以 synthetic prompt 回流

后台 subagent 完成后不是只更新 UI 状态，而是把 `<task>` 结果注入 parent session，让 parent model 在后续 loop 中处理。这保持了“所有影响模型行为的信息都进入 transcript”的一致性。

### `task_id` 让 subagent 拥有局部长期上下文

fresh task 是上下文隔离；`task_id` resume 则让某个 child worker 能连续工作。这是一种很克制的 memory：只在 child session 内延续，不自动扩散到 parent。

## FAQ：最近几个问题

### 1. background task 的结果是如何回到主 agent 的？

不是通过单独的事件总线直接送回 provider，而是先回到 parent session 的 transcript。

后台 job 完成后，[`TaskTool.injectBackgroundResult`](./opencode/packages/opencode/src/tool/task.ts) 会调用 `ops.prompt(...)`，向 parent session 写入一条 `synthetic: true` 的 `user` message；这条消息的文本内容是 `<task ...><task_result>...</task_result></task>` 这种封装。随后下一轮 [`MessageV2.toModelMessagesEffect`](./opencode/packages/opencode/src/session/message-v2.ts) 会把它序列化成普通 provider `role: "user"` 消息。

所以可以准确地说：

> background task 的输出结果先被包装成 synthetic user message，再在下一轮 provider request 中作为普通 user role 送回主 agent。

### 2. 模型会看到 `synthetic` 这个标记吗？

不会。`synthetic` 是 opencode 的内部元数据，用来标记“这是系统注入的消息，而不是用户手工输入的消息”。它在送入 provider 前就已经被消化掉了，模型只会看到文本内容本身。

这也是为什么 `synthetic` 不需要单独写进 prompt 里解释。opencode 只需要在自己的 session/state 层知道它是什么即可。

### 3. `<system-reminder>`、`<conversation-checkpoint>`、`<task>` 这些标签需要预先说明吗？

不是统一讲一遍，而是按 tag 分开处理。

- `<system-reminder>`：明确在模型提示词里说明过，`default.txt` 和 `anthropic.txt` 都写了它是系统自动附加的提醒，不是原始输入的一部分。
- `<conversation-checkpoint>`：在 [`to-llm-message.ts`](./opencode/packages/core/src/session/runner/to-llm-message.ts) 里直接声明它是“历史上下文”，不是新指令。
- `<task>`：更像 `task` 工具的固定结果封装格式，由 [`TaskTool.renderOutput`](./opencode/packages/opencode/src/tool/task.ts) 生成；opencode 没把它当成一门通用 tag 语法系统去教学，而是依赖任务提示、固定输出形状和上下文约定让模型理解。

换句话说，opencode 并不是假定模型“天生懂所有 tag”，而是对少数关键 tag 给出显式语义，其余则作为稳定约定和结构化输出模板来使用。

## 限制与风险

1. **调度依赖模型判断**：没有强制任务拆分器，模型可能不用 subagent、用错 subagent，或写出信息不足的 task prompt。
2. **通信带宽窄**：child 主要返回最终文本；复杂中间状态不会自然结构化回流。
3. **并行写冲突需要 parent 自控**：task prompt 要明确文件范围，否则多个 subagent 可能修改同一区域。
4. **background job 非 durable**：进程重启会丢 live job 状态。
5. **权限语义有反直觉处**：parent agent restriction 不自动继承给 child，安全边界应放在 session/global deny 或 subagent permission。
6. **V2 迁移未完全统一**：agent profile 已 core 化，但 task/subagent 执行仍主要在旧路径。

## 总结

opencode 的 multi-agent 能力可以概括为：

```text
primary agent = orchestrator
subagent = isolated child session worker
task tool = call boundary
agent profile = role + model + permissions
session parentID = hierarchy
task result = communication packet
background job = async execution wrapper
```

它的设计并不追求复杂自治，而是追求 coding agent 中更实用的几个目标：并行探索、上下文隔离、角色权限边界、可审计 child session、以及 parent 对用户叙事的持续掌控。

这也是 opencode 和很多“多智能体框架”口号不同的地方：它没有把 multi-agent 当作抽象戏剧，而是把它压缩成一个可运行、可取消、可权限控制、能复用同一 session loop 的工程机制。
