# hermes-agent 研究知识库索引

本 topic 针对 [`NousResearch/hermes-agent`](https://github.com/NousResearch/hermes-agent) 做代码分析，源码作为 submodule 放在 [`./hermes-agent`](./hermes-agent)。当前已完成 topic 初始化、权威参考资料索引和 core report。

## 仓库信息

- 上游仓库：<https://github.com/NousResearch/hermes-agent>
- submodule 路径：[`./hermes-agent`](./hermes-agent)
- 初始化分支：`main`
- 初始化 commit：`bb7ff7dc302cbcbe41cf6bc09424ffc9fb2d062f`
- 递归子模块：无
- 依赖初始化：未执行

## 文档列表

### [`hermes-agent-core-report.md`](./hermes-agent-core-report.md)

主研究报告，重点覆盖：

- CLI、messaging gateway、ACP、cron 等入口如何收敛到 `AIAgent.run_conversation`
- `AIAgent` 如何作为 facade，把初始化、turn prologue、provider loop、工具执行、prompt、session 和 memory/skills 拆到 `agent/` 内部模块
- system prompt 的 `stable / context / volatile` 分层，以及为什么 Hermes 尽量保持 prompt cache prefix 稳定
- provider runtime 如何解析 `chat_completions`、`codex_responses`、`anthropic_messages` 等 API mode
- 工具系统如何通过 registry、toolset、availability、agent-level interception、middleware 和并发执行组成行动能力
- SQLite session store、FTS5 search、memory、skills、delegation、context compression 如何支撑跨 turn 持续性

适合先读，用来建立 Hermes Agent 的主干执行链路和源码地图。

## 参考资料

- [`ref/INDEX.md`](./ref/INDEX.md)：官方文档、上游仓库和源码内权威资料入口。

## 后续分析入口

后续开始分析时，可优先补充：

1. `hermes-agent-tool-runtime.md`：工具 registry、toolsets、approval、MCP、terminal backend 与 managed gateway 专题。
2. `hermes-agent-prompt-memory-skills.md`：system prompt、memory、skills、自我改进闭环专题。
3. `hermes-agent-gateway-session.md`：gateway session key、agent cache、platform callbacks、reset/resume 专题。
