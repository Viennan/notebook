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
| Read path | [`codex-rs/memories/read/src/lib.rs`](./codex/codex-rs/memories/read/src/lib.rs) | 提供 memory root、citation parsing、usage 分类。 |
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

### 3.4 记忆如何被整理

Codex 采用两阶段：

1. **Phase 1**：从 eligible rollout 抽取 `raw_memory` 和 `rollout_summary`，写入 DB。
2. **Phase 2**：把选择出的 stage-1 输出同步到工作区，生成 `raw_memories.md` 和 `rollout_summaries/`，再交给 consolidation agent 整合成更高层产物。

这套设计的关键不是“自动总结”，而是“先抽取，再整合，再让 agent 审阅 workspace diff”。它避免了把所有长期记忆都塞进单次模型输出。

### 3.5 记忆如何被消耗与回收

被引用的 stage-1 output 会更新 usage 统计：

- `usage_count + 1`
- `last_usage = now`

然后 Phase 2 选取时会参考：

- `max_unused_days`
- `max_raw_memories_for_consolidation`
- 最新的 `last_usage/generated_at`

这说明 Codex 不是只“积累”，而是会根据真实使用情况做选择性保留。

### 3.6 清理与复位

Codex 还提供了显式 reset 路径：

- TUI 里可以清空本地 memory 文件和 rollout summaries。
- `clear_memory_roots_contents` 会清理 `memories/` 和 `memories_extensions/`。
- Phase 2 会 prune 过期 extension resources 和旧 summary 文件。

## 4. 记忆在 prompt 里到底是什么

这里要区分两种东西：

- **真正的 memory 文件/DB**：长期持久化、可增长、可清理。
- **模型当前看到的 memory 片段**：作为 prompt 注入的上下文。

Codex 不是把 memory 当成“全局对话历史”，而是把它变成可复用的上下文材料。它会：

- 把 memory 注入后续 session 的读取路径。
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
