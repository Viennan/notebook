# hermes-agent memory system

## 核心结论

`hermes-agent` 的 memory system 不是一个统一数据库，而是多条长期上下文通道的编排：

```text
1. built-in curated memory
   MEMORY.md / USER.md
   -> session prompt build 时读取
   -> frozen snapshot 进入 cached system prompt

2. external provider static block
   provider.system_prompt_block()
   -> session prompt build 时读取
   -> cached system prompt 里只有静态说明和工具使用提示

3. external provider dynamic recall
   provider.prefetch(query)
   -> turn start 读取
   -> <memory-context> fenced block
   -> 追加到当前 API call 的 user message, 不写入 cached system prompt

4. normal tool-result recall
   provider tools / session_search / memory tool
   -> tool result 进入本 turn 的 normal messages
   -> 后续 API iteration 可见, 不是自动常驻 memory

5. post-turn durable write/sync
   memory tool / background review / provider.sync_turn()
   -> 写本地文件或外部 backend
   -> 下一 session、下一 turn prefetch、或按需工具查询时再读

6. SQLite transcript archive
   SessionDB + FTS5
   -> session_search 按需读取真实 transcript window
   -> 不做 LLM 总结, 不常驻 prompt
```

因此，分析 Hermes memory 时必须先问两个问题：

- 这条信息存在哪里：本地 memory 文件、当前 API user message、tool result、外部 backend、还是 SQLite transcript？
- 这条信息什么时候进入模型上下文：session prompt build、turn start、tool call 之后、还是 session_search 按需查询？

官方文档有时会笼统说 external provider “injects provider context into the system prompt”。当前源码需要更精确地区分：静态 provider header 进入 system prompt；动态 recalled memory 进入当前 user message 的 `<memory-context>` block。这个区分是理解 prompt cache、读写时机和安全边界的关键。

当前 submodule commit：`bb7ff7dc302cbcbe41cf6bc09424ffc9fb2d062f`。本文以当前源码为准，官方文档用于补充产品语境。

## 全局视角

一次普通用户 turn 的 memory 数据流可以压缩成下面这张图：

```text
session start / resume
  -> MemoryStore.load_from_disk()
  -> build_system_prompt()
       MEMORY.md snapshot
       USER.md snapshot
       provider.system_prompt_block()
       timestamp/model/provider line
  -> cache as agent._cached_system_prompt

turn start
  -> append original user message to in-memory messages
  -> early persist to SessionDB
  -> memory_manager.on_turn_start(turn_count, original_user_message)
  -> ext_prefetch_cache = memory_manager.prefetch_all(original_user_message)

first and later API calls in this turn
  -> copy messages into api_messages
  -> if current user message and ext_prefetch_cache:
       append build_memory_context_block(ext_prefetch_cache)
  -> send cached system prompt + api_messages + tools

tool loop
  -> built-in memory tool can write MEMORY.md / USER.md
  -> provider tools can search/store/delete in external backend
  -> session_search can read SQLite transcript windows
  -> tool results enter normal conversation messages

turn finalization
  -> persist assistant/tool messages to SessionDB
  -> _sync_external_memory_for_turn()
       memory_manager.sync_all(user_text, assistant_text, messages=messages)
       memory_manager.queue_prefetch_all(user_text)
  -> maybe spawn background_review fork

session boundary / compression / switch
  -> memory_manager.on_session_end(messages)
  -> memory_manager.on_pre_compress(messages)
  -> memory_manager.on_session_switch(...)
  -> provider-specific extraction/commit/flush
```

这里最重要的边界是：

- cached system prompt 是 session 级别的稳定前缀。
- external dynamic recall 是 turn 级别的临时 user-message context。
- tool result 是 loop 级别的普通消息。
- provider sync 是响应完成后的后台 durable write。
- SessionDB transcript 是长期档案，不等同于 memory provider。

## Memory 在 prompt 中的位置

| 信息类型 | 来源 | 进入模型的位置 | 是否缓存 | 是否持久化 | 刷新时机 |
| --- | --- | --- | --- | --- | --- |
| Built-in agent memory | `$HERMES_HOME/memories/MEMORY.md` | system prompt 的 volatile tier | 是，随整个 system prompt 缓存 | 是，本地文件 | session start、resume restored prompt、context compression 后 rebuild |
| Built-in user profile | `$HERMES_HOME/memories/USER.md` | system prompt 的 volatile tier | 是 | 是，本地文件 | 同上 |
| External provider static block | `provider.system_prompt_block()` | system prompt 的 volatile tier | 是 | provider 自己决定 | system prompt build/rebuild |
| External provider dynamic recall | `provider.prefetch(query)` | 当前 API user message 末尾的 `<memory-context>` | 否，只在本 turn API request 中临时拼接 | provider backend 中可能有 | turn start / provider queue_prefetch 结果 |
| Provider tool result | `provider.handle_tool_call()` | normal tool message | 否，作为 conversation messages 的一部分 | 通常会进 SessionDB；provider 是否吸收看 `sync_turn` | tool call 后立即可见 |
| Built-in memory tool result | `memory_tool()` | normal tool message | 否 | 成功写本地文件；失败只返回错误 | tool call 后立即可见 |
| `session_search` result | SQLite SessionDB | normal tool message | 否 | 已在 SessionDB | tool call 后立即可见 |
| Background review write | forked agent 调 `memory` 工具 | 不进入主 turn prompt | 否 | 若审批通过则写本地文件 | 响应完成后异步 |

`system_prompt.py` 明确把 memory snapshot、USER profile、external provider block 和 timestamp 放到 volatile tier，但 `build_system_prompt()` 的结果整体缓存在 `agent._cached_system_prompt`。所以 “volatile” 在这里不是每个 API call 重新渲染，而是相比 stable identity/context files 更接近 session-specific。

`conversation_loop.py` 进一步写明：external recall context is injected into the user message, not the system prompt。`messages` 原始列表不会被这个注入修改，因此 `<memory-context>` 不会因为 prefetch 注入而写进 SessionDB 的用户原文。

## Built-in curated memory

内建 memory 由 `tools/memory_tool.py` 的 `MemoryStore` 管理，维护两个 profile-scoped 文件：

```text
$HERMES_HOME/memories/MEMORY.md
$HERMES_HOME/memories/USER.md
```

默认容量：

| 文件 | 默认上限 | 语义 |
| --- | --- | --- |
| `MEMORY.md` | 2200 chars | agent 对环境、项目、工具、约定、稳定工作状态的个人笔记 |
| `USER.md` | 1375 chars | 用户身份、偏好、沟通风格、工作方式、长期期望 |

条目用 `\n§\n` 分隔，可以是多行。`MemoryStore.load_from_disk()` 做三件事：

```text
read MEMORY.md / USER.md
  -> split entries
  -> de-duplicate
  -> scan entries with strict threat patterns
  -> render frozen _system_prompt_snapshot
```

内建 memory 有两个并行状态：

- live state：`memory_entries` / `user_entries`，tool call 会修改并立即写盘。
- frozen snapshot：`_system_prompt_snapshot`，只在 `load_from_disk()` 时生成，用于 system prompt。

这意味着 `memory(action=add)` 成功时，只说明文件已经写入。当前 session 的 cached system prompt 不会立刻包含新条目。模型在当前 turn 只能通过 tool result 知道写入成功；下个 session 或 prompt invalidation/rebuild 后才会从 system prompt 看到它。

## Built-in memory 什么时候读

内建 memory 的读取路径有三类。

第一类是 prompt build：

```text
AIAgent init
  -> create MemoryStore
  -> MemoryStore.load_from_disk()

first run_conversation in session
  -> restore_or_build_system_prompt()
  -> build_system_prompt()
  -> MemoryStore.format_for_system_prompt("memory")
  -> MemoryStore.format_for_system_prompt("user")
  -> cache and persist system prompt
```

如果 session resume 能从 SessionDB 恢复已经存储过的 system prompt，则会复用那份 prompt，保持跨进程 resume 的前缀一致性。

第二类是 prompt invalidation：

```text
invalidate_system_prompt()
  -> agent._cached_system_prompt = None
  -> MemoryStore.load_from_disk()
```

源码注释说明这个路径主要用于 context compression。它是当前 session 内少数会让 frozen memory snapshot 刷新的机会。

第三类是 mutation 前的 live reload：

```text
memory tool mutation
  -> acquire file lock
  -> _reload_target(target)
  -> read latest disk state
  -> detect external drift
  -> apply mutation
```

这不是为了 prompt，而是为了避免多 session 或人工编辑造成的写入覆盖。

## Built-in memory 什么时候写

内建 memory 的写入口只有 `memory` 工具和审批 replay 路径，本质都调用同一个 `MemoryStore` API。

工具 schema 是一个单工具多 action 设计：

```text
single operation:
  action = add | replace | remove
  target = memory | user
  content / old_text

batch operation:
  target = memory | user
  operations = [
    { action, content?, old_text? },
    ...
  ]
```

`replace` / `remove` 不使用条目 ID，而使用 `old_text` 子串匹配。如果匹配多个不同条目，返回候选 preview 让模型更精确地指定。batch 是 all-or-nothing，并且只检查最终容量，因此可以在一次 tool call 里完成 “删旧 + 合并 + 加新”。

写入时序：

```text
model calls memory
  -> write_approval gate
       allow: continue
       block: return denied
       stage: save pending record, no write
  -> strict injection/exfil scan
  -> acquire target file lock
  -> reload disk and drift check
  -> capacity / match validation
  -> atomic temp file + os.replace()
  -> tool result
  -> MemoryManager.notify_memory_tool_write(...) if successful and committed
```

成功响应刻意不回显完整 entries，只返回 usage、entry count 和 “do not repeat” note。只有容量溢出、匹配失败或歧义时才返回当前 entries，帮助模型修复。

## 内建 memory 的生成算法

内建 memory 本身没有 embedding、reranking 或自动 fact extraction。它的“生成算法”来自两个 LLM 决策入口：

| 入口 | 谁决定写什么 | 何时运行 | 写入机制 |
| --- | --- | --- | --- |
| 前台 agent | 主模型根据 system prompt/tool schema/user 指令判断 | 正常 tool loop 中 | 直接调用 `memory` tool |
| Background review fork | 继承父 agent runtime 的 review agent 根据专门 prompt 判断 | 主响应完成后异步 | 只允许 memory/skill 类工具 |

前台写入通常来自用户明确要求、模型发现稳定偏好、或需要保存环境/项目约定。后台 review 的 prompt 明确关注：

- 用户透露的 persona、desires、preferences、personal details。
- 用户对 agent 行为、工作方式、风格的期望。

所以内建 memory 是 “LLM-curated durable notes”，不是自动从 transcript 中抽取所有事实。容量小也是故意的：它迫使模型只保存长期高信号内容。

## Built-in memory 相关 prompt

和 built-in `memory` tool 直接相关的 prompt 可以按“模型何时看到”分成几类。

| Prompt / schema | 路径 | 何时可见 | 作用 |
| --- | --- | --- | --- |
| `MEMORY_GUIDANCE` | [`hermes-agent/agent/prompt_builder.py`](./hermes-agent/agent/prompt_builder.py) | `memory` tool 在 `valid_tool_names` 中时，由 [`system_prompt.py`](./hermes-agent/agent/system_prompt.py) 加入 system prompt | 主系统提示。要求保存 durable facts，包括用户偏好、纠正、环境细节、tool quirks、稳定约定；不要保存 task progress、completed-work logs、PR/issue/commit、临时 TODO；procedure 应进入 skill；memory 要写 declarative facts 而不是命令式自我指令。 |
| `MEMORY_SCHEMA` | [`hermes-agent/tools/memory_tool.py`](./hermes-agent/tools/memory_tool.py) | 作为 function/tool schema 发送给模型 | 最直接的 tool-use prompt。`HOW` 要求优先用 `operations` batch，一次完成 add/remove/replace；`WHEN` 要求在用户表达 preference/correction/personal detail 或稳定环境/约定/工作流事实时主动保存；`TARGETS` 区分 `user` 和 `memory`；`SKIP` 排除 trivial info、raw dumps、task progress 和 procedures。 |
| Stored memory prompt block | [`hermes-agent/tools/memory_tool.py`](./hermes-agent/tools/memory_tool.py) 的 `_render_block()`，注入点在 [`system_prompt.py`](./hermes-agent/agent/system_prompt.py) | system prompt build 时 | 不是写入指导，而是读取指导。已有条目以 `MEMORY (your personal notes)` 和 `USER PROFILE (who the user is)` 标题进入 system prompt，让模型知道两个 store 的语义。 |
| `_MEMORY_REVIEW_PROMPT` | [`hermes-agent/agent/background_review.py`](./hermes-agent/agent/background_review.py) | 后台 memory review fork 触发时 | 真正的后台抽取 prompt。要求回看对话，关注用户透露的 persona、desires、preferences、personal details，以及对 agent 行为、工作风格的期望；有值得保存的内容就用 `memory` tool，否则输出 `Nothing to save.` |
| `_COMBINED_REVIEW_PROMPT` | [`hermes-agent/agent/background_review.py`](./hermes-agent/agent/background_review.py) | memory 和 skill review 同时触发时 | 同时指导 memory 与 skill 更新。Memory 侧保存 “who the user is” 和 durable preferences；skill 侧保存 “how to do this class of task”。这也是 memory/skill 边界最清楚的 prompt。 |
| `_SKILL_REVIEW_PROMPT` 中的 memory/skill 边界 | [`hermes-agent/agent/background_review.py`](./hermes-agent/agent/background_review.py) | 只触发 skill review 时 | 间接相关。它强调用户对风格/流程的纠正是 first-class skill signal，不只是 memory signal；memory 捕获 “who/current state”，skills 捕获 “how”。 |
| `profile_build_directive()` | [`hermes-agent/agent/onboarding.py`](./hermes-agent/agent/onboarding.py) | 用户首次使用时的 onboarding system note | 首次画像构建 prompt。要求 agent 先 offer、用户接受后再收集用户愿意分享的信息；外部查询前必须明确征得同意；只把 confirmed durable facts 用 `memory target="user"` 保存。 |
| `clarify` tool schema | [`hermes-agent/tools/clarify_tool.py`](./hermes-agent/tools/clarify_tool.py) | `clarify` tool schema | 间接相关。允许模型在“想询问是否保存 skill 或更新 memory”时向用户提问，提供 consent/确认路径。 |
| write approval prompt | [`hermes-agent/tools/write_approval.py`](./hermes-agent/tools/write_approval.py) | `memory.write_approval` 开启且 foreground CLI 有 inline approval callback 时 | 用户审批 prompt，不是给模型的抽取 prompt。文案形如 `Save to memory: ...`，用于批准、拒绝或 stage memory 写入。 |

这些 prompt 共同形成 built-in memory 的抽取策略：

```text
foreground path:
  MEMORY_GUIDANCE + MEMORY_SCHEMA
  -> 主模型在正常 tool loop 中判断是否保存
  -> memory(action=add/replace/remove, target=...)

background path:
  memory nudge interval reached
  -> background_review fork
  -> _MEMORY_REVIEW_PROMPT or _COMBINED_REVIEW_PROMPT
  -> memory tool write or "Nothing to save."

first-contact profile path:
  onboarding profile_build_directive()
  -> ask user consent
  -> collect confirmed durable user facts
  -> memory(target="user")

approval path:
  memory.write_approval enabled
  -> inline user approval or pending store
```

需要注意：这些都是 prompt-driven extraction。`MemoryStore` 本身只负责文件读写、容量、安全扫描、漂移防护和 snapshot 渲染；它不会用 embedding、正则或专门算法从 transcript 中自动抽取事实。

## 写入审批与治理

`tools/write_approval.py` 给 memory 和 skills 提供统一 gate：

```yaml
memory:
  write_approval: false  # default

skills:
  write_approval: false
```

决策矩阵：

| 场景 | 行为 |
| --- | --- |
| gate off | 直接写入 |
| gate on + foreground CLI memory + 有 inline callback | 询问用户，批准才写 |
| gate on + gateway/script/background memory | stage 到 `$HERMES_HOME/pending/memory/<id>.json` |
| gate on + skill 写入 | stage 到 pending skills，避免在对话里审阅巨大 diff |

审批通过前，staged memory 不会写入本地文件，也不会通过 `notify_memory_tool_write()` 镜像给外部 provider。这个细节避免外部 provider 记住一个本地 memory 尚未批准的事实。

## 外部 memory provider 协议

外部 provider 通过 `agent/memory_provider.py` 的 `MemoryProvider` ABC 接入。它不是替代内建 memory，而是 additive。当前源码和官方文档都强调：内建 memory 可以继续工作，但一次只能启用一个 external provider。

核心接口：

```text
initialize(session_id, hermes_home, platform, agent_context, identity, ...)
system_prompt_block()     -> static prompt block
prefetch(query)           -> dynamic recalled context for current/next turn
queue_prefetch(query)     -> warm next-turn recall
sync_turn(user, assistant, messages?) -> durable post-turn sync
get_tool_schemas()        -> provider-specific tools
handle_tool_call(name,args)-> tool result JSON/string
shutdown()

optional:
on_turn_start()
on_session_end()
on_session_switch()
on_pre_compress()
on_memory_write()
on_delegation()
backup_paths()
```

`MemoryManager` 是唯一编排层。它负责：

- 限制只注册一个 external provider。
- 收集 static prompt block。
- prefetch fan-out 和异常隔离。
- provider tools schema 注入与路由。
- post-turn `sync_all()` 和 `queue_prefetch_all()` 后台执行。
- session/compression/delegation hooks。
- 将 successful built-in memory writes 镜像给 provider。

`sync_all()` 和 `queue_prefetch_all()` 使用单 worker executor。这样慢 provider 不阻塞用户看到响应，同时保持 turn N 的写入先于 turn N+1。executor 不可用时会 inline fallback，优先保证写入不丢。

## External provider 什么时候读

外部 provider 的自动读取分成静态和动态两类。

静态读取：

```text
build_system_prompt()
  -> memory_manager.build_system_prompt()
  -> provider.system_prompt_block()
```

这个 block 通常只包含 “provider active、工具怎么用、mode 是什么、container/bank/session id 是什么” 等稳定说明。它会随 system prompt 缓存。

动态读取：

```text
turn start
  -> memory_manager.on_turn_start(turn_number, message)
  -> memory_manager.prefetch_all(original_user_message)
  -> provider.prefetch(query)

API request build
  -> build_memory_context_block(raw_context)
  -> append to current user message
```

`prefetch_all()` 会先调用 `_strip_skill_scaffolding()`。如果用户通过 `/skill` 或 bundle 导致模型可见消息里展开了整个 skill body，provider 只看到提取出的用户真实指令，避免把 skill 文档污染进长期 memory。

动态 context 会先经过 `sanitize_context()`，去掉 provider 返回里可能夹带的 `<memory-context>` 标签和内部 system note，再统一包成 `<memory-context>` block。流式输出路径还有 `StreamingContextScrubber`，防止模型把 memory-context 原样流到 UI。

显式读取：

```text
model calls provider tool
  -> tool_executor sees memory_manager.has_tool(name)
  -> memory_manager.handle_tool_call(name, args)
  -> provider.handle_tool_call(...)
  -> normal tool result
```

provider tools 的返回值不会自动写入 system prompt。它只是普通 tool result，被后续 API iteration 看到，也可能随 messages 被 SessionDB 记录。

## External provider 什么时候写

外部 provider 的写入路径比读取更多样：

| 路径 | 触发时机 | 典型用途 |
| --- | --- | --- |
| `sync_turn()` | 主响应完成且未 interrupted | 把 user/assistant turn 交给 provider 做存储、抽取、建模 |
| `queue_prefetch()` | `sync_turn()` 后同一 finalizer | 为下一 turn 预热 recall |
| provider write tools | 模型显式调用 | 保存、删除、更新 provider 内部 memory |
| `on_memory_write()` | 内建 `memory` 工具成功 committed 后 | 把本地 curated memory 镜像给 provider |
| `on_session_end()` | CLI exit、reset、gateway expiry 等真实边界 | session 级 extraction、commit、flush |
| `on_session_switch()` | `/resume`、`/branch`、`/reset`、`/new`、compression | 切换 provider 内部 session id，必要时提交旧 session |
| `on_pre_compress()` | live context compression 前 | 在旧消息被压缩前抽取洞察 |
| `on_delegation()` | parent agent 收到 subagent result 后 | 父 agent 观察 delegation outcome |

`_sync_external_memory_for_turn()` 会跳过 interrupted turns。原因是部分输出、被打断的工具链或 reset 中断不应成为 durable conversational truth。

## Provider 返回的 memory 如何进入 prompt

外部 provider 返回内容有三种进入模型的方式，语义不同：

```text
provider.system_prompt_block()
  -> cached system prompt
  -> static instructions/status only

provider.prefetch(query)
  -> build_memory_context_block()
  -> current user message
  -> turn-scoped dynamic recall

provider.handle_tool_call(...)
  -> normal tool result
  -> loop-scoped explicit recall/management result
```

因此，报告或调试时不要把 “provider 有记忆” 简化成 “provider 把 memory 写进 prompt”。真正的问题应该是：

- 这是 static block、dynamic prefetch，还是 tool result？
- 它是否被缓存？
- 它是否被持久化？
- 它是否会被 provider 下一次 `sync_turn()` 再吸收？

## Provider 工具暴露与调用

外部 provider tool schema 由 `get_tool_schemas()` 提供。`inject_memory_provider_tools(agent)` 会把这些 schema 加到 agent 的 `tools` 列表，并加入 `valid_tool_names`。如果 memory tool/toolset 不可用，provider tools 也不会随便暴露。

这些工具不在普通 registry 中。执行时有专门路由：

```text
tool_executor
  -> if agent._memory_manager and has_tool(function_name)
       memory_manager.handle_tool_call(function_name, args)
```

工具什么时候被调用由模型决定，常见触发包括：

- 用户明确要求 “查以前记得什么/忘掉某条/保存这点”。
- 自动 `<memory-context>` 不足，模型需要更精确搜索。
- provider static block 建议模型使用 profile/search/reasoning 类工具。
- 用户要求管理 provider 中的记忆。

工具结果进入 conversation history 后，可能被本 turn 的下一次 LLM call 看到；turn 结束时，某些 provider 的 `sync_turn(messages=...)` 也可能拿到包含 tool calls/results 的 canonical transcript。是否吸收这些 tool 结果由 provider 自己决定。例如 OpenViking 会把结构化 tool 信息转成自己的 batch payload，并跳过 OpenViking 自身的 recall tool 结果，减少反馈污染。

## Provider 工具清单

源码当前 bundled provider 有 8 个。下表以 `plugins/memory/*/__init__.py` 实际导出的 schema 为准。

| Provider | 工具 | 自动 recall | 自动写入/抽取 | 存储/原理 |
| --- | --- | --- | --- | --- |
| Honcho | `honcho_profile`, `honcho_search`, `honcho_context`, `honcho_reasoning`, `honcho_conclude` | `context`/`hybrid` 模式下 prefetch base context + dialectic supplement | `sync_turn()` 写 messages；user memory add 镜像为 conclusion；session end flush | peer/session/observation model，peer card，representation，dialectic LLM reasoning |
| OpenViking | `viking_search`, `viking_read`, `viking_browse`, `viking_remember`, `viking_forget`, `viking_add_resource` | background search result 进入 `## OpenViking Context` | `sync_turn()` 写 session；`on_session_end()` commit 触发 extraction；memory add 镜像 content/write | filesystem-style `viking://` hierarchy，semantic search，session commit 后抽取 profile/preferences/entities/events/cases/patterns |
| Mem0 | `mem0_list`, `mem0_search`, `mem0_add`, `mem0_update`, `mem0_delete` | `queue_prefetch()` search top_k，platform mode 可 rerank | `sync_turn()` 调 backend.add(messages, infer=True) | Mem0 cloud 或 OSS backend，server-side LLM extraction + semantic search |
| Hindsight | `hindsight_retain`, `hindsight_recall`, `hindsight_reflect` | `recall` 或 `reflect` 预取，按 memory_mode 控制 | `sync_turn()` 通过 retain queue 写入，支持 entity graph/observations | knowledge graph，entity resolution，多策略 recall，reflect synthesis |
| Holographic | `fact_store`, `fact_feedback` | local fact retrieval prefetch | `sync_turn()` 不自动写；`auto_extract` 开启时 session end 抽取；memory add 可镜像 | local SQLite fact store，FTS5，trust score，entity resolution，HRR compositional retrieval |
| RetainDB | `retaindb_profile`, `retaindb_search`, `retaindb_context`, `retaindb_remember`, `retaindb_forget`, `retaindb_upload_file`, `retaindb_list_files`, `retaindb_read_file`, `retaindb_ingest_file`, `retaindb_delete_file` | queue/prefetch task context | `sync_turn()` 写云端；memory write 可镜像 | Cloud memory API，hybrid search，7 memory types，文件 ingest 工具 |
| ByteRover | `brv_query`, `brv_curate`, `brv_status` | `prefetch()` 同步执行 `brv query` | `sync_turn()` 后台 `brv curate`；memory add/replace 镜像；pre-compress curate | local-first CLI knowledge tree，fuzzy/LLM-driven tiered retrieval |
| Supermemory | `supermemory-save` / `supermemory_store`, `supermemory-search` / `supermemory_search`, `supermemory-forget` / `supermemory_forget`, `supermemory-profile` / `supermemory_profile` | `get_profile(query)` 返回 profile facts + search results | `sync_turn()` buffer；`on_session_end()`/switch ingest full conversation；memory add 镜像 explicit memory | Supermemory API，profile/search/graph，container tags，多 container 可选 |

注意：RetainDB 的 README 只列了 5 个 memory 工具，但源码还导出文件相关工具。Supermemory 同时导出 snake_case 与 kebab-case alias。OpenViking 源码导出 6 个工具。

## Memory 生成算法分层

Hermes 自身只定义生命周期与上下文注入方式，很多 “memory 如何生成” 的算法在 provider backend 内。可以按生成时机拆成四类。

### 1. Model-curated durable notes

代表：built-in `memory` tool、provider explicit write tools。

```text
model observes conversation
  -> decides a durable fact is worth saving
  -> calls memory / honcho_conclude / brv_curate / mem0_add / ...
```

优点是可解释、可审计、内容短；缺点是依赖模型判断，可能漏掉隐含事实，也可能保存错误假设。因此 Hermes 配了容量限制、threat scan、write approval、pending store。

### 2. Post-turn extraction

代表：Mem0 `infer=True`、Hindsight `retain`、OpenViking session commit、Supermemory conversation ingest、ByteRover `curate`。

```text
turn completes
  -> provider receives user/assistant/messages
  -> backend extracts facts, observations, profile updates, entities, or patterns
```

优点是用户不用显式要求也能积累记忆；缺点是 backend 算法透明度和可控性取决于 provider，云端 provider 还有隐私、成本和可用性问题。

### 3. Recall-time retrieval/synthesis

代表：Honcho dialectic、Hindsight reflect、Supermemory profile/search、Mem0 search、OpenViking search、ByteRover query。

```text
turn start or explicit tool query
  -> provider searches / reasons / reranks
  -> returns formatted context
  -> Hermes wraps as <memory-context> or tool result
```

优点是 prompt 不必常驻大历史；缺点是召回可能漏、偏、过时，且动态 context 可能带来 prompt injection 风险，所以 Hermes 要加 fence/sanitizer/scrubber。

### 4. Exact transcript recall

代表：`session_search`。

```text
query
  -> SQLite FTS5
  -> real messages around anchor
```

这里没有 memory generation，也没有 LLM 总结。它是长期档案检索，适合查 “之前到底说过什么”，而不是生成用户画像。

## Honcho 深挖

仓库没有用量统计，不能证明哪个 provider “最常用”。如果按官方文档篇幅、配置复杂度、README 细节、工具数和源码实现深度衡量，Honcho 是当前最具代表性的外部 memory provider。

Honcho 的关键不是向量召回，而是 peer/session 建模：

| 概念 | 语义 |
| --- | --- |
| Workspace | 共享环境。多个 Hermes profile 可共享同一用户身份。 |
| User peer | 人类用户。CLI 来自 `peerName`，gateway 可由 runtime user id/alias/pin 解析。 |
| AI peer | 当前 Hermes profile 的 agent 身份，不同 profile 可以有不同 AI peer。 |
| Session | 按工作目录、repo、Hermes session、gateway session key 或 global strategy 解析出的对话桶。 |
| Observation | 控制 user/AI peer 是否观察自己和对方，用于建立 representation/card。 |
| Conclusion | 持久结论，可由 `honcho_conclude` 或 built-in user memory mirror 写入。 |

### Honcho recall mode

| Mode | 自动注入 | Tools | 适合 |
| --- | --- | --- | --- |
| `context` | 是 | 否 | 想让 provider 自动给上下文，不让模型管理 memory |
| `tools` | 否 | 是 | 想降低每 turn 注入成本，模型按需查 |
| `hybrid` | 是 | 是 | 默认高能力模式 |

`system_prompt_block()` 只返回 mode header 和工具说明。`prefetch()` 在 `tools` mode 下返回空。

### Honcho 自动注入的两层结构

Honcho `prefetch()` 组装两层 context：

```text
Layer 1: base context
  - Session Summary
  - User Representation
  - User Peer Card
  - AI Self-Representation
  - AI Identity Card
  refreshed by contextCadence

Layer 2: dialectic supplement
  - Honcho peer.chat() synthesized reasoning
  - cold prompt focuses on user identity/preferences/goals
  - warm prompt focuses on current session relevance
  refreshed by dialecticCadence
```

base context 更像结构化快照，dialectic supplement 是对 “当前 turn 应该记起什么” 的推理合成。两层合并后按 `contextTokens` 做粗略 char budget 截断，再交给 Hermes 包成 `<memory-context>`。

首 turn 时，如果 background prewarm 没有结果，Honcho 会启动 first-turn dialectic，并最多等待一个 bounded timeout。超时不阻塞当前响应，结果留到后续 turn 消费。

### Honcho dialectic depth

`dialecticDepth` 允许 1 到 3 pass：

```text
pass 0:
  cold: infer who this person is, preferences, goals, working style
  warm: infer current-session-relevant user context

pass 1:
  self-audit and targeted synthesis

pass 2:
  reconcile contradictions and produce concise synthesis
```

若早期 pass 已经给出足够强信号，后续 pass 可提前停止。reasoning level 来自 `dialecticDepthLevels`、proportional defaults、`dialecticReasoningLevel` 和 query-length heuristic，再受 `reasoningLevelCap` 限制。

### Honcho tools 什么时候用

| Tool | 适合场景 |
| --- | --- |
| `honcho_profile` | 快速读/写 peer card，回答稳定用户事实 |
| `honcho_search` | 找 raw excerpts，不需要 LLM synthesis |
| `honcho_context` | 查看 session summary、representation、card、recent messages |
| `honcho_reasoning` | 需要 Honcho LLM 对用户/会话进行合成推理 |
| `honcho_conclude` | 保存或删除 persistent conclusion |

显式 `honcho_reasoning` 会影响 `_last_dialectic_turn`，避免自动 dialectic 很快重复触发同类推理。

### Honcho 写入路径

```text
post-turn sync
  -> sanitize_context(user/assistant)
  -> chunk by message_max_chars
  -> session.add_message(user)
  -> session.add_message(assistant)
  -> manager flush

built-in memory mirror
  -> only action=add and target=user
  -> create_conclusion(session_key, content)

session end
  -> wait pending sync
  -> flush_all()
```

这说明 Honcho 同时吸收 conversation turns 和显式 user memory，但不会把所有本地 `MEMORY.md` 环境笔记都写成用户 conclusion。

### Honcho 优缺点

优点：

- 用户、AI、session 都有明确身份模型，适合长期多 profile、多 gateway、多 agent 场景。
- 自动 context 和显式 tools 可通过 `context/tools/hybrid` 切换。
- base context 与 dialectic 分层，成本 knob 明确。
- peer card、representation、conclusion 比单纯向量 search 更接近“用户模型”。

限制：

- 实现复杂，配置项多，错误作用域或 identity mapping 会造成记忆串线。
- dialectic 可能引入 backend LLM 成本和延迟。
- 自动合成结果需要信任 Honcho 的推理质量。
- cloud 模式涉及外部服务可用性、隐私和账单。

## 其他 provider 设计取向

### Supermemory

Supermemory 偏 “profile/search + session-level ingest”。`prefetch()` 直接调用 `get_profile(query)`，按配置决定是否包含 static facts 和 dynamic facts，再合并 search results。`sync_turn()` 只缓冲 turn；`on_session_end()` 和 `on_session_switch()` 把 full session ingest 到 conversations endpoint，驱动 profile/graph 更新。

它适合希望把 Hermes 写入集中到 Supermemory space/container 中管理的用户。优点是 profile/search API 明确、container tag 便于作用域管理；缺点是自动写入集中在 session boundary，session 很长或异常退出时要依赖 shutdown fallback。

### Mem0

Mem0 是典型 “server-side extraction + semantic search”。`sync_turn()` 把 user/assistant messages 发给 backend，并设置 `infer=True`，由 Mem0 负责抽取 memory。`queue_prefetch()` search top_k，platform mode 可以 rerank。

优点是接入简单、工具管理完整；缺点是生成算法在后端，Hermes 只看到结果，调试需要看 Mem0 backend 行为。

### Hindsight

Hindsight 的重点是 knowledge graph、entity resolution 和 observations。它把 raw facts 提炼成 observation 层，默认 recall 类型收窄到 observation，减少把重复证据塞回 prompt。`hindsight_reflect` 提供 LLM synthesis。

优点是结构化知识和多策略 recall 强；缺点是 local embedded 模式依赖 daemon/LLM provider，后台服务故障会影响体验。

### OpenViking

OpenViking 把记忆组织成 `viking://` filesystem-style hierarchy。turn sync 会把 Hermes canonical messages 转成 OpenViking batch；session end commit 后，OpenViking 自动抽取 profile、preferences、entities、events、cases、patterns。工具支持 search/read/browse/remember/forget/add_resource。

优点是层级清晰、可读可浏览、资源 ingestion 丰富；缺点是 session commit 和 writer draining 逻辑复杂，依赖 server 状态。

### ByteRover

ByteRover 通过 `brv` CLI 做 local-first knowledge tree。`prefetch()` 同步执行 `brv query`，保证第一轮 LLM call 前拿到结果；`sync_turn()` 后台 `brv curate`；`on_pre_compress()` 会把即将压缩的消息 curate。

优点是本地优先、profile-scoped 工作目录明确；缺点是同步 query 可能增加 turn-start latency，能力取决于 CLI 与本地 tree 状态。

### Holographic

Holographic 是本地 SQLite fact store，配合 FTS5、trust scoring、entity resolution 和可选 HRR algebra。工具只有 `fact_store` 与 `fact_feedback`，但 `fact_store` 内含 add/search/probe/related/reason/contradict/update/remove/list 等动作。

优点是本地、可控、轻量；缺点是相比 Honcho/Hindsight/Supermemory，自动用户建模和云端 profile 能力弱，需要更显式的事实管理。

### RetainDB

RetainDB 是云端 hybrid search provider，README 强调 Vector + BM25 + reranking 和 7 memory types。源码除了 profile/search/context/remember/forget，还提供文件 upload/list/read/ingest/delete 工具。

优点是 search 与文件 ingestion 能力集中；缺点是商业云服务依赖强，工具面较宽，对模型的工具选择纪律要求更高。

## session_search 不是 provider

`session_search` 经常会被误认为 memory provider，但它是 SessionDB transcript recall 工具。它不参与 `MemoryProvider` 生命周期，不做自动 prefetch，也不会返回用户画像。

三种调用形态：

```text
DISCOVERY:
  query=...
  -> FTS5 search
  -> top sessions with snippet, anchored window, bookends

SCROLL:
  session_id + around_message_id
  -> centered message window

BROWSE:
  no args
  -> recent session metadata
```

它的优势是 “真实发生过什么” 可追溯。memory provider 的优势是 “现在应该记起什么” 更浓缩。二者互补：

| 需求 | 更适合 |
| --- | --- |
| 用户偏好、长期工作方式、稳定环境约定 | built-in memory / provider profile |
| 上次某个具体命令输出、某次 PR 讨论、旧会话完整上下文 | `session_search` |
| 当前 turn 前自动补充相关用户画像 | provider prefetch |
| 让模型显式搜索/删除/管理长期记忆 | provider tools |

## Context compression 与 memory

Compression 不等于遗忘。它影响 live context，但长期 recall 仍有几条路径：

```text
before compression
  -> memory_manager.on_pre_compress(messages)
     provider may extract/curate insights

compression creates/switches session state
  -> memory_manager.on_session_switch(new_session_id, ...)
     provider rotates internal session/cache

SessionDB
  -> archive_and_compact()
     old messages inactive/compacted but still searchable
```

内建 memory 也会在 `invalidate_system_prompt()` 时重新 `load_from_disk()`，所以本 session 中已经写盘的 memory 有机会在 compression 后进入 rebuilt prompt。

## Profile、gateway、subagent 和 cron 隔离

内建 memory 使用 `get_hermes_home()` 动态定位，因此 profile 切换会改变 `memories/` 路径。外部 provider 初始化时会收到大量作用域信息：

```text
hermes_home
platform
agent_context
agent_identity
agent_workspace
parent_session_id
user_id / user_id_alt / user_name
chat_id / chat_name / chat_type / thread_id
gateway_session_key
```

provider 应用这些字段来隔离 profile、gateway user、chat/thread 和 session。Honcho 的 peer resolver、Supermemory 的 container tag、Hindsight 的 bank template、RetainDB project、OpenViking account/user/agent 都是这一层的具体实现。

Subagent 默认不共享父 agent 的 memory 写权限。父 agent 只在子任务完成后通过 `on_delegation()` 观察 task/result。Cron/flush 等非 primary context 也应避免污染用户画像，多个 provider 在初始化或运行时会跳过这类上下文。

## 安全与失效模式

Hermes memory system 的主要风险和防护如下：

| 风险 | 防护 |
| --- | --- |
| memory prompt injection 长期污染 system prompt | write-time strict scan + load-time snapshot scan |
| external provider 返回恶意 fenced content | `sanitize_context()` 去标签，统一 `<memory-context>` 包装 |
| 模型把 memory context 原样输出给用户 | `StreamingContextScrubber` 跨 chunk 清洗 |
| 多 session 或人工编辑覆盖 memory 文件 | file lock + external drift guard + `.bak.<ts>` |
| 自动学习保存错误假设 | `write_approval` + pending store |
| provider 慢或故障拖住主响应 | background executor + fail-open |
| 多 provider 工具/事实冲突 | 只允许一个 external provider |
| skill-expanded prompt 污染 memory backend | `_strip_skill_scaffolding()` |
| interrupted turn 被写入长期 memory | `_sync_external_memory_for_turn(interrupted=True)` skip |
| dynamic recall 破坏 prompt cache | recall 注入 user message，不改 cached system prompt |

一个仍然存在的系统性风险是：外部 provider backend 自身的抽取、合成和删除语义并不完全由 Hermes 控制。对云端 provider，用户需要额外考虑隐私、合规、成本、可用性和可观测性。

## 操作规则

分析或调试 Hermes memory 时，建议按这个顺序定位：

1. 先判断信息通道：built-in snapshot、provider static block、provider dynamic prefetch、tool result、SessionDB archive、还是 background review。
2. 如果用户问 “为什么当前 turn 没看到刚写的 memory”，先检查 frozen snapshot 语义。写盘不等于当前 cached prompt 刷新。
3. 如果用户问 “provider 返回的 memory 怎么进 prompt”，检查 `prefetch()` 和 `build_memory_context_block()`，不是只看 `system_prompt_block()`。
4. 如果用户问 “什么时候会写外部 memory”，检查 turn 是否 interrupted、`sync_turn()` 是否被调用、provider 是否在 background executor 中失败。
5. 如果 memory 内容来自 `/skill` turn，检查 `_strip_skill_scaffolding()` 是否提取到了真实用户指令。
6. 如果 provider 工具没有出现，检查 external provider 是否 active、memory tool/toolset 是否启用、tool name 是否被 core tool shadow 过滤。
7. 如果要证明历史事实，优先用 `session_search` 找 transcript；不要只信 provider 合成的 profile。
8. 如果要解释 “最常用 provider”，不要声称有用量统计。源码能支持的说法是：Honcho 是当前最突出、最完整、最适合作为代表案例的 provider。

## 源码地图

| 模块 | 作用 |
| --- | --- |
| [`hermes-agent/tools/memory_tool.py`](./hermes-agent/tools/memory_tool.py) | 内建 `MEMORY.md` / `USER.md` 文件存储、冻结 snapshot、容量、batch、漂移防护、threat scan |
| [`hermes-agent/agent/prompt_builder.py`](./hermes-agent/agent/prompt_builder.py) | `MEMORY_GUIDANCE` 等 system prompt guidance |
| [`hermes-agent/agent/system_prompt.py`](./hermes-agent/agent/system_prompt.py) | stable/context/volatile prompt 分层；内建 memory 和 provider static block 注入 cached system prompt |
| [`hermes-agent/agent/turn_context.py`](./hermes-agent/agent/turn_context.py) | turn prologue；on_turn_start；external prefetch |
| [`hermes-agent/agent/conversation_loop.py`](./hermes-agent/agent/conversation_loop.py) | 把 external prefetch 包成 `<memory-context>` 并追加到当前 user message |
| [`hermes-agent/agent/onboarding.py`](./hermes-agent/agent/onboarding.py) | 首次 profile build directive，引导确认后写 `target="user"` |
| [`hermes-agent/agent/memory_provider.py`](./hermes-agent/agent/memory_provider.py) | provider ABC 和生命周期契约 |
| [`hermes-agent/agent/memory_manager.py`](./hermes-agent/agent/memory_manager.py) | provider 编排、单 external 限制、tool routing、background sync、context sanitize/scrub |
| [`hermes-agent/agent/tool_executor.py`](./hermes-agent/agent/tool_executor.py) | built-in `memory` 工具执行、provider tools 路由、写入镜像 |
| [`hermes-agent/run_agent.py`](./hermes-agent/run_agent.py) | session boundary、turn end `_sync_external_memory_for_turn()`、provider shutdown/commit |
| [`hermes-agent/agent/background_review.py`](./hermes-agent/agent/background_review.py) | 响应后 memory/skill review fork 和 review prompts |
| [`hermes-agent/tools/clarify_tool.py`](./hermes-agent/tools/clarify_tool.py) | 允许模型询问是否更新 memory/skill 的确认工具 schema |
| [`hermes-agent/tools/write_approval.py`](./hermes-agent/tools/write_approval.py) | memory/skill 写入审批和 pending store |
| [`hermes-agent/tools/session_search_tool.py`](./hermes-agent/tools/session_search_tool.py) | SQLite transcript discovery/scroll/browse 工具 |
| [`hermes-agent/hermes_state.py`](./hermes-agent/hermes_state.py) | SessionDB、messages、FTS5/trigram index、compression archive |
| [`hermes-agent/plugins/memory/honcho`](./hermes-agent/plugins/memory/honcho) | Honcho peer/session/dialectic provider |
| [`hermes-agent/plugins/memory/supermemory`](./hermes-agent/plugins/memory/supermemory) | Supermemory profile/search/session ingest provider |
| [`hermes-agent/plugins/memory/openviking`](./hermes-agent/plugins/memory/openviking) | OpenViking filesystem-style knowledge hierarchy provider |
| [`hermes-agent/plugins/memory/hindsight`](./hermes-agent/plugins/memory/hindsight) | Hindsight knowledge graph provider |
| [`hermes-agent/plugins/memory/mem0`](./hermes-agent/plugins/memory/mem0) | Mem0 extraction/search provider |
| [`hermes-agent/plugins/memory/byterover`](./hermes-agent/plugins/memory/byterover) | ByteRover CLI knowledge tree provider |
| [`hermes-agent/plugins/memory/holographic`](./hermes-agent/plugins/memory/holographic) | Local SQLite fact store provider |
| [`hermes-agent/plugins/memory/retaindb`](./hermes-agent/plugins/memory/retaindb) | RetainDB cloud hybrid search provider |

## 权威资料入口

- [`ref/INDEX.md`](./ref/INDEX.md)：本 topic 的官方和权威参考资料索引。
- [`hermes-agent/website/docs/user-guide/features/memory-providers.md`](./hermes-agent/website/docs/user-guide/features/memory-providers.md)：官方文档里的 provider 总览。
- [`hermes-agent/plugins/memory/honcho/README.md`](./hermes-agent/plugins/memory/honcho/README.md)：Honcho provider 的详细架构说明。
- [`hermes-agent/website/docs/developer-guide/memory-provider-plugin.md`](./hermes-agent/website/docs/developer-guide/memory-provider-plugin.md)：provider plugin 开发接口说明。
