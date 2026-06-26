# hermes-agent skill system

## 核心结论

`hermes-agent` 的 skill system 是一套 procedural memory 闭环：它不只让 agent 读取“技能说明”，还允许 agent 在完成任务后主动把可复用流程沉淀为 skill，并通过后台 review 和 curator 持续修补、收敛、归档。

```text
skills package
  -> compact index in system prompt
  -> model loads relevant skill on demand
  -> task execution produces reusable workflow
  -> skill_manage create / patch / write_file
  -> usage + provenance telemetry
  -> background review extracts missed lessons
  -> curator prunes / archives / consolidates
```

它的精华不是某个单独工具，而是把“经验”建模成可审计目录包，再把读、写、复盘和长期维护都接进 agent runtime。

当前 submodule commit：`bb7ff7dc302cbcbe41cf6bc09424ffc9fb2d062f`。本文以该源码为准，官方文档用于补充产品语境。

## 为什么 skill 是亮点

LLM agent 常见的问题是：一次任务中学到的经验很快随上下文窗口消失；如果把所有经验都写进长期 memory，又会把“用户事实”和“操作流程”混在一起，后续很容易变成过度指令或错误泛化。

Hermes 的建模是：

- memory 记录 declarative facts：用户是谁、偏好是什么、环境有哪些稳定事实。
- skill 记录 procedural knowledge：这类任务应该怎么做、容易踩什么坑、需要哪些模板/脚本/参考资料。

所以 skill system 解决的是“agent 如何积累做事方法”的问题。它让经验保持文件化、可查、可修、可归档，而不是隐式藏在模型权重或历史对话中。

## 全局模型

skill 是目录包，不是单个 prompt 片段。

| 实体 | 角色 | 存储位置 | 何时进入上下文 | 维护者 |
| --- | --- | --- | --- | --- |
| `SKILL.md` | 主说明，含 frontmatter、适用场景、流程和注意事项 | `~/.hermes/skills/<skill>/SKILL.md` 或 external skill dir | 索引阶段只放 name/description；`skill_view` 后全文进入 tool result | 用户、agent、background review |
| `references/` | 权威资料、会话细节、错误复现、知识库摘录 | skill 目录内 | `skill_view(name, file_path)` 按需加载 | 用户或 `skill_manage(write_file)` |
| `templates/` | 可复制改造的起始文件 | skill 目录内 | 按需加载 | 同上 |
| `scripts/` | 可重复运行的验证、生成、探测脚本 | skill 目录内 | 按需加载并可被 agent 调用 | 同上 |
| `assets/` | 图像、二进制辅助材料 | skill 目录内 | 按需加载 | 同上 |
| `.usage.json` | 使用、查看、修补、归档状态 | `~/.hermes/skills/.usage.json` | 不进入 prompt，供 curator 和治理逻辑使用 | `tools/skill_usage.py` |
| `.archive/` | 归档 skill package | `~/.hermes/skills/.archive/` | 不进入 prompt | curator |

目录包模型带来一个重要后果：`SKILL.md` 应该负责“何时使用、怎么开始、到哪里读更多”，而不是塞满所有细节。细节进入 `references/`、`templates/`、`scripts/`，通过 on-demand loading 延迟展开。

## Prompt 位置

Hermes 的 skill prompt 是渐进披露，而不是一次性注入全部 skill。

```text
system prompt
  stable guidance:
    SKILLS_GUIDANCE
    compact skills index

tool loop
  skills_list()
    -> metadata only
  skill_view(name)
    -> full SKILL.md
  skill_view(name, file_path)
    -> support file

slash / preload
  /skill-name or --skills
    -> full skill payload as user-message style context
```

关键路径：

- `agent/system_prompt.py` 在 `skill_manage` 可用时加入 `SKILLS_GUIDANCE`。
- 同文件在 `skills_list` / `skill_view` / `skill_manage` 任一可用时调用 `build_skills_system_prompt()`。
- `agent/prompt_builder.py` 的 `build_skills_system_prompt()` 扫描本地和 external skill dir，生成紧凑索引，并带两层 cache：进程内 LRU 与磁盘 snapshot。
- `tools/skills_tool.py` 的 `skills_list()` 只返回 name、description、category 等轻量 metadata。
- `tools/skills_tool.py` 的 `skill_view()` 才读取完整 `SKILL.md` 或支持文件。
- `agent/skill_commands.py` 处理 `/skill-name`，`agent/skill_bundles.py` 参与 bundle 发现和预加载。

这个设计服务于两个目标：

- 控制 token：多数 turn 只需要“知道有哪些技能”，不需要所有技能全文。
- 稳定 prompt cache：system prompt 尽量保持 byte-stable，skill 全文通过 tool result 或用户消息样式 payload 进入当前 turn。

## Skill 加载与使用路径

一次正常 skill 使用可以分成四步：

```text
turn start
  -> system prompt 已含 compact skills index

model planning
  -> 根据 task 和 description 判断相关 skill
  -> 必要时调用 skills_list(category?)

progressive load
  -> skill_view(name) 读取 SKILL.md
  -> SKILL.md 指向 support file 时再 skill_view(name, file_path)

execution
  -> 按 skill 指导调用 terminal / browser / code / other tools
  -> skill_view 会 bump view/use telemetry
```

这解释了 skill 和普通文档的差异：普通文档只被人阅读；Hermes skill 是 agent runtime 的可见对象，模型会被系统提示要求主动扫描、加载、使用和修补。

## 自动创建与迭代

Hermes 有两条 skill 自动演化路径：前台即时写回和后台复盘写回。

前台路径：

```text
用户任务
  -> 模型完成复杂任务 / 修复棘手错误 / 发现非平凡 workflow
  -> SKILLS_GUIDANCE 提醒保存 approach
  -> skill_manage(create|patch|write_file|edit)
  -> clear_skills_system_prompt_cache()
  -> usage/provenance telemetry
```

后台路径：

```text
conversation_loop
  -> 每个 tool-calling iteration 增加 _iters_since_skill
  -> skill_manage 被使用时重置计数

turn_finalizer / codex_runtime
  -> 如果 _iters_since_skill >= _skill_nudge_interval
  -> 且 skill_manage 可用
  -> spawn_background_review_thread(review_skills=True)

background review agent
  -> 复盘完整 turn
  -> 使用 memory/skills toolset
  -> patch existing skill 或 create class-level umbrella
```

默认 nudge interval 来自 `skills.creation_nudge_interval`，初始化路径在 `agent/agent_init.py`。这个计数是按 tool iteration 而不是按自然 turn 累加，说明 Hermes 更关心“模型实际做了多少可学习的工作”。

## skill_manage 写入模型

`tools/skill_manager_tool.py` 是 skill 的统一写入口。它的 action 面如下：

| Action | 用途 | 典型场景 |
| --- | --- | --- |
| `create` | 创建新 skill package 和 `SKILL.md` | 没有现有 umbrella 覆盖某类任务 |
| `patch` | 精确替换 `SKILL.md` 或支持文件片段 | 增加步骤、修正 pitfall、补一行 references 指针 |
| `edit` | 整体重写 `SKILL.md` | 结构大改，不适合局部 patch |
| `write_file` | 写入 `references/`、`templates/`、`scripts/`、`assets/` | 沉淀资料、模板、脚本 |
| `remove_file` | 删除支持文件 | 清理过时支撑材料 |
| `delete` | 删除 skill package | curator 合并或人工清理 |

治理层也在这里：

- `create` 校验 name、category、frontmatter、大小，并检查全目录命名冲突。
- `patch` 复用 fuzzy matching，降低模型因为空格/缩进不完全一致而 patch 失败的概率。
- `write_file` 只允许写入受控子目录，且有 1 MiB 文件大小上限。
- 所有写入后做 skill 安全扫描，失败则回滚。
- 成功后清理 skills system prompt cache，让后续 prompt 索引能反映变化。
- `write_approval` 开启时，skill 写入先进入 pending store，等待用户确认。
- `pinned` skill 不能 delete，但仍允许 patch/edit/write_file。

因此 `skill_manage` 不是普通文件写工具，而是“受限 package mutation API”。

## Background review 的抽取原则

`agent/background_review.py` 的 `_SKILL_REVIEW_PROMPT` 和 `_COMBINED_REVIEW_PROMPT` 是自动创建/迭代的核心算法说明。它们要求 review agent 主动寻找以下信号：

- 用户纠正了风格、格式、verbosity、工作方式或步骤顺序。
- 本轮出现了可复用的技巧、修复、workaround、debug path 或 tool usage pattern。
- 已加载或咨询过的 skill 被证明过时、不完整或错误。

更新优先级不是“马上新建 skill”，而是 umbrella-first：

1. 更新当前加载过的 skill。
2. 查找并更新已有 class-level umbrella skill。
3. 在已有 umbrella 下增加 `references/`、`templates/` 或 `scripts/`。
4. 只有没有任何合适归宿时，才创建新的 class-level umbrella skill。

prompt 还明确禁止沉淀几类内容：

- 安装缺失、凭证未配、临时路径迁移等环境依赖失败。
- “某工具不可用”这类负面长期结论。
- 已经通过 retry 解决的瞬时错误本身。
- 一次性任务叙事。

这套规则和 memory system 的边界一致：用户偏好可以同时进入 memory 和 skill，但如果偏好是“这类任务下应该怎样做”，必须写到 skill 的 procedure 中。

## Background review 的隔离

review fork 不是在主 agent 里悄悄继续对话，而是新建一个受限 `AIAgent`：

```text
parent agent
  -> snapshot messages
  -> create review_agent(skip_memory=True)
  -> inherit parent runtime / session_id / cached system prompt when safe
  -> whitelist memory + skills toolset
  -> disable memory/skill nudges
  -> run review prompt
  -> collect successful memory/skill actions for notification
```

几个边界很重要：

- `skip_memory=True` 避免外部 memory provider 被 review prompt 污染。
- review agent 重新绑定父 agent 的 built-in memory store，所以 built-in memory 写入仍能落盘。
- runtime whitelist 只允许 memory/skill 工具，其他工具即使出现在 schema 中也会被拒绝。
- review fork 禁用递归 nudge 和 compression，避免后台任务影响主会话。
- 同模型路径会复用父 agent cached system prompt，以保持 provider cache parity。

换句话说，后台 review 负责“学习”，但不应该拥有正常任务 agent 的全部行动能力。

## Provenance 与 telemetry

Hermes 对 agent-created skill 做了来源区分：

```text
foreground skill_manage(create)
  -> 视为用户当前任务指令下的创建
  -> 不自动标记为 agent-created

background_review skill_manage(create)
  -> tools/skill_provenance.py 标记 origin
  -> tools/skill_usage.py mark_agent_created(name)
  -> 纳入 curator 管理
```

`tools/skill_usage.py` 的 `.usage.json` 记录：

- `use_count`
- `view_count`
- `patch_count`
- `last_used_at`
- `last_viewed_at`
- `last_patched_at`
- `created_by`
- `state`
- `pinned`
- `archived_at`

这让 curator 能区分“用户安装/维护的技能”和“agent 自己产生的技能”，也能判断哪些 skill 长期无人使用、需要 stale 或 archive。

## Curator：长期维护与反熵

如果只有自动创建，skill 库会很快膨胀成一堆窄而重复的条目。`agent/curator.py` 解决的是长期反熵问题。

curator 的运行不是简单 cron，而是 opportunistic maintenance：

- CLI 启动时会检查是否可运行。
- gateway housekeeping loop 会周期性触发检查。
- 内层还会看 interval、idle time、首次运行状态和配置。

默认取向：

- enabled 默认为 true。
- interval 常见为 7 天。
- min idle 常见为 2 小时。
- stale after 常见为 30 天。
- archive after 常见为 90 天。
- `consolidate` 默认不强制开启。

curator 有两层行为：

```text
deterministic phase
  -> active but unused long enough => stale
  -> stale/old enough => archive to .archive/
  -> pinned skip transitions

optional LLM phase
  -> inspect agent-created skills
  -> consolidate narrow skills into umbrella skills
  -> preserve references/templates/scripts/assets package integrity
  -> generate report and backup
```

它不会随便删除：删除路径本质上是 archive / package move，且有 backup 和 report。hub-installed skills、bundled skills、受保护 skill、pinned skill 都有额外保护。

## 维护 prompt 与合并语义

skill 维护相关 prompt 可以按运行层次分成几类：

| Prompt / schema | 路径 | 何时出现 | 维护意图 |
| --- | --- | --- | --- |
| `SKILLS_GUIDANCE` | [`agent/prompt_builder.py`](./hermes-agent/agent/prompt_builder.py) | `skill_manage` 可用时进入 system prompt | 前台提示模型：复杂任务、棘手错误、非平凡 workflow 后保存 skill；使用中发现 skill 过时、不全或错误时立即 patch。 |
| compact skills index | [`agent/prompt_builder.py`](./hermes-agent/agent/prompt_builder.py) | `skills_list` / `skill_view` / `skill_manage` 任一可用时进入 system prompt | 让模型先看到轻量 skill landscape，用 name/description 选择是否加载全文。 |
| `_SKILL_REVIEW_PROMPT` | [`agent/background_review.py`](./hermes-agent/agent/background_review.py) | turn 后台 review 只复盘 skills 时 | 主动寻找可沉淀 procedure：优先更新已加载 skill，其次已有 umbrella，再补 support file，最后才新建 class-level umbrella。 |
| `_COMBINED_REVIEW_PROMPT` | [`agent/background_review.py`](./hermes-agent/agent/background_review.py) | turn 后台 review 同时复盘 memory 和 skills 时 | 把“用户是谁”写入 memory，把“这类任务怎么做”写入 skill；用户对做法的纠正应进入相关 skill。 |
| `CURATOR_REVIEW_PROMPT` | [`agent/curator.py`](./hermes-agent/agent/curator.py) | `curator.consolidate=true` 或手动 `--consolidate` 时 | 做 umbrella-building consolidation：扫描 agent-created skill cluster，按内容收敛为 class-level skill，不做被动 duplicate audit。 |
| `CURATOR_DRY_RUN_BANNER` | [`agent/curator.py`](./hermes-agent/agent/curator.py) | curator dry-run | 禁止 mutation，只输出 would-do 报告，供人工决定是否 live run。 |
| `SKILL_MANAGE_SCHEMA` | [`tools/skill_manager_tool.py`](./hermes-agent/tools/skill_manager_tool.py) | `skill_manage` 工具 schema | 定义 create/patch/edit/delete/write_file/remove_file，并要求 delete 时用 `absorbed_into` 声明 consolidation vs pruning。 |

这里的 `umbrella skill` 不是单纯路由表。开启 consolidation 后，相似 skill 通常不会“各自内容完整不变地挂到同一个 umbrella 下”。curator prompt 鼓励三种处理：

1. 选一个已有 broad skill 作为 umbrella，patch 进其他 sibling 的独特经验。
2. 没有合适 umbrella 时，create 一个新的 class-level `SKILL.md`，把 shared workflow 和关键差异重构成短小分节。
3. 对窄但有价值的 session-specific 内容，移动到 umbrella 的 `references/`、`templates/` 或 `scripts/`。

旧 sibling 随后会被 archive，并在 `skill_manage(action="delete")` 时用 `absorbed_into=<umbrella>` 声明内容去向。也就是说，umbrella 的本质是“合并后的主承载 skill”：它兼具发现入口、流程说明和支撑资料索引，而不是运行时把请求转发给旧 skill。

确实存在“完整重构出新 skill”的情况。`CURATOR_REVIEW_PROMPT` 明确要求：如果 cluster 中没有任何现有成员足够 broad，就创建新的 class-level umbrella skill，新的 `SKILL.md` 直接覆盖共享 workflow，并用 labeled subsections 吸收旧 skill 的 unique insight。这是 LLM 按 prompt 重写出的独立 skill，不是一个简单引用文件。

## 维护算法与质量控制

Hermes 的 skill 维护不是只有 prompt，但语义判断主要依赖 LLM。额外算法更多是治理、审计和状态机：

| 机制 | 是否语义算法 | 作用 |
| --- | --- | --- |
| `apply_automatic_transitions()` | 否 | 根据使用时间把 active 标 stale，把 stale / old skill archive。 |
| `agent_created_report()` / `.usage.json` | 否 | 提供 use/view/patch/state/pinned 等候选信息。 |
| curator candidate list | 弱语义，主要是结构化输入 | 把 agent-created skill 摘成 LLM 可读清单。 |
| `CURATOR_REVIEW_PROMPT` | 是，LLM 判断 | 判断 cluster、umbrella class、merge/keep/prune。 |
| `absorbed_into` | 否 | delete 时显式声明旧 skill 是合并到 umbrella 还是无目标 pruning。 |
| `_classify_removed_skills()` / `_reconcile_classification()` | 否，启发式审计 | 对 LLM final YAML、tool calls 和 `absorbed_into` 做一致性分类，生成 rename/report。 |
| `skill_manage` validation | 否 | 校验路径、frontmatter、大小、写入目录、pinned、security scan、approval gate。 |

因此当前源码里没有看到 embedding 相似度、聚类打分、自动 reward model 或自动质量评分器。所谓“相似技能合并”，主要是 LLM 在 curator prompt 下阅读候选列表、加载 skill、调用 `skill_manage` 和 terminal 完成；程序负责限制边界、记录来源、归档恢复、报告差异和迁移引用。

防止 skill 劣化主要靠规则约束和可恢复治理：

- 粒度标准：偏好 class-level skill，不要 one-session-one-skill。
- 内容标准：保留 trigger conditions、明确步骤、pitfalls、verification、support file 指针。
- 负样本标准：不保存临时环境失败、未配置凭证、一次性任务叙事、对工具的长期负面判断。
- 合并标准：问“人类维护者会写 N 个 skill，还是一个 skill 加 N 个分节？”而不是看名字是否完全相同。
- 完整性标准：合并前把 source skill 当完整 package；有 support files 或相对链接时，必须 re-home 文件并改路径，或保留 standalone / 整包 archive。
- 安全标准：不碰 hub-installed、protected built-in、pinned skill；delete 实际是可恢复 archive；写入前后有校验和扫描。
- 审计标准：每次 curator run 写 report，记录 consolidated / pruned，rename summary 可用于发现错误。

这里确实有“激励信号”，但它是 prompt 层的软激励，不是算法 reward。比如 background review 明说“不更新不是中性结果”，curator 明说“少于 10 个 archive 可能停太早”，并持续强调 umbrella-building。这些语句会推动 LLM 更积极维护 skill，但最终质量仍取决于模型判断、人工审计和后续使用中的 patch。

## Skill 生命周期

可以把一个 agent-created skill 的生命周期看成状态机：

```text
created / active
  -> viewed or used
  -> patched when lesson changes
  -> unused beyond stale threshold
  -> stale
  -> still unused beyond archive threshold
  -> archived

pinned
  -> skip stale/archive/consolidation delete
  -> still patchable
```

这个生命周期体现了一个很现实的判断：经验本身会过期。Hermes 没有把 agent 总结的东西视为永久真理，而是把它放进一个会被使用统计、空闲时间和 curator 反复审查的维护循环。

## 与 memory system 的边界

memory 和 skill 最容易混淆，因为两者都属于“长期学习”。区分方式是：

| 问题 | 应进 memory | 应进 skill |
| --- | --- | --- |
| 这是谁？ | 用户身份、偏好、稳定事实 | 不适合 |
| 这是什么事实？ | 环境、项目、约定的 declarative fact | 只有当事实支撑某类流程时，放 references |
| 下次怎么做？ | 不适合写成命令式自我指令 | 步骤、流程、检查清单、pitfall |
| 用户纠正了我的做法 | 可记录偏好事实 | 必须修补对应任务 skill |
| 一次任务完成记录 | 通常不保存 | 通常不保存，除非抽象成 class-level workflow |

这也是我们在本 notebook 中借鉴 Hermes 时最有价值的点：`profile/EXP.md` 更像 skill，而不是 memory。它应该沉淀“以后如何维护这个仓库/这类 topic”，而不是记录流水账。

## 设计优点与风险

优点：

- 经验文件化，可审计、可 diff、可人工修补。
- system prompt 只放索引，全文按需加载，token 成本可控。
- foreground、background review、curator 形成闭环，既能新增也能清理。
- support files 让 skill 能携带脚本、模板和权威资料，不必把所有内容塞进 `SKILL.md`。
- provenance/usage/pinning/write approval 给自动写入加上治理边界。

风险：

- 抽取质量依赖 prompt 与模型判断，仍可能过度总结或漏掉经验。
- `write_approval` 默认是否开启取决于配置；关闭时 agent 写入更顺滑，但治理更依赖后验检查。
- foreground `create` 不标记 agent-created，curator 不会自动维护这类用户指令下创建的 skill。
- curator consolidation 默认不一定开启，重复 skill 的收敛可能滞后。
- external skill dirs 可被发现和部分修改路径影响，需要依赖路径校验和配置约束。

## 源码地图

- [`agent/prompt_builder.py`](./hermes-agent/agent/prompt_builder.py)：`SKILLS_GUIDANCE`、`build_skills_system_prompt()`、skill index cache。
- [`agent/system_prompt.py`](./hermes-agent/agent/system_prompt.py)：把 skill guidance 和 compact index 接入 stable system prompt。
- [`tools/skills_tool.py`](./hermes-agent/tools/skills_tool.py)：`skills_list()`、`skill_view()` 和 progressive disclosure。
- [`tools/skill_manager_tool.py`](./hermes-agent/tools/skill_manager_tool.py)：`skill_manage()` 的 create/patch/edit/delete/write_file/remove_file。
- [`agent/skill_commands.py`](./hermes-agent/agent/skill_commands.py) / [`agent/skill_bundles.py`](./hermes-agent/agent/skill_bundles.py)：slash skill、bundle 和预加载入口。
- [`agent/conversation_loop.py`](./hermes-agent/agent/conversation_loop.py)：tool iteration 计数和 `skill_manage` reset。
- [`agent/turn_finalizer.py`](./hermes-agent/agent/turn_finalizer.py)：普通 conversation path 的后台 skill review 触发。
- [`agent/codex_runtime.py`](./hermes-agent/agent/codex_runtime.py)：Codex runtime path 的 skill review 触发。
- [`agent/background_review.py`](./hermes-agent/agent/background_review.py)：skill review prompt、fork 隔离和工具白名单。
- [`tools/skill_provenance.py`](./hermes-agent/tools/skill_provenance.py)：background review 来源标记。
- [`tools/skill_usage.py`](./hermes-agent/tools/skill_usage.py)：usage telemetry、agent-created 标记和生命周期状态。
- [`agent/curator.py`](./hermes-agent/agent/curator.py)：长期维护、归档、consolidation、report。
- [`cli.py`](./hermes-agent/cli.py) / [`gateway/run.py`](./hermes-agent/gateway/run.py)：curator opportunistic trigger。

## 测试证据

相关测试覆盖了几个关键约束：

- `tests/tools/test_skill_manager_tool.py`：foreground create 不标记 agent-created，background review create 会标记。
- `tests/tools/test_write_approval.py`：skill 写入审批会 stage，而不是直接落盘。
- `tests/tools/test_skill_usage.py`：usage / view / patch telemetry。
- `tests/run_agent/test_background_review_toolset_restriction.py`：background review runtime whitelist 与 schema parity。
- `tests/agent/test_curator.py`：curator prompt invariants、归档、pinning 和 consolidation 约束。

## 参考资料

- [`ref/INDEX.md`](./ref/INDEX.md)
- [`hermes-agent/website/docs/user-guide/features/skills.md`](./hermes-agent/website/docs/user-guide/features/skills.md)
- [`hermes-agent/website/docs/developer-guide/creating-skills.md`](./hermes-agent/website/docs/developer-guide/creating-skills.md)
- [`hermes-agent/website/docs/developer-guide/prompt-assembly.md`](./hermes-agent/website/docs/developer-guide/prompt-assembly.md)
- [`hermes-agent/website/docs/user-guide/features/curator.md`](./hermes-agent/website/docs/user-guide/features/curator.md)
- [`hermes-agent/website/docs/reference/tools-reference.md`](./hermes-agent/website/docs/reference/tools-reference.md)
