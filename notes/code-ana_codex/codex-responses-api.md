# Codex Responses API 使用与 Prompt Cache 设计分析

本文分析 `openai/codex` 当前源码中 Codex 如何使用 OpenAI Responses API，利用了哪些 API 特性做运行时优化，以及它围绕 Responses API 的 `prompt_cache_key` 和前缀复用做了怎样的 prompt cache 设计。分析以本 topic submodule 的当前源码为准。

## 核心结论

Codex 的模型调用是 Responses-first 的 agent runtime。它不是把一段聊天文本扔给模型，而是把 thread history、工具调用、reasoning、图片、compaction、metadata 和 token 账本都组织成 Responses API 的结构化请求与流式事件。

1. **Responses API 是 core 的模型边界**：一次 sampling request 由 [`run_turn`](./codex/codex-rs/core/src/session/turn.rs) 构造 `Prompt`，交给 [`ModelClientSession::stream`](./codex/codex-rs/core/src/client.rs)，再走 HTTP SSE 或 Responses WebSocket。
2. **请求体高度结构化**：Codex 使用 `instructions`、`input: Vec<ResponseItem>`、`tools`、`tool_choice`、`parallel_tool_calls`、`reasoning`、`include`、`text`、`prompt_cache_key`、`client_metadata` 等字段，而不是退化成 Chat Completions 风格的 message list。
3. **优化分两类**：传输层优化包括 WebSocket prewarm/preconnect、`previous_response_id` 增量请求、HTTP fallback、zstd 压缩；语义层优化包括 Responses Lite、reasoning encrypted content、structured output、remote compaction、deferred/namespace tool specs、metadata routing。
4. **Prompt cache 的中心策略是“稳定 key + 稳定前缀 + 追加式变化”**：默认 `prompt_cache_key` 是 thread id；上下文、权限、环境、用户指令等被组织成稳定前缀；模型/权限等变化尽量作为后续 history item 追加，而不是改写旧 prefix。
5. **`previous_response_id` 不是 prompt cache，但与 prompt cache 互补**：prompt cache 依赖服务器对请求前缀的缓存；WebSocket 增量依赖上一轮 `response.id`，只发送新增 input delta。两者都服务于减少重复上下文成本，但边界和失败回退不同。
6. **缓存效果被纳入账本**：Responses 完成事件里的 `input_tokens_details.cached_tokens` 被解析为 `cached_input_tokens`，再进入 UI token count、analytics、compaction 和 goal budget 的用量计算。

一句话概括：Codex 把 Responses API 当作“可状态化、可工具化、可流式、可缓存”的 agent protocol，而 prompt cache 设计的核心是让同一 thread 的请求长期保持可复用前缀，同时用 WebSocket 增量和 compaction 减少必须重新发送或重新计入的上下文。

## 源码地图

| 主题 | 关键文件 | 作用 |
| --- | --- | --- |
| core 模型客户端 | [`core/src/client.rs`](./codex/codex-rs/core/src/client.rs) | 构造 Responses 请求；选择 HTTP/WebSocket；prewarm、incremental、fallback、headers、prompt cache key。 |
| Prompt 抽象 | [`core/src/client_common.rs`](./codex/codex-rs/core/src/client_common.rs) | `Prompt` 字段和 Responses Lite 输入预处理。 |
| turn 内模型循环 | [`core/src/session/turn.rs`](./codex/codex-rs/core/src/session/turn.rs) | 从 history 构造 prompt；处理 stream event；执行工具；记录 token usage。 |
| history 管理 | [`core/src/context_manager/history.rs`](./codex/codex-rs/core/src/context_manager/history.rs) | 追加/规范化 Responses item，截断工具输出，按模型能力剥离图片。 |
| Responses API wire 类型 | [`codex-api/src/common.rs`](./codex/codex-rs/codex-api/src/common.rs) | `ResponsesApiRequest`、`ResponseCreateWsRequest`、`ResponseEvent`、reasoning/text controls。 |
| HTTP SSE transport | [`codex-api/src/endpoint/responses.rs`](./codex/codex-rs/codex-api/src/endpoint/responses.rs)、[`codex-api/src/sse/responses.rs`](./codex/codex-rs/codex-api/src/sse/responses.rs) | 发送 `/responses`，解析 SSE，提取 usage/cache/turn-state/rate-limit。 |
| WebSocket transport | [`codex-api/src/endpoint/responses_websocket.rs`](./codex/codex-rs/codex-api/src/endpoint/responses_websocket.rs) | 连接 Responses WebSocket，发送 `response.create`，解析事件和 turn-state。 |
| Responses item 模型 | [`protocol/src/models.rs`](./codex/codex-rs/protocol/src/models.rs) | `ResponseItem` / `ResponseInputItem` / multimodal content / tool output / reasoning / compaction。 |
| 工具 spec 转换 | [`tools/src/responses_api.rs`](./codex/codex-rs/tools/src/responses_api.rs) | Codex/MCP/dynamic tools 转 Responses API function/namespace/deferred tool specs。 |
| metadata | [`core/src/responses_metadata.rs`](./codex/codex-rs/core/src/responses_metadata.rs) | `client_metadata`、兼容 headers、turn metadata、reserved key 过滤。 |
| retry/fallback | [`core/src/responses_retry.rs`](./codex/codex-rs/core/src/responses_retry.rs) | retryable stream error、WebSocket 到 HTTP fallback、用户可见 reconnect。 |
| remote compaction v2 | [`core/src/compact_remote_v2.rs`](./codex/codex-rs/core/src/compact_remote_v2.rs) | 通过同一 Responses stream 请求 compaction output，并替换 history。 |
| prompt cache 测试 | [`core/tests/suite/prompt_caching.rs`](./codex/codex-rs/core/tests/suite/prompt_caching.rs) | 验证前缀稳定、`prompt_cache_key` 跨设置覆盖保持不变。 |
| Responses Lite 测试 | [`core/tests/suite/responses_lite.rs`](./codex/codex-rs/core/tests/suite/responses_lite.rs) | 验证 Lite 下 instructions/tools 移入 input、header、reasoning context、parallel 禁用。 |
| WebSocket 测试 | [`core/tests/suite/client_websockets.rs`](./codex/codex-rs/core/tests/suite/client_websockets.rs) | 验证 `previous_response_id`、delta input、非 input 字段变化时回退完整请求。 |
| 调试代理 | [`responses-api-proxy`](./codex/codex-rs/responses-api-proxy) | 只转发 `POST /v1/responses` 的本地 proxy，用于以 API key 直连和 dump 请求。 |

## 主链路：从 turn 到 Responses stream

一次普通 sampling request 的路径如下：

```text
run_turn
  -> clone_history().for_prompt(...)
  -> built_tools(...)
  -> build_prompt(...)
  -> ModelClientSession::stream(...)
       -> build_responses_request(...)
       -> WebSocket response.create 或 HTTP POST /responses
       -> map_response_stream(...)
  -> try_run_sampling_request 消费 ResponseEvent
       -> output item / delta / completed
       -> 工具执行与工具输出写回 history
       -> token usage 写入 session 账本
```

[`run_turn`](./codex/codex-rs/core/src/session/turn.rs) 每次采样前从 `ContextManager` 克隆 history，并调用 `for_prompt` 得到模型可见的 `Vec<ResponseItem>`。这个过程会确保工具 call/output 成对、删除孤儿 output，并按模型 `input_modalities` 去掉不支持的图片。随后 `built_tools` 生成当前 step 的 model-visible tool specs，`build_prompt` 组合出：

- `input`：结构化 Responses history。
- `tools`：本轮模型可见工具。
- `parallel_tool_calls`：来自 model info。
- `base_instructions`：线程/模型/personality 解析后的基础指令。
- `output_schema` 和 strict 配置：用于 Responses `text.format`。

[`ModelClientSession::stream`](./codex/codex-rs/core/src/client.rs) 在 `WireApi::Responses` 下优先走 WebSocket，如果 provider 不支持或本 session 已进入 fallback，就走 HTTP SSE。两条 transport 都共享同一个 `ResponsesApiRequest` 构造逻辑，所以功能特性和 prompt cache 策略保持一致。

## Responses 请求形状

[`ResponsesApiRequest`](./codex/codex-rs/codex-api/src/common.rs) 是 Codex 对 `/responses` 的核心请求模型：

| 字段 | Codex 用法 |
| --- | --- |
| `model` | 当前 turn 解析后的 model slug。 |
| `instructions` | 普通 Responses 模式下放 base instructions；Responses Lite 下为空。 |
| `input` | `Vec<ResponseItem>`，包含用户消息、assistant 输出、reasoning、工具调用/输出、context、compaction 等。 |
| `tools` | 普通模式下为 Responses API tool specs；Lite 下为 `None`。 |
| `tool_choice` | 固定 `"auto"`。 |
| `parallel_tool_calls` | 模型支持且非 Lite 时才允许。 |
| `reasoning` | effort、summary、Lite 下的 `context=all_turns`。 |
| `store` | Azure Responses endpoint 下为 true；也影响 item id 保留策略。 |
| `stream` | 固定 true。 |
| `include` | reasoning 存在时加入 `reasoning.encrypted_content`。 |
| `service_tier` | 由 model info 和 turn config 合并。 |
| `prompt_cache_key` | 默认 thread id，可被特殊 session 覆盖。 |
| `text` | verbosity 和可选 JSON schema output format。 |
| `client_metadata` | Codex installation/session/thread/turn/window/subagent/metadata 等非 prompt 控制面信息。 |

`ResponseItem` 的类型很丰富，定义在 [`protocol/src/models.rs`](./codex/codex-rs/protocol/src/models.rs)。Codex 会把以下内容都当作 Responses history item：

- `message`：user/developer/assistant 文本或图片。
- `agent_message`：多 agent 之间的作者/接收者消息，可包含 encrypted content。
- `reasoning`：summary、raw content、encrypted content。
- `function_call` / `function_call_output`。
- `custom_tool_call` / `custom_tool_call_output`。
- `tool_search_call` / `tool_search_output`。
- `local_shell_call`。
- `web_search_call`、`image_generation_call`。
- `compaction`、`context_compaction`、`compaction_trigger`。
- `additional_tools`：Responses Lite 用于把工具 spec 放进 input 前缀。

这使得 Codex 可以把“工具调用已经发生过、工具输出是什么、reasoning 怎么续、哪些内容被 compact 过”都作为结构化 history 交给模型，而不是在文本里手工拼接。

## Responses API 特性利用

### 1. 流式事件驱动 turn 内状态机

HTTP SSE 和 WebSocket 最终都被映射成 [`ResponseEvent`](./codex/codex-rs/codex-api/src/common.rs)：

- `Created`
- `OutputItemAdded`
- `OutputTextDelta`
- `ToolCallInputDelta`
- `ReasoningSummaryDelta`
- `ReasoningContentDelta`
- `OutputItemDone`
- `Completed`
- `RateLimits`
- `ServerModel`
- `ModelVerifications`
- `TurnModerationMetadata`
- `SafetyBuffering`
- `ServerReasoningIncluded`
- `ModelsEtag`

[`try_run_sampling_request`](./codex/codex-rs/core/src/session/turn.rs) 消费这些事件时做三件事：

1. streaming UI：在 `OutputItemAdded` 后发 item started，在 text/reasoning/tool input delta 到达时发增量事件。
2. agent loop：在 `OutputItemDone` 后把完整 item 交给 `handle_output_item_done`，决定是否执行工具、是否需要 follow-up sampling。
3. token 账本：在 `Completed` 后写入 token usage，并触发 `TokenCount` 和 turn diff。

这就是 Codex 能边生成、边展示、边准备工具调用参数 diff、边等待最终 item 落地的原因。

### 2. Reasoning summary 与 encrypted content

当 `model_info.supports_reasoning_summaries` 为 true，Codex 会构造 Responses `reasoning` 字段。`effort=ultra` 会被映射成自定义 `"max"`；summary 按配置传递；普通 Responses 模式省略 `context`，让服务端使用默认 current turn；Responses Lite 则显式设置 `context=all_turns`。

只要启用 reasoning，Codex 就把 `include` 设置为 `["reasoning.encrypted_content"]`。对应的 `ResponseItem::Reasoning` 保存 `encrypted_content`，供后续请求延续推理上下文，同时 UI 只展示 summary 或配置允许的 raw content。

HTTP/WebSocket 响应还可能带 `x-reasoning-included`，底层解析成 `ServerReasoningIncluded(true)`。`ContextManager` 计算 total token usage 时会据此决定是否本地估算历史 reasoning tokens，避免重复估算。

### 3. Tool specs、namespace 和 deferred loading

工具 spec 的转换集中在 [`tools/src/responses_api.rs`](./codex/codex-rs/tools/src/responses_api.rs)。Codex 将本地工具、dynamic tools、MCP tools 转成 Responses API function tool，并支持：

- namespace tool specs：同 namespace 的工具会 coalesce。
- deferred MCP tools：`defer_loading=true`，减少首轮工具 spec 负担。
- dynamic tools：运行时注入的 tool spec 转 Responses function。
- output schema：在内部 `ToolSpec` 中存在，但当前序列化路径跳过。

普通 Responses 模式下，工具作为顶层 `tools` 发送。Responses Lite 下，工具被包装为 `ResponseItem::AdditionalTools { role: "developer" }` 插入 `input` 最前面，顶层 `tools` 变成 `None`。

### 4. Structured output 与 verbosity

`create_text_param_for_request` 会把 Codex 的 verbosity 和最终输出 JSON schema 合并成 Responses `text` 字段：

- `text.verbosity`：仅当模型支持 verbosity 时发送。
- `text.format`：当 `final_output_json_schema` 存在时发送，名称固定为 `codex_output_schema`，strict 由上下文决定。

这让 Codex 不需要把 JSON schema 全部写进 prompt 文本，而是使用 Responses API 的结构化输出控制面。

### 5. Multimodal history

`ContentItem` 支持 `input_image`，本地图片会读取、压缩/转 data URL，并可带 `detail`。工具输出也可以使用结构化 content items，包括 text、image、encrypted content。

Codex 有两层降级：

- `ContextManager::for_prompt` 会在模型不支持 image modality 时剥离图片。
- Responses Lite 会调用 `strip_image_details`，保留图片但去掉 `detail` 字段，以符合 Lite transport contract。

### 6. Metadata 与 sticky routing

Codex 把运行时元数据分成两层：

- `client_metadata`：installation/session/thread/turn/window、request kind、subagent、turn metadata 等。
- HTTP/WebSocket headers：兼容投影，例如 `x-codex-turn-metadata`、`x-codex-parent-thread-id`、`x-openai-subagent`、`x-codex-window-id`。

[`responses_metadata.rs`](./codex/codex-rs/core/src/responses_metadata.rs) 维护这些字段，并过滤 app-server 传入的 reserved metadata key，避免外部 client 覆盖 Codex-owned 字段。

另一个关键 header 是 `x-codex-turn-state`。HTTP SSE、compact endpoint 和 WebSocket 都会从响应 header 或 wrapped event 里捕获该值，并在同一 turn 后续请求中回放，用于服务端 sticky routing。它是 per-turn 状态，不是模型可见 prompt。

## 性能和可靠性优化

### 1. WebSocket preconnect / prewarm

Codex 有两种 WebSocket 预热：

- `preconnect_websocket`：只建立连接，不发送 prompt payload。
- `prewarm_websocket`：发送 v2 `response.create`，并设置 `generate=false`，等待 completed 后复用连接和 `previous_response_id`。

startup prewarm 在 [`session_startup_prewarm.rs`](./codex/codex-rs/core/src/session_startup_prewarm.rs) 中构造一个空 input 的 prompt，但带完整工具和 base instructions。这样第一轮真实 turn 可以尽量复用已建立的连接和服务端状态，降低首 token 前延迟。

### 2. `previous_response_id` 增量请求

WebSocket 路径的核心优化是 [`prepare_websocket_request`](./codex/codex-rs/core/src/client.rs)。它在满足这些条件时，把完整 `response.create` 压缩成：

```text
previous_response_id = 上次 response.id
input = 只包含新增 ResponseItem delta
```

条件非常保守：

1. 上次 response 已 completed，并拿到 `response_id` 和输出 items。
2. 当前请求的非 input 字段与上次请求一致：model、instructions、tools、tool_choice、parallel、reasoning、store、stream、include、service_tier、prompt_cache_key、text 都要匹配。
3. 当前 input 必须以前一次 request input 为前缀。
4. 去掉上次 request input 后，还要先去掉上次 response output items，剩下的才是本次 incremental input。

测试 [`client_websockets.rs`](./codex/codex-rs/core/tests/suite/client_websockets.rs) 覆盖了三种边界：prefix 匹配时发送 `previous_response_id`，非 input 字段变更时不发送，stream 出错后下一次回到完整请求。

这个机制和 prompt cache 的关系是：

- prompt cache 让服务器缓存长 prefix 的计算结果。
- `previous_response_id` 让 WebSocket 后续请求连长 prefix 本身都不必重复发送。
- 失败或条件不满足时，Codex 回退完整请求，仍可依赖 prompt cache key 和稳定前缀。

### 3. HTTP fallback 和 retry

[`handle_retryable_response_stream_error`](./codex/codex-rs/core/src/responses_retry.rs) 处理 stream 断开、retryable error 和 WebSocket fallback：

- 未超过 `stream_max_retries` 时按 backoff 重试。
- release build 下第一个 WebSocket retry 默认不打扰用户，减少 transient reconnect 噪声。
- 如果 retry 用尽且 WebSocket 仍启用，则 session-scoped 切到 HTTP transport，发 warning，重置 retry 计数。

`ModelClientSession::stream` 的选择顺序也体现了这一点：优先 WebSocket，收到 `UPGRADE_REQUIRED` 或 fallback 标记后走 HTTP SSE。

### 4. 请求压缩

HTTP Responses 请求可启用 zstd 压缩，条件是：

- feature `EnableRequestCompression` 打开。
- auth 使用 Codex backend。
- provider 是 OpenAI。

这主要优化大 history、大工具 spec、大图片/tool output 场景下的上传成本。WebSocket 增量则从“少发 input”层面解决重复上下文传输。

### 5. Responses Lite

Responses Lite 是 Codex 对一类模型/provider 的特殊 transport contract，不是简单“少发字段”。它的变化包括：

- 顶层 `instructions` 为空。
- 顶层 `tools` 为 `None`。
- `AdditionalTools` developer item 插入 input 前缀。
- base instructions 作为 developer message 插入 input 前缀。
- `parallel_tool_calls=false`，即使 model info 支持 parallel。
- `reasoning.context=all_turns`。
- HTTP header `x-openai-internal-codex-responses-lite: true`，WebSocket client_metadata 中也标记 Lite。
- 去掉图片 detail。

这会改变 prompt cache 的前缀形态：普通模式的稳定内容分布在顶层 `instructions/tools` 和 input 中；Lite 模式的稳定内容被显式移动到 input 前缀里。因此 Lite 下更需要保持 input 前几项稳定。

### 6. Remote compaction

Codex 有本地/远端 compaction 多条路径。与 Responses API 关系最密切的是 remote compaction v2：

```text
history.for_prompt(...)
  -> input.push(CompactionTrigger {})
  -> client_session.stream(...)
  -> 收集唯一 ResponseItem::Compaction
  -> build_v2_compacted_history(...)
  -> replace_compacted_history(...)
```

它仍走同一个 `ModelClientSession::stream`，所以会继承 `prompt_cache_key`、reasoning、tools、text、metadata、retry/fallback 等机制。完成事件里的 `cached_input_tokens` 还会进入 compaction analytics。

compaction 对 prompt cache 的意义不是提升旧 prefix 命中，而是当 thread 太长时重建一个更短的新 context window，避免上下文窗口和网络/计费成本失控。它会改变后续请求的 prefix，因此 Codex 把 compaction 作为必要的窗口切换，而不是普通 turn 的小优化。

### 7. Item ID 策略

`prepare_response_items_for_request` 默认会清除 input item 上的 Responses item id，除非：

- feature `ItemIds` 打开。
- 或请求 `store=true`，例如 Azure Responses endpoint。

这样可以减少无用或 provider 不接受的 server item id 被带回请求。对 prompt cache 来说，这也有助于减少旧响应 id 这类易变字段污染模型输入。

## Prompt cache 设计

### 设计目标

Codex 面临的 prompt cache 问题不是“某一轮如何加一个 key”这么简单，而是 agent runtime 的请求有这些变化源：

- 用户每轮新增消息。
- 工具调用和工具输出不断追加。
- 权限、cwd、environment、model、reasoning、approval policy 可能变化。
- tools 列表可能随 MCP、plugin、dynamic tools、feature gate 变化。
- reasoning encrypted content、compaction、图片和大工具输出会让 history 变大。
- WebSocket/HTTP/Responses Lite/Azure store 等 transport contract 不完全相同。

因此 Codex 的策略是：

1. 为同一复用边界使用稳定 `prompt_cache_key`。
2. 尽量让长而稳定的内容出现在请求最前面。
3. 对变化内容采用追加式记录，而不是改写老 prefix。
4. 当 prefix 无法继续增长时，用 compaction 开新窗口。
5. 用 `cached_input_tokens` 反馈验证和记账，而不在客户端猜测命中。

### Cache key：默认 thread id，特殊场景覆盖

[`ModelClient::prompt_cache_key`](./codex/codex-rs/core/src/client.rs) 默认返回 `thread_id.to_string()`，并写入每个 Responses 请求的 `prompt_cache_key`。测试 [`core/tests/suite/client.rs`](./codex/codex-rs/core/tests/suite/client.rs) 验证了普通请求 body 中的 key 等于 thread id。

特殊场景可以调用 `with_prompt_cache_key_override`。当前典型例子是 Guardian review session：[`prompt_cache_key_override_for_review_session`](./codex/codex-rs/core/src/guardian/review_session.rs) 对 guardian 子线程返回 `guardian:{parent_thread_id}`。这说明 Codex 的 cache scope 按“共享稳定前缀的业务边界”设计，而不是机械等于当前 session id。

### 稳定前缀：普通 Responses 模式

普通 Responses 模式下，稳定内容主要分成两部分：

- 顶层字段：`instructions` 和 `tools`。
- `input` 前缀：初始权限/context/user instruction/environment，以及历史较早的 conversation items。

[`prompt_caching.rs`](./codex/codex-rs/core/tests/suite/prompt_caching.rs) 明确验证了：

- 第一轮 request 的 `instructions` 和 `tools` 在第二轮保持一致。
- 第二轮 input 以前一轮 input 为前缀，再追加第二轮用户消息。
- thread settings override 后，旧 prefix 不变，新权限/环境/模型切换说明追加在后面。
- per-turn override 同样保持旧 prefix 和 `prompt_cache_key` 不变。

这套测试很关键：它没有证明服务端一定命中 cache，但证明 Codex 发出的请求满足 prefix cache 的前提。

### 稳定前缀：Responses Lite 模式

Lite 模式把 `instructions` 和 `tools` 移到 input 前缀：

```text
input[0] = AdditionalTools(role="developer", tools=[...])
input[1] = developer message(base_instructions)
input[2..] = 原始 conversation history
```

对应测试 [`responses_lite_uses_input_items_for_instructions_and_tools`](./codex/codex-rs/core/tests/suite/responses_lite.rs) 验证顶层 `instructions/tools` 被省略，工具和指令都进入 input。这样 Lite 的 prompt cache 更依赖 input 前缀稳定；Codex 通过统一的 `build_responses_request` 保证这一点。

### 追加式变化：不重写旧 prefix

`ContextManager` 的 history 是 oldest-to-newest 的 `Vec<ResponseItem>`。`record_conversation_items` 只记录 API message，并按 truncation policy 处理工具输出；`record_step_world_state_if_changed` 和 context update 逻辑会把环境/权限变化渲染成新的 contextual items 追加到 history。

这解释了 prompt cache 测试里 override 的布局：当权限或 cwd 变化时，Codex 不去修改第一轮的环境消息，而是在第二轮旧 prefix 后追加一个新的 permissions/context message，再追加用户消息。这样旧 prefix 仍可被服务器 cache 复用。

### 哪些字段会破坏 WebSocket 增量，但不一定破坏 prompt cache key

`responses_request_properties_match` 对 WebSocket incremental 非常严格。只要 model、instructions、tools、reasoning、service_tier、prompt_cache_key、text 等字段不同，就不能使用 `previous_response_id` delta。

但这不意味着 prompt cache key 会变化。比如 per-turn model override 的测试仍要求 `prompt_cache_key` 不变，只是由于 model 或 reasoning 等字段不同，服务端能否复用、复用多少，由 Responses API 的缓存规则决定。Codex 的设计选择是：key 表示 thread 级缓存域，具体命中由稳定前缀和服务器判定。

### `cached_input_tokens`：缓存命中的反馈路径

Responses SSE `response.completed` 中的 usage 会被解析：

```text
usage.input_tokens
usage.input_tokens_details.cached_tokens -> cached_input_tokens
usage.output_tokens
usage.output_tokens_details.reasoning_tokens -> reasoning_output_tokens
usage.total_tokens
```

之后 `cached_input_tokens` 进入：

- `TokenUsage` 和 `TokenUsageInfo`。
- UI token count：显示 non-cached input + output，并把 cached input 单独展示。
- session telemetry / analytics。
- compaction analytics。
- goal mode 的 token delta 计算。

这说明 Codex 不在客户端自行判断 prompt cache 命中，而是把服务器返回的 cached token 数作为权威反馈。

## 可以抽象出的 prompt cache 设计原则

如果围绕 Responses API 设计类似 Codex 的 prompt cache，可以借鉴这些规则：

1. **cache key 选 conversation scope**：普通 thread 用 thread id；如果多个子会话共享同一长前缀，就显式 override 到父级或业务级 key。
2. **把稳定大块放到请求前部**：base instructions、工具 spec、长期上下文、初始环境和权限尽量早出现。
3. **变化追加，不改旧项**：权限/cwd/model/环境变化用新的 developer/user context item 表达，让旧 prefix 仍保持 byte-level/structure-level 稳定。
4. **区分 cache 和 stateful delta**：`prompt_cache_key` 是服务器缓存域；`previous_response_id` 是 WebSocket 上的上一响应状态引用。两者都用，但失败时各自独立回退。
5. **字段稳定性要纳入测试**：像 Codex 的 `prompt_caching.rs` 一样，用请求快照验证 prefix、key、instructions、tools、override 后追加位置。
6. **避免 volatile id 污染 prompt**：除非 provider 需要 store/item ids，否则清掉 item id，减少不稳定字段。
7. **用服务端 usage 做账**：不要假设命中；从 `cached_tokens` 进入 token 账本、预算、UI 和 telemetry。
8. **长线程要有 compaction**：prompt cache 不能替代 context management。过长 history 仍要 compact，否则窗口、传输和模型注意力都会退化。

## 限制与风险

- `prompt_cache_key` 只是 cache scope，不保证命中。模型变化、工具列表变化、reasoning/text controls 变化、Responses Lite/普通模式切换都可能降低复用。
- `client_metadata` 被 WebSocket incremental 比较忽略，但它仍会进入请求控制面。Codex 把它设计为非 prompt 语义字段，不能依赖它影响模型行为。
- Responses Lite 改变了稳定前缀的位置；同一 thread 在普通模式和 Lite 模式之间切换时，WebSocket incremental 会失效，prompt cache 的可复用前缀也会显著变化。
- compaction 会重写 history，通常意味着旧长 prefix 不再直接复用；它换来的是新窗口的可控长度。
- tool specs 如果随 MCP/plugin 动态变化，会改变顶层 `tools` 或 Lite 的 `AdditionalTools` 前缀，直接影响缓存。

## 总结

Codex 对 Responses API 的使用可以看成三层：

1. **协议层**：把 agent history 和工具世界映射成 Responses API 的结构化请求和流式事件。
2. **运行时层**：用 WebSocket、SSE、retry/fallback、request compression、metadata、turn-state 支撑长 turn 的可靠执行。
3. **缓存层**：用稳定 `prompt_cache_key`、稳定前缀、追加式 context update、Responses Lite 前缀布局和 `cached_input_tokens` 反馈，尽量让复杂 agent thread 的重复上下文被复用。

因此，Codex 的 prompt cache 不是一个孤立开关，而是贯穿 history 组织、request 构造、transport 增量、compaction 和 token accounting 的整体设计。
