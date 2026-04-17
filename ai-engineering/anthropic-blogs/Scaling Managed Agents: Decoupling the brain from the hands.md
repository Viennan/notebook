# Scaling Managed Agents: Decoupling the brain from the hands

- 原文标题：Scaling Managed Agents: Decoupling the brain from the hands
- 原文链接：https://www.anthropic.com/engineering/managed-agents
- 作者：Lance Martin、Gabe Cemaj、Michael Cohen
- 主题：Managed Agents、agent architecture、harness design、session durability、sandbox isolation、security boundary、long-running agents
- 关联产品：Claude Managed Agents
- 备注：Anthropic 原文页未在当前抓取结果中直接显示可见发布日期；本文内容依据原文正文、页面结构化数据与官方文档整理。

## 一句话总结

这篇文章的核心结论是：如果我们把 agent 的能力持续提升视为常态，那么真正应该稳定下来的，不是某一代具体的 harness，而是 agent 周围那组足够抽象、能跨代延续的接口。Anthropic 用 `session / harness / sandbox` 三层解耦，把 “brain（Claude 与其调度逻辑）” 从 “hands（执行环境与外部工具）” 和 “session（可恢复的事件日志）” 中拆开，构建出一套可以随着模型能力演进而持续替换内部实现的 `meta-harness`。

## 这篇文章在回答什么问题

Anthropic 近几篇工程博客一直围绕一个问题展开：怎样让 AI agent 在真实世界里长时间、稳定、可恢复地工作。此前他们已经讨论过：

- 如何构建有效的 agent
- 如何为 long-running work 设计 harness
- 如何做 context engineering

这篇文章进一步推进到平台层设计：如果 harness 本身也会随着模型能力提升而过时，那么平台不应把“今天最有效的 harness 策略”写死为系统边界，而应抽象出一组更稳定的运行接口，让未来的 harness 可以自由替换。

文章给出的答案是：不要把 agent 看成一个大容器里的单体程序，而要像操作系统设计一样，把它视作若干可虚拟化（virtualize）的部件。

## 核心观点

### 1. Harness 会过时，接口比策略更值得固化

文章开篇延续 Anthropic 的一个持续观点：harness 本质上编码了很多“模型现在还做不到什么”的假设。这些假设一旦被模型能力进步击穿，就会从补丁变成负担。

原文用一个很典型的例子说明这一点：他们曾发现 Claude Sonnet 4.5 在接近 context window 上限时会出现 “context anxiety”，也就是倾向于过早收尾。于是他们给 harness 加了 context reset 机制；但把同样机制迁到 Claude Opus 4.5 后，这一行为已经消失，原先的补丁反而成了 dead weight。

这背后的方法论非常重要：

- 短期内，harness 是提升 agent 表现的关键抓手。
- 长期看，任何写死在 harness 里的假设都可能过期。
- 因此平台应该对“接口”强约束，对“实现”弱约束。

### 2. 目标不是做一个 harness，而是做一个 meta-harness

Anthropic 用了一个非常经典的计算机系统视角来解释它们的设计目标：为 “programs as yet unthought of” 设计系统。

这句话来自《The Art of Unix Programming》，意思并不是预知未来所有程序，而是设计出一组足够通用的抽象，让未来尚未出现的程序也能在其上运行。操作系统之所以能跨几十年演进，正是因为 `process`、`file`、`read()` 这些抽象足够稳定，而底层硬件却可以不断变化。

Anthropic 想做的并不是“今天最强的 agent 容器”，而是 agent 时代的这类稳定抽象层。于是他们把 agent 拆成三部分：

- `session`：append-only log（追加式事件日志），记录发生过的一切
- `harness`：调用 Claude、路由工具调用、进行上下文管理的循环
- `sandbox`：Claude 可执行代码、编辑文件、访问执行环境的地方

关键不在于三者的存在，而在于三者之间通过接口连接，可以被独立替换、独立失败、独立恢复。

## 从单容器到解耦架构

### 1. 为什么单容器方案会失效

Anthropic 最初把 session、harness、sandbox 都放在同一个 container 中。这种方式一开始有明显优点：

- 文件编辑就是本地 syscall，路径简单
- 没有服务边界，系统实现更直接
- 调试心智模型也更线性

但随着系统进入真实生产环境，这个设计暴露出几个结构性问题。

第一，它把整套系统变成了一个 `pet`，而不是 `cattle`。

这里的 `pets vs cattle` 是基础设施领域的经典比喻：

- `pet`：需要人工照料、不能轻易丢失的单个实例
- `cattle`：可随时替换、故障后直接重建的同质实例

当 session、harness、sandbox 全部耦合在同一容器里时，容器一旦失效，session 状态也一起丢失；容器一旦卡死，工程师就不得不手工介入“救火”。

第二，问题定位极其困难。

外部只能看到 WebSocket event stream，但这个视角无法区分故障到底来自：

- harness 本身的 bug
- event stream 的网络问题
- container 已离线

最终调试往往需要直接进入容器 shell，但容器里又常常混有用户数据，这让可调试性和安全性都变得尴尬。

第三，harness 被默认假设为“和工作资源共处一地”。

这在早期看似自然，但当企业用户希望 Claude 访问他们自己 VPC（Virtual Private Cloud，虚拟私有云）中的资源时，这个假设反而成了障碍。客户要么与 Anthropic 网络打通，要么自己托管 Anthropic 的 harness。也就是说，原本只是实现细节的部署假设，最终演化成了产品边界。

### 2. 解法：把 brain、hands、session 分开

Anthropic 的方案是把 agent 的三个核心部分拆开：

- `brain`：Claude 与其 harness
- `hands`：执行动作的 sandboxes 与各类 tools
- `session`：对所有 session events 的可恢复日志

这带来三个直接结果：

- 每一层都可以单独失败，不会把整套系统一起拖垮
- 每一层都可以独立替换，不要求同步升级
- 系统边界从“容器进程”上升为“稳定接口”

## 关键接口设计

文章里最值得工程师反复琢磨的，不是概念名词，而是这些接口长什么样。

### 1. Hands 统一成 tool interface

当 harness 不再住在 container 里后，它与容器的交互就退化为普通工具调用：

```text
execute(name, input) -> string
```

这意味着对 harness 而言，container 不再是“我所在的世界”，而只是“我能调用的一种手”。只要遵守这个接口，手可以是：

- Linux container
- MCP server
- 自定义外部工具
- 手机、浏览器、甚至文章中举例的 Pokemon emulator

这一步的价值在于，把“环境类型”从架构假设降级为运行时选择。

### 2. Sandboxes 可以像 cattle 一样重建

既然 hands 只是工具，那么 sandbox 宕机时就不需要人工抢救，而是通过标准配方重新创建：

```text
provision({resources})
```

如果 Claude 认为值得重试，系统就初始化一个新的 sandbox，把失败视为普通 tool-call error 继续处理。这是典型的云原生思路：不要维护故障实例，而要快速重建可替代实例。

### 3. Harness 也变成可丢弃的无状态组件

更激进的一步在于：Anthropic 不只把 sandbox 变成 cattle，也把 harness 本身变成 cattle。

由于 session log 独立存在，harness 无需自带持久状态。它崩溃后，新的 harness 可以通过：

```text
wake(sessionId)
getSession(id)
emitEvent(id, event)
```

恢复运行：

- `wake(sessionId)`：唤醒或接管某个会话
- `getSession(id)`：拿回完整事件日志
- `emitEvent(id, event)`：在运行过程中持续写入事件

这等于把“agent 是否还活着”与“某个具体 harness 进程是否还活着”彻底分离。

### 4. Session 不是 Claude 的 context window

这是全文另一个非常关键的思想。

很多人会把“会话历史”直接等同于模型当前看到的上下文，但 Anthropic 明确把两者拆开：

- `session` 是可恢复、完整、耐久的事件存储
- `context window` 是当前一次推理真正送进模型的上下文切片

两者不应该是一回事。

原因在于 long-running agent 常常要跨越 context window 限制。无论你用 compaction、memory tool、trimming 还是 summary，本质上都在做不可逆的信息保留决策，而这些决策很难保证未来推理一定够用。

Anthropic 的做法是：把完整上下文保存在 session log 中，再通过：

```text
getEvents()
```

按位置切片、回看、重读、选择性取回事件；至于这些事件如何被转换、压缩、组织进 Claude 的上下文窗口，则交给 harness 自由发挥。

这实际上把两类职责分开了：

- session 负责“可恢复存储”
- harness 负责“上下文工程（context engineering）”

这也是文章里最具长期价值的设计点之一。

## 安全边界的重构

这篇文章并不只是在谈可扩展性，也在谈更本质的安全边界问题。

在早期耦合设计里，Claude 生成的不可信代码（untrusted code）与凭证（credentials）在同一个 container 内运行。这样一来，prompt injection 一旦成功诱导 Claude 读取环境变量，攻击者就可能拿到 token，再开出新的受控会话，风险会沿着 agent 链路放大。

文章强调，单纯做 narrow scoping 当然有帮助，但这仍然建立在“模型暂时做不到某些越界行为”的假设上。随着模型持续变强，这类假设会越来越不稳。

Anthropic 采用的是结构性修复（structural fix）：

- 让 sandbox 根本接触不到凭证
- 认证信息要么跟资源绑定，要么保存在 sandbox 之外的 vault 中

两个具体例子很值得注意：

### 1. Git 凭证不交给 agent 本身

仓库访问 token 用于 sandbox 初始化时的 clone 与 remote wiring，因此 Claude 可以在 sandbox 内执行 `git push` / `git pull`，但并不直接持有 token。

### 2. MCP 工具通过代理访问

对于 custom tools，Anthropic 支持 MCP（Model Context Protocol，模型上下文协议），并把 OAuth 凭证存进 secure vault。Claude 调用 MCP 工具时，走的是带 session token 的专用代理；代理再去 vault 取真实凭证并访问外部服务。harness 本身也不感知真实 credentials。

这背后的原则是：

- 不要只做权限收窄
- 更要做能力隔离

也就是把“模型生成的代码能否接触凭证”从策略问题变成架构事实。

## 性能收益：Many brains, many hands

### 1. Many brains

解耦之后，一个直接收益是可以更自然地扩展多个 `brains`。

在旧设计里，如果每个 brain 都绑定一个 container，那么每次 session 启动都要先付出完整容器初始化成本，包括：

- provision container
- clone repo
- boot process
- 拉取 pending events

这会显著拉高 `TTFT`（Time-To-First-Token，首个 token 生成延迟）。

而在新设计里，inference 可以先开始；只有在确实需要 hands 时，brain 才通过工具调用去 provision 对应执行环境。Anthropic 在文中给出两项非常醒目的结果：

- `p50 TTFT` 下降约 60%
- `p95 TTFT` 下降超过 90%

这说明脑和手的拆分不仅是架构洁癖，而是用户可直接感知的延迟优化。

### 2. Many hands

另一侧收益是支持一个 brain 面向多个 hands 工作。

这会提升 Claude 在多执行环境之间规划和调度的负担，但也打开了更强的系统表达力：每只 hand 都只是一个工具端点，brain 可以根据任务性质决定把工作发往哪个环境。

更重要的是，某一只 hand 失败，不会连带脑和其它 hands 的状态一起崩塌。因为 hand 不再与某个 brain 独占绑定，所以理论上 brains 之间还可以传递 hands。

这为未来的 multi-agent / multi-environment orchestration 留出了非常大的空间。

## 系统架构视角：可编程工作流与 Event Sourcing

从更经典的分布式系统架构看，Managed Agents 很像一种 `LLM-native programmable workflow runtime`（面向 LLM 的可编程工作流运行时）。它与 Azure Durable Functions、Temporal/Cadence 这类 durable workflow engine，以及 `Event Sourcing`（事件溯源）模式有很强的同构关系。

可以做一个近似映射：

| Managed Agents | Durable Workflow / Azure Durable Functions | Event Sourcing 视角 |
| --- | --- | --- |
| `session` | workflow history / orchestration history | append-only event log，事实来源 |
| `harness` | orchestrator / workflow runtime | 读取事件流并构造当前执行状态 |
| `sandbox / tools / MCP` | activity functions | 产生外部 side effects 的命令端 |
| `wake(sessionId)` | resume / replay orchestration | 从事件日志恢复执行 |
| `emitEvent(id, event)` | append workflow event | 追加不可变事实 |
| `context engineering` | replay 后重建局部状态 | 从事件流投影出模型当前可见上下文 |

这个类比能解释为什么 `session != context window` 是全文最关键的设计之一。在 Event Sourcing 里，event log 是完整事实来源，而业务状态通常是从事件流投影（projection）出来的 materialized view。Managed Agents 里也是类似关系：

- `session` 是完整、耐久、可回放的事件日志
- `context window` 是 harness 从 session 中选择、压缩、排序、摘要后生成的临时 projection
- `tool result`、模型输出、错误、人工输入、环境变更都应该被记录为事件，而不是只存在于进程内存中

但它和传统 durable workflow 也有一个根本差异：传统 workflow 通常要求 orchestrator replay 是确定性的。以 Azure Durable Functions 或 Temporal 为例，orchestrator code 在 replay 时不能随意调用非确定性 API，因为运行时需要依靠历史事件稳定重建执行状态。

Managed Agents 面对的核心执行者却是 LLM。LLM 推理天然是概率性的、上下文敏感的，不能假设“重新调用一次 Claude 会得到同样结果”。因此更合理的做法不是重跑过去的推理，而是把已经发生过的模型输出、tool call、tool result、error、human feedback 都固化进 session log；恢复时重放这些事实，只在新的决策边界继续调用模型。

所以 Managed Agents 可以被理解为：

```text
event-sourced agent workflow runtime
```

它与传统 workflow engine 的最大区别在于：workflow graph 不是开发者预先写死的 DAG 或 orchestrator code，而是 Claude 在 harness 约束下，根据 session history、tool schema、policy 与当前目标动态生成的执行轨迹。

这个视角也能帮助我们重新理解文中的几个设计选择：

- `hands` 必须隔离成 tools / activity，因为所有外部 side effects 都应该有明确边界。
- `harness` 要尽量 stateless，因为 durable state 应该来自 session log，而不是某个进程的内存。
- `sandbox` 可以 cattle 化，因为核心状态不应该绑定在某个执行容器上。
- `credentials` 不能放进 sandbox，因为 activity 执行环境不应该天然拥有 orchestration 权限。
- `context compaction` 只是 projection 策略，不能替代完整 event history。

如果说 Durable Functions / Temporal 是“用代码编写可恢复工作流”，那么 Managed Agents 更像是“让 LLM 在受控接口上即时编写、执行、恢复工作流”。这也是它区别于普通 agent demo 的地方：它不是只让模型调用工具，而是在构建一个能承载模型动态工作流的持久化运行时。

## 从同步工具调用到原生异步分布式 Agent Runtime

顺着上面的视角再往前走一步，Managed Agents 其实还隐含着一个非常重要的方向：一旦 agent runtime 建立在 event log、stateless harness 和可替换 hands 之上，它就天然具备演进为“原生异步系统”的基础。

这意味着 agent 平台不必把执行流程理解成：

```text
用户请求 -> LLM 推理 -> 同步调用工具 -> 等待返回 -> 再次推理 -> 返回结果
```

它更像是：

```text
事件到达 -> harness 消费 session -> 生成下一步命令 -> 异步派发给 tool / sandbox / LLM activity
-> 外部结果回流为新事件 -> 任意 worker 继续推进 session
```

在这种设计下，`session` 是 durable state，`harness` 是无状态调度者，而 tool 调用乃至 `LLM inference` 本身，都可以被视作异步 activity，而不是必须阻塞当前线程的同步步骤。

这会带来几个非常本质的变化：

### 1. Agent 生命周期不再绑定某个同步请求

传统聊天式 agent 往往隐含一个假设：用户发起一次请求，系统在一个连续阻塞的调用链里完成推理和工具调用，然后返回结果。但对 long-running agent 来说，真正合理的模型是：

- session 持续存在
- worker 可以随时被唤醒处理新事件
- 某一步耗时很长时，整个 agent 不需要“卡住”
- 当前 worker 消失后，另一个 worker 仍可继续推进同一个 session

这使 agent 从“同步请求处理器”变成“可持续推进的异步执行体”。

### 2. Tool call 不必是同步函数调用

一旦 tools 被抽象成带边界的 activity，它们就不必表现为 blocking RPC。更自然的方式是：

- harness 发出 command
- 外部系统异步执行
- 执行完成后把结果作为 event 写回 session
- 任意 harness worker 看到新事件后继续决策

这样无论工具是：

- 远程浏览器自动化
- 代码执行容器
- 企业内网审批流
- 人工 review
- 第三方 webhook 回调

都可以统一成异步事件驱动模式，而不是强行包成同步接口。

### 3. 连 LLM inference 都可以被 activity 化

很多人默认“工具可以异步，但模型推理一定是当前线程里阻塞完成的”。但从 runtime 设计看，推理同样可以被建模为一种 activity：

- harness 决定需要一次新的 model call
- 生成 inference request event
- 独立推理服务异步处理
- 模型输出作为 event 回写 session
- 下游 worker 再继续推进工作流

这让系统在高并发下更容易做：

- 推理队列化
- 优先级调度
- 超时与取消
- 多模型路由
- 资源隔离与弹性扩缩容

换句话说，agent 不再是“包着工具调用的 LLM request”，而是“把 LLM 本身也纳入可调度工作流的异步执行系统”。

### 4. Interruptibility 会成为一等能力

在 event-sourced runtime 中，`pause / resume / cancel / retry / handoff` 不是额外贴上的功能，而是架构自然产物：

- `pause`：停止消费新事件
- `resume`：重新唤醒 worker 继续推进
- `cancel`：向 session 写入取消事件，并由后续组件协同收敛
- `retry`：基于失败事件触发重试策略
- `handoff`：更换 harness、worker、模型或工具环境继续执行

同理，human-in-the-loop 也不再是特判逻辑，而只是“向 session 注入新事件”的另一种来源。

### 5. 这为规模化 agentic service 提供了真正的分布式基础

如果目标是面向大量用户、长生命周期任务和复杂企业流程提供 agentic service，那么同步阻塞式架构很快会遇到瓶颈。相反，基于事件回放与 stateless harness 的设计天然支持：

- 横向扩展 worker
- 抢占与迁移执行权
- 分离 control plane 与 execution plane
- 用队列和事件总线吸收高峰流量
- 按工具类型或模型类型做资源池化
- 在多 region / 多 VPC / 多租户场景下分发 hands

因此，Managed Agents 的深层意义并不只是“更容易接工具”，而是它为规模化 agent 平台提供了一种异步、分布式、可中断的系统底座。

### 6. 但这也意味着要用分布式系统的标准来设计 agent

一旦 agent runtime 真正进入异步分布式形态，工程问题也会发生变化。团队需要认真处理：

- tool execution 的 `idempotency`
- event delivery 的 `at-least-once` 语义
- `dedupe`、`correlation id`、`lease`、`ownership`
- 并发 worker 是否允许同时推进同一 session
- cancellation 是软中断还是强终止
- 哪些状态必须 durable，哪些可以只放在临时缓存

这说明 agent platform 最后会越来越像 workflow engine、actor system 和 event-driven orchestration 的结合体，而不再只是“会调工具的大模型应用”。

## 我对这篇文章的理解与评价

如果只看表面，这篇文章像是在介绍 Anthropic 新产品 Managed Agents 的底层实现；但从研究和工程方法论角度看，它真正重要的地方在于给出了一个很清晰的 agent platform 设计原则：

### 1. 把“模型能力会持续变化”视为一等约束

很多 agent 系统把今天有效的 prompt、memory 机制、error recovery 机制直接沉淀成硬编码系统逻辑。Anthropic 的做法更克制：他们承认这些机制会随着模型进步而失效，因此尽量只把稳定接口产品化。

### 2. 把上下文管理从存储层剥离

很多团队会混淆“历史如何存”和“当前喂模型什么”。这篇文章给出的 session/context 分离，对做长任务 agent 的团队特别重要。它本质上把 agent memory 从 prompt 技巧升级成了系统设计问题。

### 3. 把安全从 policy 问题升级为 architecture 问题

对于 prompt injection、tool misuse、credential leakage 这类问题，这篇文章的态度非常明确：与其不断猜模型不能做什么，不如从架构上让高风险能力不可达。

### 4. 把 agent runtime 平台化，而不是把 agent demo 工程化

很多自建 agent 系统在 demo 阶段运行良好，但一旦接入真实企业网络、真实长时任务、真实权限体系，就会被调试性、恢复性、安全边界和多环境接入问题压垮。本文可以视为 Anthropic 对“从好用 demo 迈向可运营平台”这一步的系统回答。

## 对实践者的启发

如果你在做自己的 agent 平台，这篇文章最值得借鉴的不是 Anthropic 的具体 API，而是这几条设计原则：

1. 不要把 harness 和 sandbox 放进同一个不可分割的故障域。
2. 不要把完整 session history 与当前 context window 混为一谈。
3. 不要让执行环境直接接触长期凭证。
4. 不要把“今天的最佳上下文策略”硬编码成平台的永久边界。
5. 尽量让运行时组件可替换、可重放、可恢复、可水平扩展。
6. 可以把 agent session 设计成 event-sourced workflow，把 prompt/context 视为可替换 projection，而不是事实来源本身。

对于中小团队，这也意味着一个现实判断：

- 如果你的核心竞争力不在 agent runtime 基础设施，而在业务工作流、知识、工具生态或产品体验，那么直接采用托管式 agent 平台可能比自建更划算。
- 如果你必须自建，也应该优先把 session durability、tool isolation、environment abstraction 设计好，而不是先堆更多 prompt 技巧。

## FAQ

### 1. 什么是 `meta-harness`？

可以把它理解为“承载多种 harness 的平台层”。它不规定未来 Claude 必须如何规划、如何做 context compaction、如何切换 memory 策略；它只规定这些能力运行时依赖的稳定边界，比如 session、sandbox、tool interface 等。

### 2. 为什么说 `session` 不等于 `context window`？

因为 `session` 记录的是完整、可恢复的历史，而 `context window` 是当前推理时送进模型的那部分切片。前者强调 durability（耐久性）与 replayability（可回放性），后者强调 token budget 下的即时推理效率。二者职责完全不同。

### 3. 为什么 brain/hands 解耦后延迟会下降？

因为旧架构要求每个 session 一开始就把 container 启好，哪怕之后根本不需要使用 sandbox；而新架构允许先开始推理，只有真正需要 hands 时才去 provision 对应环境，从而显著减少首 token 等待时间。

### 4. 这和 MCP 有什么关系？

MCP 在这里扮演的是“标准化外部工具接入层”。当 hands 被统一为工具接口后，MCP server 就能自然成为一种 hand；同时，Anthropic 还把凭证管理放在 sandbox 外部，通过代理访问 MCP，进一步强化了安全边界。

### 5. 这和 Azure Durable Functions / Temporal 有什么关系？

它们都强调 durable execution：执行过程不能只依赖某个进程还活着，而要能通过持久历史恢复。区别在于，Durable Functions / Temporal 的 workflow 通常由开发者写成确定性的 orchestrator code；Managed Agents 的执行轨迹则由 Claude 在运行时动态生成，因此更依赖把模型输出、工具调用和工具结果都记录为 session events。

### 6. 这和 Event Sourcing 有什么关系？

关系非常直接：`session` 可以理解为 append-only event log，是系统恢复和审计的事实来源；`context window` 则是从事件流投影出来的临时视图。这个区分能避免把 prompt 摘要误当成完整记忆，也能让未来的 harness 使用不同 projection 策略重读同一条 session history。

### 7. 这篇文章最适合哪些读者？

最适合三类人：

- 正在自建 agent runtime / agent platform 的工程师
- 需要让 agent 接企业内网、私有工具或多执行环境的团队
- 已经感受到 prompt engineering 不足，开始进入 context engineering、tool orchestration 与 infra design 阶段的实践者

## 建议的延伸阅读

建议按下面顺序阅读，能更容易看出 Anthropic 这条思想脉络：

1. `Building effective agents`
2. `Effective harnesses for long-running agents`
3. `Effective context engineering for AI agents`
4. `Harness design for long-running application development`
5. `Claude Managed Agents overview`

如果想把这篇文章放进更广的系统设计语境里，还建议补两篇经典材料：

1. `The Bitter Lesson`
2. `Programs as Yet Unthought Of`（《The Art of Unix Programming》相关章节）

## 参考资料

### Anthropic 原文与官方文档

1. Scaling Managed Agents: Decoupling the brain from the hands
   https://www.anthropic.com/engineering/managed-agents
2. Claude Managed Agents overview
   https://platform.claude.com/docs/en/managed-agents/overview

### Anthropic 相关文章

1. Building effective agents
   https://www.anthropic.com/engineering/building-effective-agents
2. Effective harnesses for long-running agents
   https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
3. Effective context engineering for AI agents
   https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
4. Harness design for long-running application development
   https://www.anthropic.com/engineering/harness-design-long-running-apps

### 文中引用的外部材料

1. The Bitter Lesson
   http://www.incompleteideas.net/IncIdeas/BitterLesson.html
2. Programs as Yet Unthought Of
   http://www.catb.org/esr/writings/taoup/html/ch03s01.html
3. The History of Pets vs Cattle and How to Use the Analogy Properly
   https://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/
4. arXiv: 2512.24601
   https://arxiv.org/pdf/2512.24601

## 可关联到本知识库的关键词

- Managed Agents
- meta-harness
- brain and hands decoupling
- session durability
- context engineering
- sandbox isolation
- MCP
- long-running agents
- TTFT
