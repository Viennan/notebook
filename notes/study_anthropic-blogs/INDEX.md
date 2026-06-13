# Anthropic Blogs 知识库索引

本目录用于沉淀对 Anthropic 工程与产品博客的研究导读。当前索引按照“主题摘要 + 文档链接”的方式维护，便于后续持续扩展。

## 已收录文章

### Managed Agents 与 Agent 架构

1. [Scaling Managed Agents: Decoupling the brain from the hands](./Scaling%20Managed%20Agents:%20Decoupling%20the%20brain%20from%20the%20hands.md)

   这篇导读聚焦 Anthropic 如何把 `brain / hands / session` 解耦，构建可恢复、可扩展、可隔离的 Managed Agents 运行架构。重点包括：

   - 为什么 harness 会随着模型能力提升而过时
   - 为什么 `session` 不能等同于 `context window`
   - 为什么它可以被理解为 `event-sourced agent workflow runtime`
   - 为什么它天然支持异步、可中断、分布式的 agent 执行模型
   - 为什么安全边界应通过架构隔离而非权限假设来建立
   - 为什么解耦能同时改善可靠性、企业接入能力与 `TTFT`

### Long-Running Harness 与 Agent Workflow Design

1. [Harness design for long-running application development](./Harness%20design%20for%20long-running%20application%20development.md)

   这篇导读聚焦 Anthropic 如何围绕长时间应用开发任务设计 `harness`，把主观设计评审、任务拆解、QA 验证与上下文治理组合成可持续迭代的 agent workflow。重点包括：

   - 为什么 naive harness 会在长任务里遭遇 context drift 与 self-evaluation 失真
   - 为什么 `generator / evaluator` 结构能显著改善前端设计和应用开发质量
   - 为什么 `planner / generator / evaluator` 比单一 coding agent 更适合 full-stack application development
   - 为什么 `context reset`、`compaction` 与 structured artifact 是长任务稳定性的关键
   - 为什么 harness 复杂度应随着模型能力与任务难度共同变化持续做 ablation，而不是简单做加法或减法
   - 为什么这篇文章与 Managed Agents 文章分别代表任务级 workflow 设计与平台级 runtime 设计两条互补路线
