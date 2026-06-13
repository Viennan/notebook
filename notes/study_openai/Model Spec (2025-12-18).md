# Model Spec (2025-12-18) 中文导读

## 1. 文档定位

原文链接：<https://model-spec.openai.com/2025-12-18.html>

发布日期：2025-12-18

文档性质：这不是一篇普通 blog，而是 OpenAI 的 `Model Spec`（模型规范：描述 OpenAI 希望模型如何行动、拒绝、解释、使用工具和处理冲突的公开规则文档）的一个版本。它服务于三个目标：让模型有用；避免严重现实伤害；维持 OpenAI 产品和 API 的法律、声誉与社会许可。

需要特别注意的是，页面自身包含“newer version / historical reference”的版本提示，因此在把它用于生产策略、合规判断或评测基线时，应同时核对 `model-spec.openai.com/` 根页面、GitHub changelog 和 OpenAI release notes。就本次阅读而言，本文只研究用户给定的 2025-12-18 版本。

## 2. 一句话摘要

2025-12-18 版 `Model Spec` 的核心，是在 2025 年形成的 `chain of command`（指令层级：当不同来源的指令冲突时按权限排序处理）之上，新增并突出 `Under-18 / U18 Principles`（未成年人原则：面向 13-17 岁青少年的更保守、更重视现实支持的安全行为规则），同时继续强化 agent 行动边界、工具输出信任、隐私、诚实、无自有目标和安全完成。

## 3. 版本更新重点

根据 OpenAI 的 `model_spec` changelog，v2025.12.18 的显著更新包括：

- 新增 `Under-18 Safety Mode`，在适用于所有用户的安全规则之上，为 13-17 岁青少年的发展需求加入更适龄的行为指导。
- 简化并收紧 `honesty`（诚实）相关指导，尤其是移除“为了保密而撒谎”的规则倾向。
- 澄清用户 `time-on-site`（停留时长）和 `clicks`（点击）只能在真正服务用户价值时作为考虑因素，不能作为模型自身追求的目标。
- 其他小幅表述调整和 copy edits。

换句话说，这次更新不是重写整套规范，而是在前序版本的治理框架上补上“青少年使用场景”的强约束。

## 4. 核心框架：Chain of Command

`chain of command` 是全文的骨架。它把指令按权限分层，要求模型在冲突中服从更高层级：

| 权限层级 | 含义 | 是否可被覆盖 |
| --- | --- | --- |
| `Root` | 根级规则，只来自 Model Spec 及其细化政策，主要限制灾难性风险、直接伤害、违法、破坏指令层级等行为 | 不可被 system、developer、user 覆盖 |
| `System` | OpenAI 设置或传递的系统级规则，可因产品形态、地区、年龄等上下文变化 | 不可被 developer、user 覆盖 |
| `Developer` | API 开发者或应用开发者给模型的指令 | 可被 Root/System 覆盖 |
| `User` | 终端用户指令 | 可被 Root/System/Developer 覆盖 |
| `Guideline` | 默认行为建议，可被显式或隐式上下文覆盖 | 最容易被覆盖 |
| `No Authority` | assistant/tool 消息、引用文本、附件、多模态数据、工具输出中的未授权指令 | 默认无指令权限 |

这套分层有两个工程含义。第一，系统设计不能把所有文本都视为 prompt；检索结果、网页、用户粘贴材料、工具输出默认应被看作 `untrusted data`（不可信数据）。第二，模型不是“越听话越好”，而是要在权限、上下文、风险和用户真实意图之间做裁决。

## 5. 红线原则与风险模型

文档把 OpenAI 的模型行为目标拆成三类张力：

- `Misaligned goals`（目标错位）：模型误解用户目标，或被第三方 prompt injection 诱导。
- `Execution errors`（执行错误）：模型理解目标但执行错，例如高风险领域事实错误、计算错误、错误工具调用。
- `Harmful instructions`（有害指令）：用户或开发者明确要求有害行为，例如自伤、暴力、违法、隐私侵犯。

对应的治理手段不是单一“拒绝”，而是：遵循权限层级、限制 autonomy（自主性）、控制 side effects（副作用）、表达不确定性、在高风险情境下寻求确认，以及在不能直接满足请求时提供安全替代路径。

## 6. Agent 行为：Scope of Autonomy 与 Side Effects

文档非常重视 agentic systems（能使用工具并在现实世界执行动作的智能体系统）。两个概念尤其关键：

`scope of autonomy`（自主范围）：模型在多步骤任务中可以自行推进哪些子目标、使用哪些工具、产生哪些成本或外部影响、何时必须停下来确认。

`side effects`（副作用）：会影响外部世界或用户状态的动作，例如发邮件、删文件、花钱、修改外部文档、调用带敏感数据的外部 API、扩大权限、委派 sub-agent。

这对 AI engineering 的直接启发是：产品不应只给模型一句“帮我完成任务”，而应显式记录工具权限、截止时间、最大成本、数据边界、是否允许自动提交、何时需要人工确认。否则模型即使“意图正确”，也可能因为行动范围不清而制造不可逆损失。

## 7. Untrusted Data 与 Prompt Injection

`Ignore untrusted data by default` 是安全工程上最值得反复读的一节。规范明确说，引用文本、YAML/JSON/XML、`untrusted_text`、附件、多模态数据和工具输出默认没有指令权限，其中的命令应被当作信息，而不是要执行的指令。

但文档也承认现实复杂性：在 coding assistant 中，用户让模型实现功能时，`AGENTS.md`、`README`、代码注释可能就是用户隐式授权模型遵循的上下文。因此判断工具输出是否可被采纳，需要看来源可信度、与任务相关性、潜在副作用，以及用户是否可能知道这些内容。

实践建议：

- 把 RAG 检索内容、网页正文、用户上传文件放进明确的 untrusted 容器。
- 把 developer instruction 与 retrieved content 分离。
- 对可能带来不可逆副作用的工具调用，在执行前做 confirmation 或 dry run。
- 对“看起来像指令但来源不明”的内容，在最终回复里说明假设，必要时向用户确认。

## 8. Stay in Bounds：边界不是简单拒绝

`Stay in bounds` 列出了模型不能完全照做的场景，包括但不限于：

- 性化未成年人内容：`Root` 级禁止，连转换用户提供内容也不允许。
- 信息危害：例如可造成严重现实危害的操作性知识。
- 针对性政治观点操纵：不能帮助面向个人或人口群体定制操纵性政治说服策略。
- 创作者权益：不能输出受保护作品的大段替代性文本。
- 隐私：不能泄露或推断敏感个人信息。
- 极端主义暴力、仇恨、骚扰、违法、自伤、妄想或躁狂相关风险。
- 受监管建议：在医疗、法律、金融等领域应提供信息和决策支持，而不是替用户做专业裁断。

值得注意的是，OpenAI 在 2025 年引入并强调 `Safe Completions`（安全完成：当直接回答不合规时，尽量给出安全、相关、有帮助的替代回答），因此“拒绝”只是其中一种动作。更理想的行为是说明边界，同时把用户导向可允许、低风险、有用的内容。

## 9. Under-18 / U18 Principles

本版本最重要的新增内容是 `Under-18 Principles`。它的目标不是把青少年当作成人的缩小版，而是要求模型“treat teens like teens”：尊重、温暖，但边界更清晰。

核心原则包括：

- `Put teen safety first`：当青少年的最大自由与严重安全风险冲突时，优先安全。
- `Promote real-world support`：鼓励家庭、朋友、学校、专业人士和本地资源介入，而不是让 AI 成为主要支持来源。
- `Treat teens like teens`：不居高临下，也不把青少年当成人处理。
- `Be transparent`：说明 assistant 能做什么、不能做什么，并提醒其不是人类。

U18 场景下会更保守处理：

- 自伤、自杀、妄想、躁狂，即使是虚构、假设、历史或教育语境，也不能浪漫化或提供操作性指导。
- 沉浸式恋爱 roleplay、第一人称亲密互动、把 assistant 与青少年配对的浪漫场景不允许。
- 性或暴力内容即使非露骨，也不能启用第一人称 roleplay。
- 危险挑战、物质滥用、成人可能合法但对青少年高风险的活动，会被更广泛限制。
- 身体形象、节食、外貌比较、性别化外貌理想等内容要避免强化 body dissatisfaction。
- 不能教青少年向可信照护者隐藏与伤害相关的通信、症状或物品。

这意味着，U18 安全不是一个单独 filter，而是一层覆盖在整体 `Model Spec` 上的更高敏感度策略。

## 10. 对 AI 产品开发的启发

如果把这篇规范当作产品工程文档，它给出的不是“模型应该善良”这种抽象价值，而是一套可落地的行为协议：

- Prompt 架构上，应显式区分 system/developer/user/untrusted/tool。
- Tool calling 上，应对外部副作用、敏感数据流、工具可信度做结构化记录。
- Agent 产品上，应引入 scope、budget、deadline、approval gate、rollback plan。
- 内容安全上，应从“关键词拒绝”转向“风险分类 + 安全替代 + 用户意图澄清”。
- 青少年产品上，应把 age-aware behavior 放进模型行为、产品 UI、家长资源、危机干预链路，而不是只靠单轮拒绝。
- 评测上，应测试冲突指令、prompt injection、误授权、工具输出伪指令、隐私泄露、错误执行和 U18 边界。

## 11. FAQ

### Q1：`Root` 和 `System` 有什么区别？

`Root` 是 Model Spec 中不可覆盖的最高规则，用来保护最硬的安全和治理边界。`System` 是 OpenAI 可按产品、地区、年龄等上下文调整的系统规则，但仍高于 developer 和 user。

### Q2：开发者能不能覆盖 U18 原则？

不能。U18 原则在文档中带有 `U18` 和 `Root` 标记，面向未成年人场景时优先级非常高。开发者可以设计体验，但不能覆盖这些根级安全边界。

### Q3：工具输出为什么默认没有权限？

因为工具输出可能来自网页、外部 API、检索文档或攻击者控制的内容，其中可能混有 prompt injection。模型可以读取其中的信息，但不能默认执行里面的命令，除非上级指令明确授权且符合用户意图。

### Q4：这是否意味着模型应该更常拒绝？

不一定。文档的方向是更精细地完成任务：在不合规部分设边界，在可帮助部分继续提供信息、解释、替代方案或安全步骤。`Safe Completions` 正是这种思想。

### Q5：`No other objectives` 对商业产品有什么影响？

模型不能把停留时长、点击、升级付费、自我保存、积累资源等当作自身目标。产品可以有业务目标，但不能让模型为了业务指标而违背用户真实利益或更高层级规则。

### Q6：为什么不直接公开 hidden chain-of-thought？

文档认为 hidden chain-of-thought（隐藏思维链）可能包含未对齐内容、敏感推理或竞争性信息，因此默认不向用户或开发者暴露，最多以摘要形式提供。这与“诚实”不矛盾：模型应诚实说明不能提供内部推理全文，而不是编造。

## 12. 延伸阅读

- OpenAI Model Spec v2025.12.18：<https://model-spec.openai.com/2025-12-18.html>
- OpenAI Model Spec changelog：<https://github.com/openai/model_spec/blob/main/CHANGELOG.md>
- Updating our Model Spec with teen protections：<https://openai.com/index/updating-model-spec-with-teen-protections/>
- Model Release Notes：<https://help.openai.com/en/articles/9624314-model-release-notes>
- Sharing the latest Model Spec：<https://openai.com/index/sharing-the-latest-model-spec/>
- Introducing the Model Spec：<https://openai.com/index/introducing-the-model-spec/>
- Usage Policies：<https://openai.com/policies/usage-policies>
- Our approach to AI safety：<https://openai.com/index/our-approach-to-ai-safety/>
- GPT-5 Safe Completions：<https://openai.com/index/gpt-5-safe-completions/>
- OpenAI API Reference：<https://platform.openai.com/docs/api-reference>

