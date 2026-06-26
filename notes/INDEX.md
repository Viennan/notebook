# Notes Index

本索引用于快速定位各 topic。总索引只链接 topic 入口，具体文件请进入对应 topic 的 `INDEX.md` 按需加载。

## Topics

- [code-ana_codex](./code-ana_codex/)：`openai/codex` 代码仓库分析。当前已完成 agent core 主链路报告，覆盖交互式 CLI/TUI 到 app-server、core SQ/EQ、`submission_loop`、`run_turn`、Responses stream、工具执行、exec policy、approval/sandbox 与事件回传；源码以本 topic 内 submodule 为准。
- [code-ana_graphify](./code-ana_graphify/)：`graphify` 代码仓库分析。覆盖 agent-native knowledge graph 工具链定位、AST/语义抽取、NetworkX 构图、聚类分析、导出查询、多代码 agent 平台集成与源码子模块研究入口。
- [code-ana_hermes-agent](./code-ana_hermes-agent/)：`hermes-agent` 代码仓库分析初始化入口。源码已作为 submodule 引入，当前仅记录仓库信息、初始化 commit 与后续报告规划，尚未展开源码分析。
- [code-ana_opencode](./code-ana_opencode/)：`opencode` 代码仓库分析。当前聚焦 AI 编码代理主干：session continuation loop、LLM streaming、工具调用与权限、plan/coding 切换、subagent child session，以及 V2 durable runner 迁移路径。
- [study_anthropic-blogs](./study_anthropic-blogs/)：Anthropic 工程与产品博客学习笔记。当前聚焦 Managed Agents 架构、`brain / hands / session` 解耦、长任务 harness、planner/generator/evaluator workflow、context reset 与 structured artifact。
- [study_openai](./study_openai/)：OpenAI 官方资料与规范学习笔记。当前聚焦 `Model Spec`、chain of command、指令层级、agent autonomy、side effects、prompt injection、防现实伤害、隐私与 U18 Principles。

## Naming

- `study_*`：专题学习、博客/规范/资料导读。
- `code-ana_*`：代码仓库分析与源码研究。
