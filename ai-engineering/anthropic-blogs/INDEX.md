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
