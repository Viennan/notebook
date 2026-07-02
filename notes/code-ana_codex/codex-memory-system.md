# Codex Memory System 代码研究报告

本文聚焦三个问题：Codex 是否存在可随时间增长、逐步完善的记忆系统；它是否利用“记忆”在后续 session 中优化协作；以及它如何管理这些记忆。分析以本 topic 内 `openai/codex` submodule 的当前源码 commit `6d2168f06ae275d5e1f73cabf935d2bcc8549998` 为准，官方 Codex manual 用于校准产品语境。

## 核心结论

1. **Codex 确实有一套可增长的本地 memory 系统**。它不是模型权重里的长期记忆，而是 `$CODEX_HOME/memories/` 下的文件化记忆层，加上 `state` 里的 SQLite 记忆表和后台提取/整合管线。
2. **它会把 memory 用到后续 session 里**。启用后，Codex 会把已有 memory 注入后续会话的 prompt，并在后续 thread 中继续累积新的 rollout、summary 和 consolidated memory。
3. **它对记忆的治理很强**。有显式 feature flag、线程级 use/generate 控制、external-context 污染标记、速率限制门槛、secret redaction、usage 记账、retention/prune 和 reset 路径。
4. **它把 memory 视为辅助 recall，而不是硬规则来源**。官方手册明确提示，团队必须规则应放在 `AGENTS.md` 或 checked-in 文档里，memory 只是本地可回读的上下文层。

一句话概括：Codex 的 memory 是一种“**能慢慢长出来、也能被约束和清理的本地协作记忆层**”。

## 系统轮廓

```text
session prompt read path
  -> 读已有 memories / citations
  -> 注入后续 session 的上下文

startup write pipeline
  -> 从历史 rollout 抽取 stage-1 memory
  -> 合并到 raw_memories.md / rollout_summaries/
  -> 用 consolidation agent 产出更高层产物

governance path
  -> use_memories / generate_memories / disable_on_external_context
  -> rate-limit guard / secret redaction / retention / reset / pruning
```

## 源码地图

| 主题 | 关键文件 | 作用 |
| --- | --- | --- |
| 官方语境 | [Codex manual memory section](https://developers.openai.com/codex/memories) | 产品层对 memories 的定义、启用方式、线程级控制和安全提醒。 |
| 记忆总览 | [`codex-rs/memories/README.md`](./codex/codex-rs/memories/README.md) | 说明 read/write 分层、启动管线、两阶段整合、workspace 目录。 |
| 启动入口 | [`codex-rs/memories/write/src/start.rs`](./codex/codex-rs/memories/write/src/start.rs) | 线程启动后异步触发 memory pipeline。 |
| Phase 1 | [`codex-rs/memories/write/src/phase1.rs`](./codex/codex-rs/memories/write/src/phase1.rs) | 从历史 rollouts 抽取结构化 memory。 |
| Phase 2 | [`codex-rs/memories/write/src/phase2.rs`](./codex/codex-rs/memories/write/src/phase2.rs) | 合并 raw memories，生成 workspace diff，并启动 consolidation agent。 |
| Read path prompt | [`codex-rs/ext/memories/templates/memories/read_path.md`](./codex/codex-rs/ext/memories/templates/memories/read_path.md)、[`codex-rs/ext/memories/src/extension.rs`](./codex/codex-rs/ext/memories/src/extension.rs) | 把 memory 使用规则和 `memory_summary.md` 注入 developer prompt，并暴露可选 memory tools。 |
| Citation parsing | [`codex-rs/memories/read/src/citations.rs`](./codex/codex-rs/memories/read/src/citations.rs)、[`codex-rs/utils/stream-parser/src/citation.rs`](./codex/codex-rs/utils/stream-parser/src/citation.rs) | 解析隐藏 `<oai-mem-citation>`，抽取展示 citation 和 rollout ids。 |
| State DB | [`codex-rs/state/src/runtime.rs`](./codex/codex-rs/state/src/runtime.rs)、[`codex-rs/state/src/runtime/memories.rs`](./codex/codex-rs/state/src/runtime/memories.rs) | 打开 `memories.sqlite`，管理 stage1_outputs/jobs。 |
| 记忆表结构 | [`codex-rs/state/memory_migrations/0001_memories.sql`](./codex/codex-rs/state/memory_migrations/0001_memories.sql) | `stage1_outputs` / `jobs` schema。 |
| 线程记忆模式 | [`codex-rs/core/src/session/session.rs`](./codex/codex-rs/core/src/session/session.rs) | 新建/恢复 thread 时写入 `memory_mode`。 |
| 外部上下文污染 | [`codex-rs/core/src/stream_events_utils.rs`](./codex/codex-rs/core/src/stream_events_utils.rs) | 记录 memory citation，必要时把 thread 标成 polluted。 |
| 记忆使用记账 | [`codex-rs/core/src/memory_usage.rs`](./codex/codex-rs/core/src/memory_usage.rs) | 对读取 memory 文件的 shell/read 行为做 telemetry 分类。 |
| TUI 控制面 | [`codex-rs/tui/src/bottom_pane/memories_settings_view.rs`](./codex/codex-rs/tui/src/bottom_pane/memories_settings_view.rs) | `use_memories` / `generate_memories` / reset 的用户入口。 |

## 1. Codex 有“能长大”的记忆系统吗

有，而且是明确的文件化与数据库化设计。

官方手册把 memories 描述成能从 earlier threads 带 useful context 到 future work 的 local recall layer；源码侧则把它拆成两层：

- **持久文件层**：`$CODEX_HOME/memories/`，包含 `MEMORY.md`、`memory_summary.md`、`raw_memories.md`、`rollout_summaries/`、`skills/` 等产物。
- **状态数据库层**：`memories.sqlite` 里的 `stage1_outputs` 和 `jobs`，保存抽取结果、usage 统计、选择标记和后台任务状态。

因此，memory 会随时间增长：

- 新 thread / rollout 被抽取为 stage-1 memory。
- stage-1 memory 被合并成 raw memory workspace。
- consolidation agent 再把 workspace 整理成更高层产物。
- 旧 memory 还会记录 `usage_count` 和 `last_usage`，供后续筛选和保留。

这不是“无限堆积”，而是“增长 + 选择 + 整理 + 过期”的循环。

## 2. 它会不会用 memory 优化后续 session

会，而且路径很直接。

官方手册说，启用 memories 后，Codex 可以记住稳定偏好、重复工作流、技术栈、项目惯例和已知坑点。源码里对应两件事：

1. **会话启动时的读路径**：read crate 负责 memory developer-instruction injection 和 citation parsing，说明 memories 会进入后续 prompt 组织。
2. **线程启动时的写入标记**：`Session::new` 在创建 thread metadata 时会把 `memory_mode` 写入 `enabled` 或 `disabled`，后续是否产生/使用 memory 都受这个状态影响。

后续 session 的优化体现在：

- 读取已有 memory 作为上下文，而不是靠用户重复说明。
- 线程级 `use_memories` 控制是否把已有 memory 注入后续会话。
- 线程级 `generate_memories` 控制当前 thread 是否能成为未来 memory 的来源。
- 被 citation 引用的 stage-1 output 会回写 `usage_count` 和 `last_usage`，让系统知道哪些记忆真的被用过。

所以这里的“优化协作”不是抽象愿景，而是：

```text
旧 thread / rollouts
  -> 抽取 / 合并 / 整理
  -> 下次 session 读取
  -> 缩短重复说明，恢复项目惯例
```

## 3. 它如何管理记忆

Codex 对 memory 的管理比“开关一个缓存”复杂，但边界清晰。

### 3.1 入口开关

- `config.toml` 里 `[features] memories = true` 才能启用 feature。
- TUI 里 `/memories` 可单线程设置 `use_memories` 和 `generate_memories`。
- `Session::new` 只会给启用生成记忆的 thread 写入 `memory_mode = Enabled`。

### 3.2 线程级控制

源码把记忆分成两个线程维度：

- `use_memories`：当前线程是否读取已有 memory。
- `generate_memories`：当前线程是否允许作为未来 memory 的来源。

这两个维度是解耦的，所以可以出现：

- 读旧记忆，但不产出新记忆。
- 产出新记忆，但当前线程不使用旧记忆。

### 3.3 何时不该写 memory

Codex 对记忆写入有明确抑制条件：

- 线程太短命或不是合格 rollout，不进入后台抽取。
- `disable_on_external_context = true` 时，如果线程用了 web search、tool search 或 MCP，会把 thread 标成 `polluted`，避免继续纳入 memory 生成。
- rate-limit remaining percentage 低于阈值时，memory startup 直接跳过。
- secrets 会在 memory 字段生成后做 redaction。

### 3.4 Phase 1 如何产出 DB-backed raw memory

Phase 1 是“单个历史 rollout -> 一条 DB 原始记忆”的抽取过程。启动时，`claim_stage1_jobs_for_startup` 会从历史 `threads` 中挑选候选 rollout：

- `threads.memory_mode = 'enabled'`。
- 不是当前 thread。
- 来源属于允许的 interactive session source。
- `updated_at` 在最大年龄窗口内，并且已经 idle 足够久。
- `stage1_outputs.source_updated_at` 或 `jobs.last_success_watermark` 还没有覆盖当前 `source_updated_at`。
- 没有正在运行、冷却中或重试耗尽的同类 job。

claim 成功后，Phase 1 读取 rollout `.jsonl`，把可用于记忆抽取的 `RolloutItem` 序列化给模型。过滤规则很重要：

- developer messages 被排除。
- 用户上下文中的 `AGENTS.md` 片段和 skill 片段被排除，避免把已有规则/skill 原文再次写进 memory。
- 环境上下文、subagent notification、inter-agent communication 等仍可保留。
- 输入 rollout 内容和模型输出都会经过 secret redaction。

Phase 1 prompt 使用严格 JSON schema，模型必须返回：

```json
{
  "rollout_summary": "compact routing/index summary",
  "rollout_slug": "optional-slug-or-null",
  "raw_memory": "markdown raw memory for this rollout"
}
```

如果 `raw_memory` 或 `rollout_summary` 为空，就走 no-output 路径：job 标记成功，但不会保留有效 `stage1_outputs`；如果之前已经有旧输出，旧行会被删除并触发后续 Phase 2 遗忘。成功输出则 upsert 到 `stage1_outputs`：

```sql
thread_id TEXT PRIMARY KEY,
source_updated_at INTEGER NOT NULL,
raw_memory TEXT NOT NULL,
rollout_summary TEXT NOT NULL,
rollout_slug TEXT,
generated_at INTEGER NOT NULL,
usage_count INTEGER,
last_usage INTEGER,
selected_for_phase2 INTEGER NOT NULL DEFAULT 0,
selected_for_phase2_source_updated_at INTEGER
```

`thread_id` 是这个 rollout/thread 的身份；`source_updated_at` 是这次抽取对应的源 thread 更新时间，也是后续判断“这一条 stage1 output 属于哪一版 rollout”的版本号。

### 3.5 Phase 2 如何产出本地 memory workspace

Phase 2 是全局 consolidation。它先 claim 单例 job：

```text
kind = memory_consolidate_global
job_key = global
```

然后从 `stage1_outputs` 中选一批作为本轮文件系统输入。选择逻辑不是全量读取，而是：

- 只看非空 `raw_memory` 或 `rollout_summary`。
- 对应 thread 仍然是 `memory_mode = 'enabled'`。
- 如果这条 memory 曾经被引用过，要求 `last_usage` 在 `max_unused_days` 窗口内。
- 如果从未被引用过，要求 `source_updated_at` 在窗口内。
- 候选排序：`usage_count DESC`，再 `COALESCE(last_usage, source_updated_at) DESC`，再 `source_updated_at DESC`，再 `thread_id DESC`。
- 选出 top-N 后，再按 `thread_id ASC` 返回，使生成文件稳定。

Phase 2 把这批 Stage 1 output 同步成两类输入文件：

```md
# Raw Memories

Merged stage-1 raw memories (stable ascending thread-id order):

## Thread `019...`
updated_at: 2026-...
cwd: /path/to/repo
rollout_path: /path/to/rollout.jsonl
rollout_summary_file: 2026-...-slug.md

<raw_memory markdown body>
```

以及每个 rollout 一个 summary 文件：

```md
thread_id: 019...
updated_at: 2026-...
rollout_path: /path/to/rollout.jsonl
cwd: /path/to/repo
git_branch: main

<rollout_summary>
```

`$CODEX_HOME/memories` 是一个由 Codex 管理的 git workspace。Phase 2 同步 DB 输入后，用 git diff 生成 `phase2_workspace_diff.md`。如果 workspace 没有变化，就直接成功；如果有变化，就启动一个受限 consolidation agent。这个 agent：

- cwd 是 memory root。
- `ephemeral = true`。
- `generate_memories = false` 且 `use_memories = false`，避免递归读写 memory。
- 无网络，approval never，只能写 memory root。
- 禁用 apps、plugins、collab、memory tool 等能力。

consolidation agent 的目标产物是：

- `MEMORY.md`：durable、可 grep 的任务族 handbook。
- `memory_summary.md`：总是注入 prompt 的高密度导航摘要，第一行必须是 `v1`。
- `skills/*`：可选的可复用 procedure。

`MEMORY.md` 的严格块结构大致如下：

```md
# Task Group: <cwd / project / workflow / task family>

scope: <what this block covers>
applies_to: cwd=<path-or-family>; reuse_rule=<reuse boundary>

## Task 1: <task description, outcome>

### rollout_summary_files

- rollout_summaries/file.md (cwd=..., rollout_path=..., updated_at=..., thread_id=...)

### keywords

- keyword1, keyword2, error-string, repo-concept

## User preferences

- when <situation>, user asked/corrected: "<near-verbatim>" -> future behavior guidance [Task 1]

## Reusable knowledge

- validated repo/system facts, commands, decision triggers, procedures [Task 1]

## Failures and how to do differently

- symptom -> cause -> fix/pivot [Task 1]
```

`memory_summary.md` 则更像 prompt-loaded route map：

```md
v1

## User Profile

<concise user/workflow profile>

## User preferences

- actionable preference likely to matter again

## General Tips

- broadly useful workflow, environment, verification, or tool guidance
```

这套设计的关键不是“自动总结一次就完”，而是“先抽取、再选择、再文件化、再让受限 agent 基于 workspace diff 增量整合”。它避免了把所有长期记忆都塞进普通 session prompt。

### 3.6 Snapshot 与 watermark

Stage 2 成功后会重写 `selected_for_phase2`，源码注释称它保留“exact selected stage-1 snapshots”。这里的 snapshot 不是另一个文件或表，而是某条 Stage 1 output 的具体版本：

```text
(thread_id, source_updated_at)
```

`stage1_outputs` 以 `thread_id` 为主键，同一个 thread 后续更新时会覆盖旧行：

```text
thread_id=A, source_updated_at=100  # 旧版 stage1 output
thread_id=A, source_updated_at=101  # 新版 stage1 output
```

Phase 2 标记本轮成功选择时使用：

```sql
WHERE thread_id = ? AND source_updated_at = ?
```

所以它只会标记“本轮实际同步进文件系统的那一版 Stage 1 output”。如果 consolidation 运行过程中同一 thread 又被新的 Phase 1 输出覆盖，旧的 `(thread_id, source_updated_at)` 不会误伤新版。

`watermark` 是 jobs 表里的输入进度账本，核心字段是：

```text
input_watermark
last_success_watermark
```

在 Phase 1 中，`input_watermark = source_updated_at`。claim job 时，如果 `stage1_outputs.source_updated_at >= source_updated_at`，或 `jobs.last_success_watermark >= source_updated_at`，说明这个 rollout 版本已经处理过，可以跳过。成功后 `last_success_watermark = input_watermark`，避免同一版本重复抽取；如果 `source_updated_at` 前进，新版本又可重新 claim，并可重置 retry。

在 Phase 2 中，watermark 是全局 consolidation 的调度账本。Stage 1 成功或删除旧输出时，会推进 global job 的 `input_watermark`；Phase 2 完成时写入：

```text
completed_watermark = max(claimed_watermark, selected_outputs.source_updated_at.max())
last_success_watermark = max(old_last_success_watermark, completed_watermark)
```

但 Phase 2 **不靠 DB watermark 判断是否真的有整合工作**。源码注释明确把 git workspace diff 作为真实 dirty check：同步 `raw_memories.md` / `rollout_summaries/` 后，如果 git diff 为空，就无需启动 consolidation agent。

### 3.7 记忆如何被引用、计数与回收

被引用的 stage-1 output 会更新 usage 统计：

```sql
UPDATE stage1_outputs
SET
    usage_count = COALESCE(usage_count, 0) + 1,
    last_usage = ?
WHERE thread_id = ?
```

触发点不是“读了 memory 文件”，而是“完成的 assistant 输出里出现了可解析的 memory citation”。完整路径是：

```text
assistant 最终输出
  -> strip_citations 抽出隐藏 <oai-mem-citation>...</oai-mem-citation>
  -> parse_memory_citation 解析 <citation_entries> 与 <rollout_ids>/<thread_ids>
  -> 合法 rollout/thread UUID 转成 ThreadId
  -> record_stage1_output_usage 对每个 ThreadId 更新 usage_count 和 last_usage
```

read-path prompt 要求模型在使用任何 relevant memory files 后，于最终回答末尾追加一个隐藏 citation：

```md
<oai-mem-citation>
<citation_entries>
MEMORY.md:234-236|note=[responsesapi citation extraction code pointer]
rollout_summaries/2026-02-17T21-23-02-LN3m-example.md:10-12|note=[weekly report format]
</citation_entries>
<rollout_ids>
019c6e27-e55b-73d1-87d8-4e01f1f75043
019c7714-3b77-74d1-9866-e1f484aae2ab
</rollout_ids>
</oai-mem-citation>
```

这意味着 usage 的直接决策权在 LLM 输出侧：LLM 需要按 prompt 主动声明自己使用了哪些 rollout ids。系统只负责剥离隐藏块、解析 id、更新 DB；它不会根据文件访问记录自动反推“这次用了哪个 rollout”。

边界如下：

- 只有 `<citation_entries>` 不会让 `usage_count + 1`；它主要服务 UI/rendering provenance。
- `<rollout_ids>` 或 legacy `<thread_ids>` 中能转成合法 `ThreadId` 的 id 才会进入 usage 记账。
- parser 会对同一 citation 中重复的 rollout id 去重，通常单次最终回复同一 rollout 只加一次。
- DB 中缺失的 id 被忽略。
- 如果 citation 有 entries 但没有合法 rollout id，turn 仍会被标成“有 memory citation”，但不会增加任何 stage1 row 的 usage。

这说明 Codex 不是只“积累”，而是会根据真实使用情况做选择性保留。

### 3.8 清理与复位

Codex 还提供了显式 reset 路径：

- TUI 里可以清空本地 memory 文件和 rollout summaries。
- `clear_memory_roots_contents` 会清理 `memories/` 和 `memories_extensions/`。
- Phase 2 会 prune 过期 extension resources 和旧 summary 文件。

## 4. 记忆在 prompt 里到底是什么

这里要区分两种东西：

- **真正的 memory 文件/DB**：长期持久化、可增长、可清理。
- **模型当前看到的 memory 片段**：作为 prompt 注入的上下文。

Codex 不是把 memory 当成“全局对话历史”，而是把它变成可复用的上下文材料。普通 read path 中，直接注入当前 session prompt 的不是完整 `$CODEX_HOME/memories`，而是：

1. memory read-path developer instructions。
2. `memory_summary.md` 的截断内容。

注入文本的大致组织方式是：

```md
## Memory

You have access to a memory folder with guidance from prior runs...

Decision boundary:
- clearly self-contained trivial requests can skip memory
- use memory by default when the query mentions workspace/repo/module/path/files
- use memory when prior decisions, project conventions, or ambiguity may matter

Memory layout:
- <codex_home>/memories/memory_summary.md (already provided below; do NOT open again)
- <codex_home>/memories/MEMORY.md (searchable registry; primary file to query)
- <codex_home>/memories/skills/<skill-name>/
- <codex_home>/memories/rollout_summaries/

Quick memory pass:
1. skim MEMORY_SUMMARY below and extract task-relevant keywords
2. search MEMORY.md
3. open 1-2 relevant rollout summaries or skills only if MEMORY.md points there
4. search rollout_path only when exact evidence is needed

Memory citation requirements:
<oai-mem-citation>...</oai-mem-citation>

========= MEMORY_SUMMARY BEGINS =========
<contents of memory_summary.md>
========= MEMORY_SUMMARY ENDS =========
```

因此，模型看到的是“导航层 + 检索规则”，不是一次性塞入所有细节。它会：

- 把 memory 注入后续 session 的读取路径。
- 先用 `memory_summary.md` 判断是否值得查。
- 需要时搜索 `MEMORY.md`。
- 只有 `MEMORY.md` 指向更具体材料时，再打开少量 `rollout_summaries/` 或 `skills/`。
- 把 citation 解析成 `MemoryCitation`，并据此统计 usage。
- 把 memory 文件保持在本地工作区，不假装它是系统规则本体。

这也解释了官方手册的提醒：`AGENTS.md` 和 checked-in 文档负责硬规则，memories 只负责 helpful recall。

## 5. 风险与边界

我认为 Codex 这套 memory 设计更像“可审计的个人记忆库”，而不是“自我演化的长期人格”。

它的优点是：

- 能跨 session 复用稳定背景。
- 能用后台任务逐步完善。
- 能用 usage、retention、pollution 和 reset 做治理。

它的限制也很明显：

- 记忆是本地生成物，不能替代仓库里的正式规则。
- 记忆质量依赖抽取与 consolidation 模型。
- 外部上下文、短线程和低 quota 都会让它保守退出。

## 结论

如果只回答三个问题：

1. **是否存在可随时间增长、完善的记忆系统？**
   有。它由文件化 memory + state DB + 两阶段后台 pipeline 构成。
2. **是否利用“记忆”在后续 session 中优化协作？**
   有。后续 session 会读取已有 memories，把稳定偏好、项目约定和常见坑带回 prompt。
3. **如何管理记忆？**
   通过 feature flag、线程级 use/generate 开关、external-context 污染标记、rate-limit gate、secret redaction、usage 记账、retention/prune、reset 和 consolidation workspace diff。

可以把它理解成一种“能慢慢长出来、但随时能被约束和清理”的本地协作记忆层。
