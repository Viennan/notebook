# hermes-agent core report

## 核心结论

`hermes-agent` 的主干不是一个单纯 CLI chat wrapper，而是一个多入口共享的 agent runtime：CLI、messaging gateway、ACP、cron、batch runner 等入口最终都收敛到 `AIAgent.run_conversation()`，由它在一次用户 turn 内完成 system prompt 组装、provider/runtime 选择、可中断模型调用、工具调用循环、session 持久化、上下文压缩、memory/skill 后处理和 fallback。

当前 commit：`bb7ff7dc302cbcbe41cf6bc09424ffc9fb2d062f`。本文以该 submodule 源码为准，官方文档作为定位和概念辅助。

## 源码地图

| 模块 | 作用 |
| --- | --- |
| [`hermes-agent/run_agent.py`](./hermes-agent/run_agent.py) | `AIAgent` facade。保留公共 API、状态属性、provider transport、工具执行 forwarder、session/memory/compression 边界。 |
| [`hermes-agent/agent/agent_init.py`](./hermes-agent/agent/agent_init.py) | `AIAgent.__init__` 的真实实现：provider/client、tool surface、context compressor、memory manager、回调、预算与 session 状态初始化。 |
| [`hermes-agent/agent/conversation_loop.py`](./hermes-agent/agent/conversation_loop.py) | 单 turn 的 agent loop：构造 API messages、调用 provider、解析响应、处理 tool calls、retry/fallback、最终返回结果。 |
| [`hermes-agent/agent/turn_context.py`](./hermes-agent/agent/turn_context.py) | 每个 turn 的 prologue：安全 stdio、session DB、MCP refresh、message sanitize、系统提示词缓存、memory/context 注入、预压缩准备。 |
| [`hermes-agent/agent/system_prompt.py`](./hermes-agent/agent/system_prompt.py) / [`hermes-agent/agent/prompt_builder.py`](./hermes-agent/agent/prompt_builder.py) | system prompt 三层组装：stable、context、volatile。包含身份、工具指导、skills、项目上下文、memory/user profile、平台提示。 |
| [`hermes-agent/hermes_cli/runtime_provider.py`](./hermes-agent/hermes_cli/runtime_provider.py) | CLI、gateway、cron 共用的 provider/runtime 解析：model、provider、api mode、base URL、credentials、fallback/custom provider。 |
| [`hermes-agent/model_tools.py`](./hermes-agent/model_tools.py) / [`hermes-agent/tools/registry.py`](./hermes-agent/tools/registry.py) | 工具 schema 发现、toolset 过滤、工具调用 dispatch、plugin/MCP 工具接入和 sync/async 桥接。 |
| [`hermes-agent/agent/tool_executor.py`](./hermes-agent/agent/tool_executor.py) | assistant tool_calls 的并发/串行执行、agent-level tools、tool result budget、middleware/hook、结果回填。 |
| [`hermes-agent/hermes_state.py`](./hermes-agent/hermes_state.py) | SQLite session store，保存 sessions/messages/token/cost/system prompt，并提供 FTS5 session search。 |
| [`hermes-agent/gateway/run.py`](./hermes-agent/gateway/run.py) / [`hermes-agent/gateway/session.py`](./hermes-agent/gateway/session.py) | 多平台 messaging gateway：消息授权、session key、transcript load、agent cache、平台回调、回复投递。 |
| [`hermes-agent/tools/delegate_tool.py`](./hermes-agent/tools/delegate_tool.py) | subagent delegation：创建隔离 child `AIAgent`，限制工具集，父 agent 只收到摘要结果。 |

## 主执行链路

CLI 路径：

```text
用户输入
  -> HermesCLI 维护 conversation_history / callbacks / interrupt queue
  -> self.agent.run_conversation(...)
  -> agent.turn_context.build_turn_context(...)
  -> agent.conversation_loop.run_conversation(...)
  -> provider API call
  -> tool_calls? -> AIAgent._execute_tool_calls(...)
  -> final response
  -> session persistence / memory sync / UI display
```

Gateway 路径：

```text
Platform adapter MessageEvent
  -> GatewayRunner._handle_message_with_agent(...)
  -> SessionStore.get_or_create_session(...)
  -> load transcript from SessionDB
  -> GatewayRunner._run_agent(...)
  -> cached or new AIAgent(...)
  -> AIAgent.run_conversation(...)
  -> deliver final_response through platform adapter
```

两条路径的区别主要在外围：CLI 用本地 prompt_toolkit/TUI、线程和 stdin interrupt；gateway 用 async platform adapter、session key、agent cache、进度消息、approval/clarify 的平台桥接。真正的 agent turn 仍共享同一套 `AIAgent` loop。

## AIAgent 是 facade，agent loop 已被拆层

`run_agent.py` 仍然是公共入口，但它已经不是所有逻辑都堆在一个类方法里。`AIAgent.__init__` forward 到 `agent.agent_init.init_agent()`，`run_conversation()` forward 到 `agent.conversation_loop.run_conversation()`，system prompt、tool executor、runtime helpers 等也都以 forwarder 形式拆出。

这种结构保留了旧测试和外部调用对 `run_agent.AIAgent` 的依赖，同时把主干拆成几个边界：

- init：建立 provider client、工具 surface、memory/context engine、callbacks、session counters。
- turn prologue：每 turn 只做一次的准备，例如恢复 primary runtime、MCP refresh、system prompt restore/build、memory prefetch。
- loop：provider call、response validation、tool execution、retry/fallback 和退出。
- helper：工具执行、API kwargs、transport adapter、session persistence、compression 和 memory sync。

## Prompt 设计：稳定系统提示词加临时用户上下文

官方 docs 和源码都强调一个核心约束：system prompt 尽量保持 byte-stable。`agent/system_prompt.py` 把 prompt 分成三层：

```text
stable:
  identity / tool guidance / skills index / environment hints / platform hints
context:
  caller system_message / AGENTS.md / .hermes.md / CLAUDE.md / .cursorrules
volatile:
  MEMORY.md / USER.md / external memory block / timestamp-session-model line
```

`conversation_loop.run_conversation()` 每次 API call 都把 cached system prompt 放在最前面；plugin context、external memory prefetch 等更临时的信息优先注入当前 user message，而不是重写 system prompt。这样做的结果是：多 turn session 能维持 provider prefix cache，同时又能在单 turn 内加入必要的 recall/context。

这也是分析 Hermes 时要特别注意的边界：不是所有“模型看到的上下文”都来自 transcript。有些内容来自 cached system prompt，有些只在 API-call-time 进入 user message，原始 transcript 不会被污染。

## Provider runtime：统一模型协议，再适配不同 API mode

Hermes 支持多 provider，但主 loop 不直接把每家 provider 写成独立分支散落在入口层。`hermes_cli/runtime_provider.py` 负责把配置、环境变量、OAuth/credential pool、自定义 provider 和 base URL heuristic 解析成 runtime：

- `chat_completions`：OpenAI-compatible endpoints 和大多数聚合服务。
- `codex_responses`：OpenAI Responses / Codex 相关路径。
- `anthropic_messages`：Anthropic Messages 原生协议或兼容路径。
- `bedrock_converse`、`codex_app_server` 等特殊路径也在 transport 层收敛。

在 loop 里，`agent._build_api_kwargs()` 和 `agent._get_transport()` 把内部 OpenAI-style message shape 转成对应 provider 请求。响应回来后再归一化成 internal assistant message 和 tool_calls。也就是说，Hermes 的 provider 多样性主要被压在 runtime resolution + transport adapter，工具循环本身保持统一。

## Tool runtime：registry + toolset + agent-level interception

工具系统分三层：

```text
tools/*.py
  -> registry.register(...)
model_tools.get_tool_definitions(...)
  -> 按 toolset / availability / dynamic schema 生成 provider tools
conversation_loop
  -> assistant.tool_calls
  -> agent.tool_executor
  -> agent-level tool 或 registry.dispatch
```

`tools/registry.py` 用 AST 扫描找出顶层 `registry.register(...)` 的工具模块并导入，工具由模块自注册。`model_tools.get_tool_definitions()` 按 enabled/disabled toolsets 过滤，并用 availability check 排除当前不可用工具。`handle_function_call()` 是通用 dispatcher，同时处理 `tool_search/tool_describe/tool_call` 这样的 deferred bridge，并在 dispatch 前后接入 middleware/plugin hooks。

并非所有工具都只是 registry handler。`todo`、`memory`、`session_search`、`delegate_task`、`clarify`、context-engine/memory-provider tools 等需要访问 `AIAgent` 状态，因此在 `agent/tool_executor.py` 或 `agent/agent_runtime_helpers.py` 中被拦截处理。这些工具虽然有 schema，但真实语义依赖当前 agent/session。

工具执行还有两个工程化特征：

- 并发执行只用于看起来独立的 tool batch；路径可能冲突或交互型工具会走串行。
- tool result 会做预算控制、必要时外存化，防止单个超大结果挤爆上下文窗口。

## Session 与 recall：SQLite 是跨入口的记忆底座

Hermes 的 session 持久化在 [`hermes_state.py`](./hermes-agent/hermes_state.py)。`SessionDB` 保存 session metadata、messages、tool calls、reasoning、token/cost、system prompt snapshot，并维护 FTS5 / trigram FTS5 索引用于 session search。

CLI 和 gateway 都把 transcript 作为下一 turn 的 history 来源。Gateway 额外有 `SessionStore` 维护 `session_key -> session_id` 映射、reset policy、resume/suspend 状态和平台来源。这样同一个 agent loop 可以被不同平台复用，同时每个平台对“会话 lane”的定义由 gateway session key 控制。

这解释了 Hermes 的一个产品能力：它既能继续当前 chat，也能通过 `session_search` 在历史 transcript 中找回跨 session 信息。持久 memory 和 session search 是两类不同机制：

- memory：提炼后的长期事实，进入 system prompt / external memory block。
- session search：历史 transcript 检索，按需作为 recall 资料进入当前 turn。

## Memory 与 skills：自我改进环在主 loop 两侧

Hermes README 的“closed learning loop”主要落在 memory、skills 和 background review 三类机制上。

`agent/memory_manager.py` 统一外部 memory provider：pre-turn prefetch、post-turn sync、memory provider tools、session end 生命周期。内置 `memory` tool 写本地 memory/user profile，同时会 mirror 到外部 provider。`run_agent.py` 还有 `commit_memory_session()`、`shutdown_memory_provider()`、`_sync_external_memory_for_turn()` 等边界，确保 session 边界和 turn 完成时能把信息同步出去。

Skills 则通过 `agent/skill_commands.py`、`agent/skill_bundles.py`、`tools/skills_tool.py`、`tools/skill_manager_tool.py` 等进入两条路径：

- prompt surface：skills index 进入 stable system prompt，模型被引导在适配任务时加载 skill。
- action surface：`skills_list`、`skill_view`、`skill_manage` 等工具允许查看、使用、创建和修补技能。

所以 Hermes 的“自我改进”不是让模型隐式改变自身，而是把经验沉淀到可审计文件/skill/provider 中，再在后续 prompt 和工具面重新加载。

## Delegation：subagent 是隔离子会话，不是共享上下文 swarm

`delegate_task` 会创建 child `AIAgent`，拥有自己的 task id、工具会话、精简系统提示词和受限 toolsets。默认阻断递归 delegation、clarify、memory、send_message、execute_code 等工具，避免 child agent 修改共享 memory 或产生跨平台 side effects。

父 agent 看不到 child 的完整中间 transcript，只收到 summary/result。这使 delegation 更像“隔离工作流分派”：用上下文经济换并行/专注，并通过工具集限制和 session 隔离降低风险。

## Context compression：压缩是 session lineage 的一部分

当历史接近模型上下文窗口时，Hermes 会压缩中间 turn，保留近期消息与工具调用配对，并生成 handoff summary。Gateway 还有 turn 前 hygiene compression，用于长时间运行的平台会话。

压缩不是简单覆盖聊天记录：官方 session-storage docs 和源码都体现了 session lineage 设计。压缩可以生成 child session，旧 transcript 仍可通过 search 找回；系统提示词也会在压缩后按当前上下文重建。这一点很关键：Hermes 的“继续会话”并不等同于把所有历史原样塞回模型，而是把 live context、summary、persistent memory 和 searchable archive 分层管理。

## 设计取舍

Hermes 的核心设计有几个明显取向：

- runtime-first：把 CLI、gateway、ACP、cron 等入口都压到同一个 `AIAgent` loop，而不是每个入口各自实现 agent。
- prompt-cache aware：system prompt 稳定性是显式工程目标，临时上下文尽量走 user-message 注入。
- tool-surface governed：工具不是裸函数列表，而是 registry、toolset、availability、middleware、approval、budget 和 agent-level interception 的组合。
- persistent agent：session DB、memory provider、skills、gateway session key、compression lineage 一起支撑跨 turn/跨平台连续性。
- provider-neutral but not provider-blind：内部统一 message/tool 形态，但对 Anthropic、Responses API、Bedrock、本地/自定义 endpoint 都保留专门 transport 和兼容逻辑。

## 当前分析边界

本文只建立 core execution map，暂不展开：

- 每个 gateway platform adapter 的协议细节。
- Terminal/browser/MCP/managed tool gateway 的具体实现。
- ACP、cron、kanban、trajectory training 的专题机制。
- 具体 memory provider，例如 Honcho/Supermemory 的内部协议。

后续适合继续拆专题：

1. `hermes-agent-tool-runtime.md`：工具 registry、toolsets、approval、MCP、terminal backend。
2. `hermes-agent-prompt-memory-skills.md`：system prompt、memory、skills、自我改进闭环。
3. `hermes-agent-gateway-session.md`：gateway session key、agent cache、platform callbacks、reset/resume。
