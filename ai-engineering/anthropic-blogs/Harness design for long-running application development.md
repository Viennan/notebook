# Harness design for long-running application development

- 原文标题：Harness design for long-running application development
- 原文链接：https://www.anthropic.com/engineering/harness-design-long-running-apps
- 发布日期：2026-03-24
- 作者：Prithvi Rajasekaran
- 主题：harness design、long-running agents、agentic coding、frontend design、planner-generator-evaluator、context reset、QA harness
- 关联文章：
  - [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
  - [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
  - [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
  - [Scaling Managed Agents: Decoupling the brain from the hands](./Scaling%20Managed%20Agents:%20Decoupling%20the%20brain%20from%20the%20hands.md)
- 备注：本文是 Anthropic 工程博客。导读主要依据原文正文，以及文中直接引用的官方资料整理。

## 一句话总结

这篇文章的核心结论是：`harness` 不是一次性设计好的固定脚手架，而是围绕“当前模型还不擅长什么”不断调整的外部结构。Anthropic 通过 `generator / evaluator` 与 `planner / generator / evaluator` 两套多 agent 结构，把主观审美评审、长任务拆解、QA 验证和上下文治理组合起来；而当模型从 Sonnet 4.5、Opus 4.5 进化到 Opus 4.6 后，他们又主动删掉一部分已经不再关键的 scaffolding（脚手架）。但这并不意味着模型越强，harness 就会线性贬值；更强的模型也会把团队带到更复杂的任务区间，而这些新任务又可能重新超出模型自身的稳定完成边界，因此优秀的 harness 设计本质上是在持续做“加法 + 减法”。

## 这篇文章在回答什么问题

文章实际在回答四个很重要的问题：

1. 为什么单个 agent 在长时间应用开发任务中，常常一开始看起来很强，跑久了却逐渐失控？
2. 对于“设计好不好看”这种主观任务，怎么把模糊审美变成模型可执行、可迭代优化的评估系统？
3. 对于 full-stack application development（全栈应用开发）这种既要求产品规划、又要求代码正确、还要求真实可用的任务，harness 应该怎么分工？
4. 当模型能力提升后，哪些 harness 结构应该保留，哪些应该删掉？

如果把 Anthropic 的相关文章放在一起看，这篇文章更像是“任务级 harness 设计”的实战补充，而不是平台级 runtime 架构文章。与 [Scaling Managed Agents: Decoupling the brain from the hands](./Scaling%20Managed%20Agents:%20Decoupling%20the%20brain%20from%20the%20hands.md) 相比，后者关注的是 `session / harness / sandbox` 的运行时解耦；本文关注的则是：在一个具体、长时、昂贵、容易失控的 coding task 中，怎样组织 agent 角色、上下文与评审循环，才能把结果拉升到更高质量。

## 核心观点

### 1. Naive harness 的两个核心问题：长任务失焦与自我评价失真

Anthropic 指出，长时间 `agentic coding`（具备自主规划与工具调用能力的编码流程）里，naive implementation（朴素实现）最常见的两个失败模式是：

- 随着 context window（上下文窗口）逐渐填满，模型会在长任务中丢失连贯性。
- 模型在评估自己产出的结果时，天然偏向宽松，容易“自己夸自己”，即使结果在人工看来只是一般甚至明显有问题。

其中第一个问题在他们之前的文章里已经出现过。某些模型会表现出 `context anxiety`，也就是越接近自己感知中的上下文上限，越容易过早收尾、草率结束任务。为了解决这个问题，Anthropic 早期采用了 `context reset`：清空上下文，启动一个新的 agent，并通过结构化 handoff artifact（交接产物）把前一阶段状态传给下一阶段。

本文进一步强调，`compaction` 和 `context reset` 不是一回事：

- `compaction`：对旧上下文做摘要压缩，让同一个 agent 继续工作。
- `context reset`：完全开启新上下文，用结构化交接资料恢复任务。

前者保留连续性，后者提供“干净重启”。对 Sonnet 4.5 而言，光靠 compaction 不够；对后续更强模型，这种需求则会变化。

### 2. 主观质量也能被“工程化”为可评分标准

本文最有启发性的地方之一，是它展示了如何把主观设计审美转成可操作的评价系统。

Anthropic 在 frontend design（前端设计）实验里，没有直接问模型“这个设计好不好看”，而是把评审拆成四个维度：

- `Design quality`：整体是否有统一气质与明确审美方向。
- `Originality`：是否有明确的原创决策，而不是模板化、组件库默认态、或典型 “AI slop”。
- `Craft`：技术执行是否扎实，例如排版层级、留白、配色、对比度。
- `Functionality`：用户是否能理解界面并顺利完成操作。

这里最重要的方法论，不是这四个具体维度，而是这条路线本身：

- 先把模糊判断拆成可陈述、可解释、可评分的标准。
- 再把这些标准同时交给 `generator` 和 `evaluator`。
- 让 `evaluator` 通过真实工具观察结果，再把具体批评喂回给 `generator`。

这种做法本质上是在把“品味”部分结构化，而不是假装主观问题可以被完全客观化。

### 3. 分离生成者与评审者，是很强的性能杠杆

作者从 `Generative Adversarial Networks`（GANs，生成对抗网络）获得灵感，采用了 `generator / evaluator` 结构：

- `generator` 负责生成前端。
- `evaluator` 负责与真实页面交互、打分、输出批评。

关键不只是“多一个 agent”，而是“角色分离”。

文章明确指出，让同一个 agent 既生成又评审，往往会高估自己；而把评审独立出来后，虽然 `evaluator` 本身还是 LLM，但我们可以更容易把它调成“持怀疑态度的审查者”。换句话说：

- 让生成者变得自我苛刻，很难。
- 让独立评审者变得更苛刻，更可控。

这也是后面 full-stack coding harness 能成立的前提。

### 4. Full-stack coding 需要的不只是 coding agent，而是 planner、builder、QA 的组合

把这套思路迁移到软件开发后，Anthropic 进一步扩展成 `planner / generator / evaluator` 三段式结构：

- `planner`：把用户 1 到 4 句的简短需求，扩展成完整产品规格。
- `generator`：按规划去实现应用。
- `evaluator`：通过 Playwright MCP 真实操作应用，做 UI、API、数据库层面的 QA。

其中 `planner` 的设计很关键。它不是详细规定所有低层技术细节，而是集中在：

- 产品范围定义
- 高层技术方向
- 交付物和用户体验目标
- 适合加入的 AI feature（AI 功能）

原因是如果 planner 在一开始就把技术细节写得过死，而且写错了，这些错误会沿着后续实现层层放大。作者因此刻意让 planner 偏向 high-level spec（高层规格），而不是 low-level implementation（低层实现细则）。

### 5. 真正好的 harness 不是越复杂越好，而是持续验证“哪些部件还在承担价值”

本文非常有价值的一点，是作者没有把多 agent harness 写成“固定最佳实践”，而是明确强调：harness 中每一个组件，本质上都编码了“当前模型自己还做不到什么”的假设。

这意味着：

- 如果模型能力提升，这个组件可能会从关键结构变成额外负担。
- 如果任务边界变化，某个组件也可能只在特定难度区间里有价值。
- 如果模型能力提升后，团队开始挑战更复杂、更长链路、更多约束的任务，新任务又可能把模型重新推回能力边缘，此时就需要重新向 harness 补充新的辅助结构。

作者在 Opus 4.6 上做的事情很典型：

- 删除了 sprint construct（按 sprint 拆任务的结构）。
- 保留了 `planner`，因为没有 planner 时模型会明显 under-scope（范围缩小、做得太保守）。
- 保留了 `evaluator`，但把它从“每个 sprint 一轮 QA”调整成“整次构建后再做一轮 QA”。

这说明 `evaluator` 不是“永远都需要”或“永远不需要”的固定答案，而是取决于任务是否处在当前模型 solo reliability boundary（单独稳定完成边界）之外。更准确地说，模型能力提升会让一部分旧 harness 失去必要性，但同时也会鼓励团队去探索更高难度的任务空间；一旦任务复杂度再次跑到模型稳定能力之外，新的 harness 能力又会重新变得必要。

## 文章内容梳理

### 1. 为什么朴素实现会失效

作者先回顾了此前的 long-running harness 工作。之前 Anthropic 已经用过：

- 一个 initializer agent 负责初始化环境与任务结构
- 一个 coding agent 负责逐步实现功能
- 跨 session 的 handoff artifact 负责交接上下文

这套方案已经明显优于 baseline，但到了更复杂的任务，问题依旧存在。

第一个问题是长任务中的上下文退化。模型会：

- 越跑越失焦
- 在上下文接近上限时过早收尾
- 交接给下一轮时留下半成品和不清晰状态

第二个问题是 self-evaluation（自评）不可靠。即使在 coding task 中，模型也可能：

- 发现了真实问题
- 却又说服自己“问题不大”
- 最后给出过于乐观的通过判断

对设计任务，这个问题更严重，因为“好不好看”本来就没有单一二元真值。

因此，文章把 harness 的两个核心增量说得很清楚：

1. 要想办法让长任务在上下文治理上更稳。
2. 要把“做事的人”和“检查的人”拆开。

### 2. Frontend design：把主观审美转成评审回路

作者先在 frontend design 上做实验，因为这里最容易看见 self-evaluation 的问题。默认情况下，Claude 往往会生成：

- 技术上能运行
- 布局上比较安全
- 视觉上相对平庸

于是作者构建了一个 generator-evaluator loop（生成者-评审者循环）：

- `generator` 先生成 HTML/CSS/JS 前端。
- `evaluator` 使用 Playwright MCP，直接与 live page（真实运行页面）交互。
- `evaluator` 给出每个维度的分数和批评意见。
- 批评再作为下一轮 generator 的输入。

文章给出的几个经验非常值得记下来：

1. 评价标准本身就会塑造生成风格。  
   比如当评分标准里写入类似“museum quality”这样的措辞后，生成结果会朝某种审美方向收敛。

2. 提升并不总是线性的。  
   后期结果通常整体更强，但有时中间某一轮会比最终轮更讨喜。

3. 即便还没开始 evaluator 反馈，只要把评价语言写进 prompt，首轮输出也会明显优于无约束 baseline。  
   这说明标准不仅是评分器，也是在前置约束 generator 的搜索空间。

4. evaluator 必须经过校准。  
   Anthropic 用了 few-shot examples（少样本示例）和详细分数拆解，来减少 evaluator 判断漂移。

文章中最醒目的例子，是让模型做一个 Dutch art museum（荷兰艺术博物馆）网站。到第 9 轮时，结果还是一个优雅但相对可预期的深色 landing page；第 10 轮时，模型直接改成了具有空间感的 3D 房间式网页，用 CSS 透视制造房间效果，用门洞做展厅跳转。这说明经过多轮批评后，模型不只是“修修补补”，而是可能做出风格级跳跃。

### 3. 扩展到 full-stack coding：从 generator-evaluator 升级到三 agent 结构

把前端实验的经验迁移到 long-running autonomous coding（长时间自治编码）后，文章形成了一套更完整的结构。

### 3.1 Planner：把短 prompt 扩成可执行产品规格

用户只给一段很短的需求描述，例如一句话。`planner` 负责把它扩展成完整产品规格，包括：

- 产品目标
- 功能列表
- 视觉语言
- AI 功能融入方式
- 高层技术方向

这里作者特别强调：planner 不应该过度规定底层实现细节，否则一旦前面写错，后面全链路都会被带偏。

### 3.2 Generator：负责实现，但仍然要有节奏和边界

在 Opus 4.5 版本的 harness 中，`generator` 采用按 sprint 推进的方式：

- 一次只实现一个 sprint 的范围
- 使用 React、Vite、FastAPI、SQLite，后续也扩展到 PostgreSQL
- 每个 sprint 结束后先自评，再交给 evaluator
- 用 git 管理代码变更

这个设计延续了 Anthropic 之前“feature-by-feature” 的思想，本质上是防止模型一次性把任务吃太大，最后失控。

### 3.3 Evaluator：通过真实交互做 QA，而不是只看代码

`evaluator` 使用 Playwright MCP 来测试：

- UI 交互
- API 端点
- 数据库状态

它的评分标准从前端实验借鉴而来，但改造成更适合应用开发的四类目标：

- product depth（产品深度）
- functionality（功能性）
- visual design（视觉设计）
- code quality（代码质量）

每一项都有 hard threshold（硬阈值）。任何一项没过，这个 sprint 就算失败，generator 会收到详细问题反馈。

### 3.4 Sprint contract：在写代码前先对“什么叫完成”达成一致

这是文章里非常值得借鉴的一个细节。

在每个 sprint 开始前，generator 和 evaluator 会先协商一份 sprint contract（迭代合同），明确：

- 这个 sprint 具体要实现什么
- 什么行为会被当作完成标准
- evaluator 后续如何验证

这样做的意义在于，原始 product spec 本来就是高层描述，如果不补这一层“完成定义”，generator 很容易做偏，或者 evaluator 和 generator 对“完成”理解不一致。

Anthropic 让 agent 之间通过文件交换这些内容，而不是依赖含糊的会话记忆。这个思路和他们之前 long-running harness 中依赖 artifacts 交接上下文，是一条一致的方法论。

### 4. 具体实验结果：为什么这套 harness 虽贵，但确实抬升了结果上限

文章给了两个很有代表性的案例。

### 4.1 Retro game maker：完整 harness 明显强于 solo run

测试 prompt 是：

> Create a 2D retro game maker with features including a level editor, sprite editor, entity behaviors, and a playable test mode.

官方给出的开销对比：

| Harness | Duration | Cost |
| --- | --- | --- |
| Solo | 20 min | $9 |
| Full harness | 6 hr | $200 |

看起来 full harness 贵了 20 多倍，但结果差距非常明显。

solo run 的问题包括：

- 面板布局浪费大量空间
- 用户流程不清晰
- 需要先创建 sprite 和 entity，但 UI 没有引导
- 最关键的是，实际游戏模式坏掉了，角色无法正常交互

full harness 的结果则明显更强：

- 规划出的功能更多，规格被扩展到 16 个 feature、10 个 sprint
- 视觉设计更完整、更统一
- sprite editor 更成熟
- 内置了 AI 能力来辅助生成关卡等内容
- 最关键的是，play mode 至少真正能玩了

文章还给出 evaluator 抓到的一些具体缺陷例子，说明 QA 并不是“泛泛而谈”，而是真能指出可修复问题：

- 填充矩形工具没有真正填满区域，只在拖拽起止点落点。
- 选择并删除 entity spawn point 的逻辑条件写错。
- FastAPI 路由定义顺序错误，导致 `reorder` 被误匹配成 `frame_id`。

这些例子非常重要，因为它们说明 evaluator 的价值不只是“最后打个分”，而是把可执行修复建议直接喂回 generator。

### 4.2 Digital Audio Workstation：更强模型带来新的简化空间

在更新后的 v2 harness 中，Anthropic 使用的测试 prompt 是：

> Build a fully featured DAW in the browser using the Web Audio API.

这次作者运行在 Opus 4.6 上，并删掉了 sprint construct。官方给出的总耗时和成本为：

| Agent / Phase | Duration | Cost |
| --- | --- | --- |
| Planner | 4.7 min | $0.46 |
| Build (Round 1) | 2 hr 7 min | $71.08 |
| QA (Round 1) | 8.8 min | $3.24 |
| Build (Round 2) | 1 hr 2 min | $36.89 |
| QA (Round 2) | 6.8 min | $3.09 |
| Build (Round 3) | 10.9 min | $5.88 |
| QA (Round 3) | 9.6 min | $4.06 |
| Total V2 Harness | 3 hr 50 min | $124.70 |

这里最关键的观察不是“它仍然很贵”，而是：

- builder 能连续稳定工作两个多小时
- 不再像 Opus 4.5 那样强依赖 sprint 级拆分
- 但 QA 仍然能找到 generator 漏掉的关键功能缺口

QA 抓到的问题包括：

- 时间线上的 clip 不能拖动
- 乐器面板缺少真实交互
- 音频录制仍然只是 stub
- clip resize、split 等核心交互没完成
- 效果器可视化仍然只有数字滑杆，没有图形编辑体验

这些不是边角料，而是 DAW 可用性的核心交互。也正因为如此，作者给出的判断非常有启发：

- 当任务已经落入模型的“可可靠单独完成区间”内时，evaluator 会变成额外开销。
- 当任务还在能力边缘时，evaluator 依然能提供很高的 lift（质量增益）。

### 5. 这篇文章真正重要的地方：它在讨论“harness complexity 的迁移”

如果只把这篇文章理解成“多 agent 更强”，会错过它最值钱的部分。

它真正想表达的是：

- harness 的复杂度不是固定堆叠的。
- 随着模型能力变化，负责任的 AI engineer 需要重新做 ablation（消融）和 stress test（负载验证）。
- 复杂度应该围绕“当前模型能力边缘在哪里”持续迁移。
- 这种迁移既包含删掉已经失效的旧 scaffolding，也包含当任务上探后重新补入新的辅助结构。

文章给出的迁移路径非常典型：

1. 在 Sonnet 4.5 阶段，`context reset` 是关键结构，因为模型有明显的 context anxiety。
2. 到 Opus 4.5，context anxiety 已明显减弱，因此可以改成 continuous session + automatic compaction。
3. 到 Opus 4.6，长时稳定性更强，于是连 sprint decomposition 都可以尝试拿掉。

但与此同时，又不是所有结构都能删掉：

- `planner` 依旧有价值，因为模型会 under-scope。
- `evaluator` 依旧有价值，因为模型仍会漏掉最后一公里的交互深度和功能完整性。

所以本文最好的读法不是“记住一个最佳架构”，而是理解一种工作方式：

- 先观察模型在哪些地方不可靠
- 再为这些具体薄弱点设计 harness
- 新模型上线后，重新验证每个部件是否仍然 load-bearing（承担关键作用）
- 当模型变强、任务也随之升级时，再重新判断哪些新型辅助能力需要被加回 harness

## 我对这篇文章的理解与评价

### 1. 它把“prompt engineering”升级成了“task runtime engineering”

很多团队仍在把 agent 优化理解为 prompt 优化，但本文其实展示了更高一层的问题：当任务持续数小时、跨多个阶段、需要设计审美、功能正确性、真实 QA 与产品范围控制时，单次 prompt 已经不是主战场，真正的主战场是：

- 任务如何拆解
- 角色如何分离
- 上下文如何治理
- 评审如何闭环
- 哪些 artefact 需要被显式写出来

这比“写更强的 system prompt”更接近真正的工程问题。

### 2. 它说明“可验证”与“不可验证”任务，其实可以共用一套 evaluator 思维

前端设计是主观问题，full-stack app QA 是相对客观问题。作者把二者放在一条方法链上，其实说明了一件事：

- 不同任务未必要共享同一套标准
- 但它们可以共享同一类 harness pattern：`生成 -> 评审 -> 反馈 -> 再生成`

区别只在于：

- 设计任务的评审标准来自品味与设计原则
- 应用开发任务的评审标准来自功能、产品深度与 QA 行为

换句话说，evaluator-optimizer pattern 在 agent engineering 里并不是“只适合文字润色”的模式，它对长时软件开发同样成立。

### 3. 它与 Managed Agents 那篇文章正好互补

如果结合 [Scaling Managed Agents: Decoupling the brain from the hands](./Scaling%20Managed%20Agents:%20Decoupling%20the%20brain%20from%20the%20hands.md) 一起看，可以得到一个更完整的 Anthropic 视角：

- 本文关心“任务级 harness 应该怎么组织”
- Managed Agents 那篇关心“运行时边界应该怎么设计”

前者更像 workflow design（工作流设计），后者更像 runtime architecture（运行时架构）。两者叠在一起，才更接近生产级 agent platform。

## 对实践者的启发

如果你在做自己的 coding agent、research agent 或长任务 workflow，本文最值得借鉴的不是某个具体 prompt，而是这些设计原则：

1. 把“模型做事”和“模型评审”分开。  
   尤其是当任务既复杂又长时，独立 evaluator 往往比自评可靠得多。

2. 对主观任务先写 rubric（评分规约），再做迭代。  
   不要直接问“好不好”，而要把“什么叫好”拆成可陈述标准。

3. planner 要偏高层，不要太早写死实现细节。  
   高层目标和交付标准适合前置；低层实现路径让 builder 在执行中再决定，更稳。

4. 让 QA 基于真实交互，而不是只看代码或只跑单元测试。  
   Playwright 这类 browser automation 工具之所以有价值，是因为很多问题只在真实使用路径里暴露。

5. 把 harness 当成“模型能力边界的动态补偿层”。  
   每个新模型上线，都值得重新做一次删减和重估；但如果你也因此把任务难度、时长或交付要求抬高了，就要准备为新的能力缺口继续补强 harness。

6. 把 artifact 设计成一等公民。  
   不论是 sprint contract、handoff note，还是 feature spec，它们都应该是结构化、可读、可传递的，而不是依赖 agent 自己记住一切。

## FAQ

### 1. `context reset` 和 `compaction` 到底差在哪里？

`compaction` 是保留同一个 agent 的连续会话，只把旧内容压缩摘要；`context reset` 是直接开新会话，让新 agent 通过交接材料重新上手。前者更轻，后者更“干净”。当模型有明显 `context anxiety` 时，reset 往往更有效；当模型长上下文稳定性提升后，compaction 的性价比会更高。

### 2. 为什么模型自己评自己通常不靠谱？

因为 generator 在完成任务后，常会带着“已经做了很多工作”的路径依赖去看结果，容易给自己过宽松的解释。独立 evaluator 虽然也是 LLM，但它的 prompt、目标和行为边界都可以单独调优，更容易被训练成怀疑式审查者。

### 3. 为什么 planner 不应该一开始就规定所有技术细节？

因为高层规格一旦把低层实现写死，而且写错了，后面 agent 往往会机械执行这些错误前提，导致偏差层层放大。高层设计适合前置，低层策略更适合让 generator 在执行阶段基于环境反馈自行决定。

### 4. evaluator 是不是以后都会成为标准配置？

不一定。本文给出的更准确判断是：当任务超出当前模型的稳定单打能力时，evaluator 很值；当任务已经处在模型能稳定完成的范围内时，evaluator 可能只是增加时间和 token 成本。

### 5. 小团队如果只能抄一部分，最该先抄什么？

我会优先抄三件事：

1. 用 planner 把短 prompt 扩成更完整的 spec。
2. 用真实工具做 QA，而不是只让模型口头自检。
3. 把 handoff artifact / contract / progress file 做成结构化文件。

这三件事通常比“再写一个更长的 system prompt”更能稳定提升结果。

## 建议的延伸阅读

建议按下面顺序阅读，会更容易看清 Anthropic 这条 agent engineering 线索：

1. `Building effective agents`  
   先建立 Anthropic 对 agent complexity 的总体方法论：从简单方案起步，只在证明确实有收益时增加复杂度。  
   https://www.anthropic.com/engineering/building-effective-agents

2. `Effective harnesses for long-running agents`  
   看 Anthropic 更早一版 long-running harness，理解 initializer、progress file、feature list、incremental progress 是怎么出现的。  
   https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents

3. `Effective context engineering for AI agents`  
   补齐这篇文章里反复出现的 context window、compaction、长上下文退化问题。  
   https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

4. `Scaling Managed Agents: Decoupling the brain from the hands`  
   回到平台层，看 Anthropic 怎样把 `session / harness / sandbox` 解耦成更通用的 runtime。  
   ./Scaling%20Managed%20Agents:%20Decoupling%20the%20brain%20from%20the%20hands.md

如果希望理解这篇文章里 `generator / evaluator` 灵感的来源，也可以补：

1. `Generative adversarial network`  
   https://en.wikipedia.org/wiki/Generative_adversarial_network

2. `Ralph Wiggum as a "software engineer"`  
   https://ghuntley.com/ralph/

## 参考资料

### Anthropic 原文与官方资料

1. Harness design for long-running application development  
   https://www.anthropic.com/engineering/harness-design-long-running-apps
2. Effective harnesses for long-running agents  
   https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
3. Effective context engineering for AI agents  
   https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
4. Building effective agents  
   https://www.anthropic.com/engineering/building-effective-agents
5. Agent SDK overview  
   https://code.claude.com/docs/en/agent-sdk/overview
6. Introducing Claude Opus 4.6  
   https://www.anthropic.com/news/claude-opus-4-6
7. Frontend design skill  
   https://github.com/anthropics/claude-code/blob/main/plugins/frontend-design/skills/frontend-design/SKILL.md

### 文中引用的外部资料

1. Generative adversarial network  
   https://en.wikipedia.org/wiki/Generative_adversarial_network
2. Ralph Wiggum as a "software engineer"  
   https://ghuntley.com/ralph/

## 可关联到本知识库的关键词

- harness design
- long-running application development
- planner generator evaluator
- evaluator-optimizer
- context reset
- context compaction
- frontend design rubric
- Playwright MCP
- long-running agent QA
