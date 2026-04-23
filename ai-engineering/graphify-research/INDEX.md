# graphify 研究知识库索引

## 文档列表

### [`graphify-report.md`](graphify-report.md)

一份关于 `graphify` 的主研究报告，重点覆盖：

- 项目定位与它试图解决的问题
- 检测、抽取、构图、聚类、分析、导出、查询的完整架构
- `detect`、`extract`、NetworkX graph 三种数据形态的区别，以及对应流程图说明
- AST 抽取与语义抽取之间的 ID 对齐问题，以及对应流程图说明
- 多模态语料支持、音视频转录中的 domain-aware prompt，以及 agent 集成机制
- FAQ 已整理前序高频问答，包括 Karpathy 的 `/raw` idea、NetworkX 作用、domain-aware prompt、Codex subagents 并行派发、service-like 运行形态、submodule 研究用途
- 核心技术选型，包括 Tree-sitter、NetworkX、Leiden、faster-whisper、MCP
- 工程质量、优势、局限、适用场景与后续研究方向
- 官方资料、源码核实信息与外部权威参考资料

适合在第一次接触 `graphify` 时通读，也适合作为后续专题文档的总入口。

### [`graphify-extraction.md`](graphify-extraction.md)

`graphify` 抽取层专题研究，重点覆盖：

- `extract.py` 的整体设计与输出 schema
- `LanguageConfig` 驱动的通用多语言 AST 抽取框架
- 节点、结构边、调用边、rationale 节点的生成逻辑
- `raw_calls` 与跨文件 call resolution 的两阶段处理
- 各语言支持的差异化能力与局限
- 测试如何约束 extraction 的稳定性与正确性

适合在已经了解主报告后，继续深入 `graphify` 的结构骨架是如何建立的。

### [`graphify-agent-integration.md`](graphify-agent-integration.md)

`graphify` 多代码 agent 平台适配专题研究，重点覆盖：

- 全局 skill 安装、项目级 always-on 规则、工具调用前 hook/plugin 的分层模型
- Claude Code、Codex、OpenCode、Cursor、Gemini CLI、Kiro、Copilot、Aider、Trae 等平台适配方式
- `CLAUDE.md`、`AGENTS.md`、`GEMINI.md`、Cursor rules、Kiro steering、Antigravity rules/workflows 的作用差异
- PreToolUse、BeforeTool、tool.execute.before、alwaysApply、steering 等机制对比
- Git hooks 与 agent hooks 的区别
- FAQ 已补充 Codex parallel subagent dispatch、semantic cache、chunking、service-like runtime 与 `wait_agent` / chunk-file 关系
- 平台适配的工程取舍与维护风险

适合理解 `graphify` 为什么不仅是 CLI 工具，而是一个嵌入 agent 行为循环的 knowledge graph 工具链。

## 当前结论摘要

`graphify` 不是单纯的代码可视化工具，而是一个面向 AI coding assistant 的 agent-native knowledge graph 工具链。它通过 AST（Abstract Syntax Tree，抽象语法树）抽取建立结构锚点，再结合多模态语料和代理工作流，把图摘要、图查询和平台 hook 一起接入开发助手。

从工程上看，它的强项在于：

- 分层清晰，模块化程度较高
- 多平台集成能力强
- 真实仓库场景下的细节处理较多
- 安全、防噪音、增量更新与导出能力比较完整

它的主要局限在于：

- 完整语义能力较依赖外部代理平台
- benchmark 与 insight 产出带有明显启发式成分
- 底层采用 NetworkX，更适合中小规模知识图谱

## 建议阅读顺序

1. 先读 [`graphify-report.md`](graphify-report.md)，建立全局认识。
2. 再读 [`graphify-extraction.md`](graphify-extraction.md)，理解 AST 抽取与结构骨架如何形成。
3. 接着读 [`graphify-agent-integration.md`](graphify-agent-integration.md)，理解它如何嵌入不同代码 agent 平台。
4. 如果后续继续扩展知识库，可优先补充：
   `graphify-benchmarking.md`
   `graphify-security.md`
   `graphify-vs-graphrag.md`
